---
name: linear-issue
description: Read a Linear issue by URL or identifier and summarize its details in Chinese. Use when the user gives a Linear URL, issue key like ENG-123, or asks to check/look up a Linear issue.
user-invocable: true
homepage: https://linear.app
metadata:
  { "openclaw": { "requires": { "env": ["LINEAR_API_KEY"] }, "primaryEnv": "LINEAR_API_KEY" } }
---

# linear-issue

Use the Linear GraphQL API to fetch one issue and summarize it in Chinese.

## Trigger

Use this skill when the user:

- provides a Linear issue URL
- provides an issue identifier like `TES-538`
- asks to "查 issue"、"看 Linear"、"拉 issue"、"读取 Linear issue"

## Input Parsing

Extract the issue identifier first.

- From a URL, read the issue key from the path or title slug.
- From free text, look for a key shaped like `<TEAM>-<NUMBER>`.
- Split `TES-538` into:
  - `TEAM=TES`
  - `NUMBER=538`

If you cannot find a valid identifier, ask the user for the Linear issue URL or issue key.

## Auth

Read the token from `LINEAR_API_KEY`.

Do not hardcode tokens in files, commands, or responses.

## Request

Important: keep the `curl` command on a single line. Do not use trailing `\`.

```bash
curl -s -X POST "https://api.linear.app/graphql" -H "Content-Type: application/json" -H "Authorization: $LINEAR_API_KEY" -d '{"query":"query($filter: IssueFilter) { issues(filter: $filter, first: 1) { nodes { id identifier title description url priority priorityLabel createdAt updatedAt dueDate state { name } assignee { name email } project { name } cycle { name } labels { nodes { name } } comments { nodes { body createdAt user { name displayName } } } } } }","variables":{"filter":{"number":{"eq":<NUMBER>},"team":{"key":{"eq":"<TEAM>"}}}}}'
```

Replace `<TEAM>` and `<NUMBER>` with the parsed values before execution.

## Validation

After the request:

- If `nodes` is empty, tell the user the issue was not found.
- If GraphQL returns `errors`, surface the useful error message briefly.
- Treat missing optional fields such as assignee, due date, labels, comments, cycle, or project as empty rather than failing.

## Response Format

Reply in Chinese using this structure:

```markdown
## <identifier>: <title>

**状态**: <state> | **优先级**: <priorityLabel> | **负责人**: <assignee-or-未分配>
**项目**: <project-or-无> | **Cycle**: <cycle-or-无> | **截止日期**: <dueDate-or-无>
**标签**: <comma-separated-labels-or-无>
**链接**: <url>

### 描述

<description-or-无>

### 评论

- <user> (<date>): <body>
```

If there are no comments, write `暂无评论`.

## Style

- Summarize faithfully; do not invent missing fields.
- Preserve useful structure in the description and comments.
- Keep the answer concise unless the user asks for the full details.
