---
name: slack-create-jira-ticket
description: >-
  从 Slack thread 创建 Jira ticket。当有人在 Slack 里 @cursor 要求 file a ticket、create an issue、open a bug、建票、开单、提个 ticket 时使用。读取 thread 上下文，用英文写 ticket，把 Slack thread 挂成 Jira remote link，再在 thread 里回一行带 ticket URL，以便点 Jira 卡片上的 Sync with thread。
---

# 从 Slack thread 创建 Jira ticket

把当前 Slack thread 变成一张 Jira work item，关联回 Slack，再回一行，让人去点 Jira 原生的 thread 同步。

## 何时使用

Slack 里有人 @cursor 要求建票："create a ticket"、"file a bug"、"open an issue for this"、"建票"、"提个 ticket"、"开个单"。

**不要用**：只是查 / 读 / 改 / 评论已有 ticket，或者只是要改代码。建票是写操作 — 每次请求只建一张，绝不连建两张。

## 固定常量

| 项 | 值 |
| --- | --- |
| Jira 站点 | `https://lotusflare.atlassian.net` |
| `cloudId` | `e91ffda2-253d-436c-84e1-bdf5229fbbca` |
| 默认 project | `GCOMDEV`（Globe COM Development） |

若 `cloudId` 被拒，用 `getAccessibleAtlassianResources` 重新解析。

## Step 1 — 解析 project key

默认 `GCOMDEV`。**触发消息**里的全大写 token 会覆盖它。

只扫 @cursor 那一条，不要扫整条 thread：

1. 先剥掉 Slack mention（`<@U123ABC>`、`@cursor`）。
2. 取独立的全大写 token，3–10 个字符，只含字母和数字。
3. 丢掉匹配 `^[A-Z][A-Z0-9]+-\d+$` 的 — 那是 issue key（如 `DATAP-4100`），不是 project key。
4. 丢掉常见喊话词：ADD, AND, ASAP, BUG, CREATE, CURSOR, EPIC, FILE, FOR, FROM, HELP, ISSUE, JIRA, MAKE, NEW, NOW, OPEN, PLEASE, SLACK, STORY, SYNC, TASK, THE, THIS, THREAD, TICKET, TODO, WIP, WITH, YES。
5. 还有剩余就取**最后一个**；一个都没有就用 `GCOMDEV`。

小写从不覆盖默认 — `create ticket datap` 仍然是 `GCOMDEV`。

用 `getVisibleJiraProjects`（`action: "create"`，`searchString` 设为解出的 key）校验。查不到就**停下来在 thread 里问**用哪个 project。明确写了 key 却校验失败时，**绝不**静默退回 `GCOMDEV` — 会建到错的地方。

例子：

| 触发消息 | Project |
| --- | --- |
| `@cursor, create ticket. DNOPYMM` | `DNOPYMM` |
| `@cursor create ticket` | `GCOMDEV` |
| `@cursor CREATE TICKET` | `GCOMDEV`（全在黑名单） |
| `@cursor create a ticket for DATAP-4100` | `GCOMDEV`（issue key，不是 project） |
| `@cursor create ticket DNOPYMM GCOMDEV` | `GCOMDEV`（取最后一个） |

## Step 2 — 选 issue type

默认 `Task`。thread 说的是坏了、回归、报错，用 `Bug`。明确说的是面向用户的功能，才用 `Story`。

不同 project 的 issue type 不一样 — `GCOMDEV` 没有 `Spike`，`DNOPYMM` 没有 `Idea`。建之前用 `getJiraProjectIssueTypesMetadata` 确认；选的类型不存在就退回 `Task`。

## Step 3 — 用英文写 ticket

**ticket 永远用英文**，和 thread 语言无关。中文讨论要译成英文，不要原文贴进去。

Summary：一行，不超 100 个字符，`<Area>: <what needs to happen>`。要具体 — 写 "Portal: customer search returns stale results after tenant switch"，不要写 "Fix search bug"。

