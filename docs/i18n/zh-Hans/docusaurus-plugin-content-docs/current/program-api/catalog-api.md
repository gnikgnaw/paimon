---
title: "Catalog API"
sidebar_position: 4
---

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Catalog API {#catalog-api}

## 创建数据库 {#create-database}

你可以使用 Catalog 来创建数据库。创建的数据库会持久化在文件系统中。

```java
import org.apache.paimon.catalog.Catalog;

public class CreateDatabase {

    public static void main(String[] args) {
        try {
            Catalog catalog = CreateCatalog.createFilesystemCatalog();
            catalog.createDatabase("my_db", false);
        } catch (Catalog.DatabaseAlreadyExistException e) {
            // do something
        }
    }
}
```

## 判断数据库是否存在 {#determine-whether-database-exists}

你可以使用 Catalog 来判断数据库是否存在

```java
import org.apache.paimon.catalog.Catalog;

public class DatabaseExists {

    public static void main(String[] args) {
        Catalog catalog = CreateCatalog.createFilesystemCatalog();
        boolean exists = catalog.databaseExists("my_db");
    }
}
```

## 列出数据库 {#list-databases}

你可以使用 Catalog 来列出数据库。

```java
import org.apache.paimon.catalog.Catalog;

import java.util.List;

public class ListDatabases {

    public static void main(String[] args) {
        Catalog catalog = CreateCatalog.createFilesystemCatalog();
        List<String> databases = catalog.listDatabases();
    }
}
```

## 删除数据库 {#drop-database}

你可以使用 Catalog 来删除数据库。

```java
import org.apache.paimon.catalog.Catalog;

public class DropDatabase {

    public static void main(String[] args) {
        try {
            Catalog catalog = CreateCatalog.createFilesystemCatalog();
            catalog.dropDatabase("my_db", false, true);
        } catch (Catalog.DatabaseNotEmptyException e) {
            // do something
        } catch (Catalog.DatabaseNotExistException e) {
            // do something
        }
    }
}
```

## 修改数据库 {#alter-database}

你可以使用 Catalog 来修改数据库的属性。（注意：仅支持 hive 和 jdbc Catalog）

```java
import java.util.ArrayList;
import org.apache.paimon.catalog.Catalog;

public class AlterDatabase {

    public static void main(String[] args) {
        try {
            Catalog catalog = CreateCatalog.createHiveCatalog();
            List<DatabaseChange> changes = new ArrayList<>();
            changes.add(DatabaseChange.setProperty("k1", "v1"));
            changes.add(DatabaseChange.removeProperty("k2"));
            catalog.alterDatabase("my_db", changes, true);
        } catch (Catalog.DatabaseNotExistException e) {
            // do something
        }
    }
}
```

## 判断表是否存在 {#determine-whether-table-exists}

你可以使用 Catalog 来判断表是否存在

```java
import org.apache.paimon.catalog.Catalog;
import org.apache.paimon.catalog.Identifier;

public class TableExists {

    public static void main(String[] args) {
        Identifier identifier = Identifier.create("my_db", "my_table");
        Catalog catalog = CreateCatalog.createFilesystemCatalog();
        boolean exists = catalog.tableExists(identifier);
    }
}
```

## 列出表 {#list-tables}

你可以使用 Catalog 来列出表。

```java
import org.apache.paimon.catalog.Catalog;

import java.util.List;

public class ListTables {

    public static void main(String[] args) {
        try {
            Catalog catalog = CreateCatalog.createFilesystemCatalog();
            List<String> tables = catalog.listTables("my_db");
        } catch (Catalog.DatabaseNotExistException e) {
            // do something
        }
    }
}
```

## 删除表 {#drop-table}

你可以使用 Catalog 来删除表。

```java
import org.apache.paimon.catalog.Catalog;
import org.apache.paimon.catalog.Identifier;

public class DropTable {

    public static void main(String[] args) {
        Identifier identifier = Identifier.create("my_db", "my_table");
        try {
            Catalog catalog = CreateCatalog.createFilesystemCatalog();
            catalog.dropTable(identifier, false);
        } catch (Catalog.TableNotExistException e) {
            // do something
        }
    }
}
```

## 重命名表 {#rename-table}

你可以使用 Catalog 来重命名表。

```java
import org.apache.paimon.catalog.Catalog;
import org.apache.paimon.catalog.Identifier;

public class RenameTable {

    public static void main(String[] args) {
        Identifier fromTableIdentifier = Identifier.create("my_db", "my_table");
        Identifier toTableIdentifier = Identifier.create("my_db", "test_table");
        try {
            Catalog catalog = CreateCatalog.createFilesystemCatalog();
            catalog.renameTable(fromTableIdentifier, toTableIdentifier, false);
        } catch (Catalog.TableAlreadyExistException e) {
            // do something
        } catch (Catalog.TableNotExistException e) {
            // do something
        }
    }
}
```

## 修改表 {#alter-table}

你可以使用 Catalog 来修改表，但需要注意以下几点。

- 列 %s 在 %s 表中不能指定 NOT NULL。
- 不能更新表中的分区列类型。
- 不能更改主键的可空性（nullability）。
- 如果列的类型是嵌套 row 类型，则不支持更新该列的类型。
- 不支持将列更新为嵌套 row 类型。

```java
import org.apache.paimon.catalog.Catalog;
import org.apache.paimon.catalog.Identifier;
import org.apache.paimon.schema.Schema;
import org.apache.paimon.schema.SchemaChange;
import org.apache.paimon.types.DataField;
import org.apache.paimon.types.DataTypes;

import com.google.common.collect.Lists;

import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class AlterTable {

    public static void main(String[] args) {
        Identifier identifier = Identifier.create("my_db", "my_table");

        Map<String, String> options = new HashMap<>();
        options.put("bucket", "4");

        Catalog catalog = CreateCatalog.createFilesystemCatalog();
        catalog.createDatabase("my_db", false);

        try {
            catalog.createTable(
                    identifier,
                    new Schema(
                            Lists.newArrayList(
                                    new DataField(0, "col1", DataTypes.STRING(), "field1"),
                                    new DataField(1, "col2", DataTypes.STRING(), "field2"),
                                    new DataField(2, "col3", DataTypes.STRING(), "field3"),
                                    new DataField(3, "col4", DataTypes.BIGINT(), "field4"),
                                    new DataField(
                                            4,
                                            "col5",
                                            DataTypes.ROW(
                                                    new DataField(
                                                            5, "f1", DataTypes.STRING(), "f1"),
                                                    new DataField(
                                                            6, "f2", DataTypes.STRING(), "f2"),
                                                    new DataField(
                                                            7, "f3", DataTypes.STRING(), "f3")),
                                            "field5"),
                                    new DataField(8, "col6", DataTypes.STRING(), "field6")),
                            Lists.newArrayList("col1"), // partition keys
                            Lists.newArrayList("col1", "col2"), // primary key
                            options,
                            "table comment"),
                    false);
        } catch (Catalog.TableAlreadyExistException e) {
            // do something
        } catch (Catalog.DatabaseNotExistException e) {
            // do something
        }

        // add option
        SchemaChange addOption = SchemaChange.setOption("snapshot.time-retained", "2h");
        // add column
        SchemaChange addColumn = SchemaChange.addColumn("col1_after", DataTypes.STRING());
        // add a column after col1
        SchemaChange.Move after = SchemaChange.Move.after("col1_after", "col1");
        SchemaChange addColumnAfterField =
                SchemaChange.addColumn("col7", DataTypes.STRING(), "", after);
        // rename column
        SchemaChange renameColumn = SchemaChange.renameColumn("col3", "col3_new_name");
        // drop column
        SchemaChange dropColumn = SchemaChange.dropColumn("col6");
        // update column comment
        SchemaChange updateColumnComment =
                SchemaChange.updateColumnComment(new String[]{"col4"}, "col4 field");
        // update nested column comment
        SchemaChange updateNestedColumnComment =
                SchemaChange.updateColumnComment(new String[]{"col5", "f1"}, "col5 f1 field");
        // update column type
        SchemaChange updateColumnType = SchemaChange.updateColumnType("col4", DataTypes.DOUBLE());
        // update column position, you need to pass in a parameter of type Move
        SchemaChange updateColumnPosition =
                SchemaChange.updateColumnPosition(SchemaChange.Move.first("col4"));
        // update column nullability
        SchemaChange updateColumnNullability =
                SchemaChange.updateColumnNullability(new String[]{"col4"}, false);
        // update nested column nullability
        SchemaChange updateNestedColumnNullability =
                SchemaChange.updateColumnNullability(new String[]{"col5", "f2"}, false);

        SchemaChange[] schemaChanges =
                new SchemaChange[]{
                        addOption,
                        removeOption,
                        addColumn,
                        addColumnAfterField,
                        renameColumn,
                        dropColumn,
                        updateColumnComment,
                        updateNestedColumnComment,
                        updateColumnType,
                        updateColumnPosition,
                        updateColumnNullability,
                        updateNestedColumnNullability
                };
        try {
            catalog.alterTable(identifier, Arrays.asList(schemaChanges), false);
        } catch (Catalog.TableNotExistException e) {
            // do something
        } catch (Catalog.ColumnAlreadyExistException e) {
            // do something
        } catch (Catalog.ColumnNotExistException e) {
            // do something
        }
    }
}
```
