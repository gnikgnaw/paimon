# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Apache Paimon is a lake format for building Realtime Lakehouse Architecture with Flink and Spark. It combines lake format with LSM (Log-Structured Merge-tree) structure for real-time streaming updates. Version: 1.5-SNAPSHOT.

## Syntax Requirements

- Based on JDK 8 and Scala 2.12. Higher version syntax features must not be used.

## Build & Test Commands

### Build

```shell
# Full build (skip tests)
mvn clean install -DskipTests

# Compile single module
mvn -pl <module> -DskipTests compile

# Compile with local dependency changes
mvn -pl <module> -am -DskipTests compile
```

### Test

```shell
# Single Java test method (fastest feedback)
mvn -pl <module> -Pfast-build -Dtest=TestClassName#methodName test

# Single Java test class
mvn -pl <module> -Pfast-build -Dtest=TestClassName test

# With local dependency changes
mvn -pl <module> -am -Pfast-build -DfailIfNoTests=false -Dtest=TestClassName#methodName test

# Scala tests (use scalatest-maven-plugin, NOT surefire)
mvn -pl paimon-spark/paimon-spark-ut -am -Pfast-build -DfailIfNoTests=false \
  -DwildcardSuites=org.apache.paimon.spark.sql.WriteMergeSchemaTest \
  -Dtest=none test
```

**`-Pfast-build`** skips checkstyle, spotless, scalastyle, enforcer, and RAT checks — use for local iteration only.

### Formatting

```shell
# Format all code (Java uses Google Java Format AOSP style; Scala uses scalafmt)
mvn spotless:apply
```

### IDE Setup

Mark `paimon-common/target/generated-sources/antlr4` as Sources Root.

## Maven Profiles

| Profile | Purpose | Activation |
|---------|---------|------------|
| `fast-build` | Skip all checks for local iteration | `-Pfast-build` |
| `spark3` | Spark 3.x modules (3.2–3.5, Scala 2.12) | Active by default |
| `spark4` | Spark 4.0 module (Scala 2.13, Java 17) | `-Pspark4` |
| `flink1` | Flink 1.x modules (1.16–1.20) | Active by default |
| `flink2` | Flink 2.x modules (2.0–2.2, Java 11) | `-Pflink2` |
| `scala-2.13` | Build with Scala 2.13 | `-Pscala-2.13` |
| `paimon-iceberg` | Iceberg compatibility module | Auto on JDK 11 |

## Code Style

- **Java**: Google Java Format (AOSP style) via Spotless. Import order: `org.apache.paimon` → `org.apache.paimon.shade` → others → `javax` → `java` → `scala` → static imports.
- **Scala**: scalafmt 3.10.2 with Scala 2.12 dialect, max 100 columns. Config in `.scalafmt.conf`.
- **Shaded dependencies**: Guava and Jackson are shaded. Use `org.apache.paimon.shade.guava30.com.google.common.*` and `org.apache.paimon.shade.jackson2.com.fasterxml.jackson.*` — never import the unshaded packages directly.
- **License header**: Required on all source files (from `copyright.txt`).
- **Tests**: JUnit 5 (Jupiter) with AssertJ assertions. Temporary directories via `@TempDir`.

## Architecture

### Module Dependency Layers

```
paimon-api          (public interfaces: Table, Snapshot, Catalog.Identifier, CoreOptions, types)
    ↑
paimon-common       (data types, InternalRow, FileIO, codegen, predicate, file index, compression)
    ↑
paimon-format       (file format: ORC, Parquet, Avro, CSV, JSON, text)
    ↑
paimon-core         (storage engine: FileStore, LSM merge-tree, manifest, schema, catalog impls)
    ↑
paimon-flink / paimon-spark / paimon-hive   (compute engine integrations)
```

### Core Storage Concepts (paimon-core)

- **FileStore**: Central storage abstraction. Two implementations:
  - `KeyValueFileStore` — primary-key tables using LSM merge-tree
  - `AppendOnlyFileStore` — append-only tables
- **Merge Tree** (`mergetree/`): LSM-tree with Levels, SortedRun, compaction, LookupLevels for point queries
- **Snapshot** (`Snapshot.java` in paimon-api): Entry point to committed data, referencing manifest lists (base + delta + changelog)
- **Manifest** (`manifest/`): ManifestFile tracks DataFileMeta entries; ManifestList indexes manifest files per snapshot
- **Schema** (`schema/`): SchemaManager handles schema evolution
- **Catalog** (`catalog/`): FileSystemCatalog (default), RESTCatalog, plus CachingCatalog decorator
- **Deletion Vectors** (`deletionvectors/`): Bitmap-based row-level deletion for merge-on-read
- **File Index** (`fileindex/` in paimon-common): Bloom filter, bitmap, BSI, range bitmap indices embedded in data files
- **Operations** (`operation/`): FileStoreCommit, FileStoreScan, SplitRead — the read/write execution layer

### Table Read/Write Path

- **Read**: `Table.newReadBuilder()` → `ReadBuilder` → `TableScan` (produces `Split[]`) → `TableRead` (reads splits)
- **Write**: `Table.newBatchWriteBuilder()`/`newStreamWriteBuilder()` → `TableWrite` (writes data) → `TableCommit` (atomic commit)
- **System Tables** (`table/system/`): Built-in metadata tables ($snapshots, $schemas, $manifests, $files, $options, $branches, $tags, etc.)

### Compute Engine Integrations

- **paimon-flink**: Source/Sink connectors, CDC sync (database/table from MySQL/Kafka/etc.), Flink SQL procedures, actions. Version-specific modules (1.16–2.2) extend `paimon-flink-common` / `paimon-flink1-common` / `paimon-flink2-common`.
- **paimon-spark**: Spark DataSource V2, SQL extensions, procedures. Version-specific modules (3.2–4.0) extend `paimon-spark-common`. Scala tests in `paimon-spark-ut`.
- **paimon-hive**: Hive catalog and SerDe connectors for Hive 2.1–3.1.

### Supporting Modules

- **paimon-filesystems**: Pluggable file system implementations (S3, OSS, HDFS, Azure, GCS, COS, OBS, Jindo)
- **paimon-service**: Client/runtime for lookup service (point query acceleration)
- **paimon-arrow**: Arrow-based vectorized read
- **paimon-codegen**: Runtime code generation (uses Janino)
- **paimon-bundle**: Shaded uber-jar packaging
- **paimon-iceberg**: Iceberg REST metadata compatibility layer (requires JDK 11)
- **paimon-lance**: Lance columnar format integration
- **paimon-vortex**: Vortex columnar format integration (JNI-based)
- **paimon-tantivy**: Full-text search via Tantivy engine (Rust/JNI)
- **paimon-lumina**: Vector index support

### Data Row Representation

The internal data representation lives in `paimon-common/data/`:
- `InternalRow` — row interface (implementations: `BinaryRow`, `GenericRow`, `JoinedRow`, `NestedRow`)
- `BinaryString`, `Decimal`, `Timestamp`, `BinaryArray`, `BinaryMap` — column value types
- Type system in `paimon-common/types/` and `paimon-api/types/`