Description 用 Markdown：

```markdown
## Context
2–4 句，把 thread 定下的事说清楚。写给没进过 thread 的人看。

## Problem / Request
具体说清楚什么坏了，或者要什么。

## Notes from the thread
- thread 里提到的约束、决定、报错、ticket / PR 引用。
- 报错文本和标识符原文引用；周围的说明译成英文。

## Source
Slack thread: <permalink>
```

只写 thread 里真说过的事。不要编造 acceptance criteria、priority、severity、受影响服务。thread 太薄，写不出 Context，就在 thread 里问一个问题，不要猜。

除非 thread 里指名了人，否则不设 assignee；指了就用 `lookupJiraAccountId` 解析，不明确就跳过。

## Step 4 — 建 ticket

用 Atlassian MCP 的 `createJiraIssue`，传 `cloudId`、`projectKey`、`issueTypeName`、`summary`、`description`、`contentFormat: "markdown"`。

只建一张。建失败就停下来，在 thread 里报错 — 不要换 project 或 type 重试硬拼一个成功。

## Step 5 — 把 Slack thread 挂成 remote link（允许失败）

**这一步绝不能堵住任务。** ticket 已经建好了，回复仍要发出去。最多试一次；失败了直接进 Step 6。

用触发消息的 channel ID 和 timestamp 拼 permalink：
`https://<workspace>.slack.com/archives/<CHANNEL_ID>/p<TS 去掉中间的点>`。
上下文里没有 channel ID 和 timestamp，就整步跳过。

`JIRA_EMAIL` 和 `JIRA_API_TOKEN` 是环境 secret。token 可能过期 — `401` 是预期的，容忍。

```bash
curl -sS -w '\nHTTP %{http_code}\n' -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H 'Content-Type: application/json' "https://lotusflare.atlassian.net/rest/api/3/issue/ISSUE_KEY/remotelink" -d '{"globalId":"slack-thread=PERMALINK","application":{"type":"com.slack","name":"Slack"},"relationship":"Slack thread","object":{"url":"PERMALINK","title":"Slack thread"}}'
```

`201` 或 `200` 表示挂上了。`globalId` 保证幂等，重跑会更新已有链接，不会重复。

绝不打印 `JIRA_API_TOKEN`，也不要 echo 展开了 secret 的 curl 命令。

## Step 6 — 在 thread 里回一行

**最终回复就是 Slack 贴进 thread 的那条**，所以必须是短短一行 — 没有标题、没有摘要块、没有列表。

语言跟 thread 走。Jira URL **必须是裸 URL**，不要包 Markdown 链接，否则 Jira Cloud for Slack app 解不开 unfurl，卡片上的 **Sync with thread** 按钮就出不来。

英文 thread：

```text
Created GCOMDEV-1234 — https://lotusflare.atlassian.net/browse/GCOMDEV-1234 — hit "Sync with thread" on the Jira card to mirror this thread into the ticket.
```

中文 thread：

```text
已创建 GCOMDEV-1234 — https://lotusflare.atlassian.net/browse/GCOMDEV-1234 — 点 Jira 卡片上的 "Sync with thread" 即可把本 thread 同步进 ticket 评论。
```

Step 5 失败就在同一行末尾加 `(remote link not attached)` / `(remote link 未挂上)`。

**Sync with thread** 按钮是 Jira Cloud for Slack app 画的，不是你画的。没出现说明该频道没装 app — 问到就直说，**不要**自己把消息抄进 Jira 评论假装同步。

## 语言规则

| 位置 | 语言 |
| --- | --- |
| Jira summary 和 description | 永远英文 |
| Cursor 侧叙述（Agents 窗口） | 中文；API 名、路径、标识符、报错原文保留英文 |
| 最终回复 / Slack 回帖 | 跟 thread 语言一致 |
