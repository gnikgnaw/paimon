---
title: "贡献指南"
sidebar_position: 3
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

# 贡献指南 {#contributing}

Apache Paimon 由一个开放、友好的社区开发。我们诚挚欢迎每一个人加入社区并为 Apache Paimon 做出贡献。与社区互动并为 Paimon 做贡献的方式有很多，包括提问、提交缺陷报告、提出新特性、参与邮件列表上的讨论、贡献代码或文档、改进网站、测试发布候选版本以及撰写相应的博客等。

## 你想做什么？ {#what-do-you-want-to-do}
为 Apache Paimon 做贡献远不止为项目编写代码。下面我们列出了帮助项目的不同途径：

<table class="table table-bordered">
  <thead>
    <tr>
      <th>领域</th>
      <th>更多信息</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><span class="glyphicon glyphicon-exclamation-sign" aria-hidden="true"></span> 报告缺陷</td>
      <td>要报告 Paimon 的问题，请打开 <a href="https://github.com/apache/paimon/issues">Paimon 的 issues</a>。<br/>
      请详细说明你遇到的问题，并尽可能附上有助于复现该问题的描述。</td>
    </tr>
    <tr>
      <td><span class="glyphicon glyphicon-console" aria-hidden="true"></span> 贡献代码</td>
      <td>阅读<a href="#code-contribution-guide">代码贡献指南</a></td>
    </tr>
    <tr>
      <td><span class="glyphicon glyphicon-ok" aria-hidden="true"></span> 代码评审</td>
      <td>阅读<a href="#code-review-guide">代码评审指南</a></td>
    </tr>
    <tr>
      <td><span class="glyphicon glyphicon-thumbs-up" aria-hidden="true"></span> 发布版本</td>
      <td>发布一个新的 Paimon 版本。</td>
    </tr>
    <tr>
      <td><span class="glyphicon glyphicon-user" aria-hidden="true"></span> 支持用户</td>
      <td>在<a href="https://github.com/apache/paimon#mailing-lists">用户邮件列表</a>上回复问题，
          查看 <a href="https://github.com/apache/paimon/issues">Issues</a> 中的最新 issue，找出那些实际上是用户提问的工单。
      </td>
    </tr>
    <tr>
      <td><span class="glyphicon glyphicon-volume-up" aria-hidden="true"></span> 让更多人了解 Paimon</td>
      <td>组织或参加 Paimon Meetup，为 Paimon 博客投稿，在
          <a href="https://github.com/apache/paimon#mailing-lists">dev@paimon.apache.org 邮件列表</a>上分享你的会议、Meetup 或博客文章。
      </td>
    </tr>
    <tr>
      <td colspan="2">
        <span class="glyphicon glyphicon-question-sign" aria-hidden="true"></span> 还有其他问题？联系
                     <a href="https://github.com/apache/paimon#mailing-lists">dev@paimon.apache.org 邮件列表</a>以获取帮助！
      </td>
    </tr>
  </tbody>
</table>

## 代码贡献指南 {#code-contribution-guide}

Apache Paimon 由志愿者的代码贡献来维护、改进和扩展。我们欢迎对 Paimon 的贡献。

请随时提问。你可以发送邮件到 Dev 邮件列表，或者在你正在处理的 issue 上评论。

<style>
.contribute-grid {
  margin-bottom: 10px;
  display: flex;
  flex-direction: column;
  margin-left: -2px;
  margin-right: -2px;
}

.contribute-grid .column {
  margin-top: 4px;
  padding: 0 2px;
}

@media only screen and (min-width: 480px) {
  .contribute-grid {
    flex-direction: row;
    flex-wrap: wrap;
  }

  .contribute-grid .column {
    flex: 0 0 50%;
  }

  .contribute-grid .column {
    margin-top: 4px;
  }
}

@media only screen and (min-width: 960px) {
  .contribute-grid {
    flex-wrap: nowrap;
  }

  .contribute-grid .column {
    flex: 0 0 25%;
  }
}

.contribute-grid .panel {
  height: 100%;
  margin: 0;
}

.contribute-grid .panel-body {
  padding: 10px;
}

.contribute-grid h2 {
  margin: 0 0 10px 0;
  padding: 0;
  display: flex;
  align-items: flex-start;
}

.contribute-grid .number {
  margin-right: 0.25em;
  font-size: 1.5em;
  line-height: 0.9;
}
</style>

<div class="contribute-grid">
  <div class="column">
    <div class="panel panel-default">
      <div class="panel-body">
        <h2 id="consensus"><span class="number">1</span><a href="#consensus">讨论</a></h2>
        <p>创建一个 Issue 或邮件列表讨论，并达成共识</p>
        <p><b>申请处理某个 issue 时，请注意它不仅仅是一句“请分配给我”，你需要阐述你对该 issue 的理解、你的设计方案，并在可能的情况下提供你的 POC 代码。</b></p>
      </div>
    </div>
  </div>
  <div class="column">
    <div class="panel panel-default">
      <div class="panel-body">
        <h2 id="implement"><span class="number">2</span><a href="#implement">实现</a></h2>
        <p>按照 issue 中达成一致的方案创建 Pull Request。</p>
        <p><b>1. 只有在该 issue 已分配给你时才创建 PR。2. 请关联一个 issue（如果有），例如 fix #123。3. 请在你自己 fork 的克隆项目中启用 actions。</b></p>
      </div>
    </div>
  </div>
  <div class="column">
    <div class="panel panel-default">
      <div class="panel-body">
        <h2 id="review"><span class="number">3</span><a href="#review">评审</a></h2>
        <p>与评审者协作。</p><br />
        <p><b>1. 确保不包含任何无关或不必要的重新格式化改动。2. 请确保测试通过。3. 请不要 resolve conversation。</b></p>
      </div>
    </div>
  </div>
  <div class="column">
    <div class="panel panel-default">
      <div class="panel-body">
        <h2 id="merge"><span class="number">4</span><a href="#merge">合并</a></h2>
        <p>Paimon 的 committer 检查该贡献是否满足要求，并将代码合并到代码库中。</p>
      </div>
    </div>
  </div>
</div>

## 代码评审指南 {#code-review-guide}

每次评审都需要检查以下六个方面。**我们鼓励按顺序检查这些方面，以避免在尚未满足形式要求、或社区尚未就接受该改动达成共识时，就花费时间进行细致的代码质量评审。**

#### 1. 贡献是否描述清晰？ {#1-is-the-contribution-well-described}

检查该贡献是否描述得足够清晰，以支持一次良好的评审。琐碎的改动和修复不需要冗长的描述。如果实现完全符合此前在 issue 或开发邮件列表上的讨论，那么只需简短地引用该讨论即可。

如果实现与共识讨论中达成一致的方案不同，则在对该贡献进行任何进一步评审之前，都需要提供对实现的详细描述。

#### 2. 该贡献是否需要某些特定 committer 的关注？ {#2-does-the-contribution-need-attention-from-some-specific-committers}

某些改动需要特定 committer 的关注和批准。

如果该 pull request 需要特定的关注，则应由被标记的某位 committer/贡献者给予最终批准。

#### 3. 整体代码质量是否良好，是否达到了我们希望在 Paimon 中保持的标准？ {#3-is-the-overall-code-quality-good-meeting-standard-we-want-to-maintain-in-paimon}

- 代码是否遵循了正确的软件工程实践？代码是否正确、健壮、可维护、可测试？
- 在修改性能敏感的部分时，改动是否注意到了性能问题？
- 改动是否被测试充分覆盖？这些测试执行是否快速？
- 如果依赖发生了变更，NOTICE 文件是否已更新？

代码规范可参见 [Flink Java 代码风格与质量指南](https://flink.apache.org/how-to-contribute/code-style-and-quality-java/)。

#### 4. 文档是否已更新？ {#4-are-the-documentation-updated}

如果该 pull request 引入了新特性，则该特性应当被记录到文档中。

## 成为 Committer {#become-a-committer}

当你做出了足够的贡献后，你可以被提名为 Paimon 的 Committer。参见 [Committer](./committer)。
