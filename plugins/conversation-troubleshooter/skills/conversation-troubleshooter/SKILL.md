---
name: conversation-troubleshooter
description: Dynamically discover and investigate AI-support behavior across authorized logs, tenant-scoped databases, Sentry projects, local-to-remote dependency code, and environment-specific code, including validated exports. Use for open-ended data/log/code/error searches, cross-repository development analysis, conversation IDs, knowledge-card or UID problems, missing AI replies, tool execution or interception, and Sentry project, Issue, Event, trace, release, or runtime queries in dev, chess, UAT, or production.
---

# Conversation Troubleshooter

## Activation

Use this skill for a conversation investigation, an open-ended authorized log/database/code search,
or a standalone Sentry inventory, aggregate, or incident investigation. A conversation ID is not required when the user asks which Sentry
projects are connected, requests a bounded top-Issue ranking for a service/environment, or supplies
a Sentry service/project plus an Issue ID, Event ID, trace ID, request ID, UID, or bounded error
phrase. Mentions of Sentry project visibility, top errors, error counts, an Issue/Event URL, runtime
exceptions, stack traces, crash/error spikes, release failures, or a service error in one of the
supported environments are sufficient to activate the skill.

Skill/MCP implementation maintenance is a local code-development task, not an incident query.
When the user explicitly requests edits to this skill or its MCP service, inspect and test the local
source without requiring an MCP login. Authentication below applies before protected runtime/data
queries; maintenance itself does not authorize querying live business data or deploying changes.

## Mandatory step zero: authenticate before reasoning

Authentication is the first action for every matching request. Before describing the investigation
scope, interpreting the incident, choosing a time window, reading plugin or local files, inspecting
cache paths, searching code, or announcing an investigation plan, check whether the host has an
authenticated `conversation_troubleshooter` MCP connection.

- If the connection state is unknown, make only the minimum MCP connection or protected-tool attempt
  needed for the host to determine it. Do not investigate the incident first.
- If the host reports that authentication is required, unavailable, or disconnected because there
  is no active login, the only user-facing preamble for that turn is:
  “插件尚未登录。正在启动 MCP 登录授权；若客户端未自动打开页面，我会显示可用的手动登录
  链接。登录成功后我再继续本次查询。”
- In the same turn, immediately initiate only the host-managed MCP OAuth operation or native
  `Authenticate` control. Never use a general browser tool, Codex in-app browser, computer-use tool,
  webview navigation, `open` command, or URL-opening shell command to open an OAuth page. Browser
  selection and callback handling belong exclusively to the MCP host.
- If the native OAuth operation returns an exact authorization URL, include it in the response as
  `若未自动打开：[手动打开登录页](<exact authorization URL>)`. Preserve its complete query string,
  including state, client ID, redirect URI, and PKCE challenge; first verify that its origin matches
  the authorization server discovered from MCP metadata. Never fabricate, shorten, cache, or reuse
  this URL. If the host does not expose the generated URL, state that no safe manual link is
  available and give the host-native login action instead; a fixed `/oauth/authorize` link cannot
  complete OAuth.
- After initiating login, keep the current turn, the host-managed OAuth operation, and its loopback
  callback listener alive for up to five minutes. Treat the login command or authorization session
  as the primary status signal and wait for it in bounded intervals of at most 30 seconds. Do not
  finish the turn or terminate a CLI login process merely because the browser page has opened.
- When the host does not expose callback completion directly, poll only its MCP authentication or
  connection state every 10-15 seconds during the same five-minute window. Do not run the incident
  query as a login probe and do not ask the user to send an “已登录” message while this window is
  active.
- While authentication is pending, do not read files, rediscover tool names, explain environments,
  select a time window, or perform any part of the investigation.
- If the host already opened the OAuth page, do not launch a duplicate. If it cannot open one, give
  the exact host-provided authorization link when available, followed by the appropriate native MCP
  login action: `codex mcp login conversation_troubleshooter` for Codex CLI or
  `agent mcp login conversation_troubleshooter` for Cursor CLI.
- A `http://127.0.0.1...` or `http://localhost...` redirect after successful login is the MCP
  client's loopback callback, not another login page. Do not open, copy, follow, or navigate to that
  callback independently. Keep the native OAuth operation alive so its local listener consumes the
  authorization response. If the authorization tab itself lands on the callback, do not open a
  second tab or an in-app browser panel for it.
- As soon as the callback or authentication-state probe reports success, resume the original request
  automatically without asking the user to repeat it or send a confirmation message, and retry the
  intended tool call once. If five minutes elapse without success, close only the expired login
  operation and report that login timed out; preserve the original request so a new login attempt can
  resume it.
- If the MCP connection is already authenticated, continue silently without discussing login.

## Investigate within the authenticated role

Build an evidence-backed explanation of what happened in one conversation. Use the
`conversation_troubleshooter` MCP tools; do not request or expose upstream GitHub, Sentry,
log-platform, database, or MCP credentials.

Never ask the user to type a username, email, password, authorization code, or token into the
conversation. Never send users to `/admin/login`; that page creates an administrator web session,
not a plugin OAuth session. Only show the current host-generated OAuth authorization URL under the
manual-link rule above; do not fabricate or reuse one because each flow contains client-specific
state and PKCE parameters.

Treat `PERMISSION_DENIED` as an enforced service-side role decision. Do not retry through another
tool or ask for upstream credentials to bypass it. Administrators and developers may query logs,
databases, code, Sentry issue summaries, and individual Sentry event details. Operations users may
query logs, databases, and Sentry issue summaries, but must not use GitHub tools or request full
Sentry event details.

## Production-query invariants

- Respond in Chinese unless the user explicitly requests another language.
- Remain read-only. Runtime, log, database, Sentry, and export work must be bounded by environment,
  tenant, timezone, and an explicit time range. For code-only local development, the GitHub code
  tools may omit environment and then deliberately use `main`; treat the returned
  `environment=local`, `environmentDefaulted=true`, `ref=main`, and commit SHA as the code basis,
  not as proof about production runtime behavior. For
  business-data evidence, establish the tenant from the request or an authoritative conversation
  record and never combine records across tenants. If an MCP tool does not accept a tenant field,
  verify the tenant in returned evidence when available and state that limitation.
- Use only the authenticated MCP tools. The MCP service owns shared-host, database, log, GitHub, and
  Sentry credentials. Do not instruct plugin users to upload a PEM, connect to the shared host, or
  paste platform tokens into the conversation.
- Start with the smallest query that can answer the question. For data/export requests, establish
  object ownership, live schema or field meaning, sample records, indexes or identifiers, and counts before requesting
  rows. Do not issue arbitrary SQL, raw LogQL, or raw credential-bearing Sentry requests through a
  model-facing tool. Use the MCP's structured query tools when fixed-purpose tools cannot express
  the request.
- Every completed answer must state its query basis and validation: environment, tenant when
  applicable, timezone, exact time boundaries, filters, counts or returned limits, ID mapping, and
  unmatched or incomplete records.
- Read [references/production-data-query.md](references/production-data-query.md) for knowledge-card,
  conversation, ACK, export, and identifier semantics.

## Establish scope

Before querying, identify:

- environment: `dev`, `chess`, `uat`, or `production`; explicit “dev/开发环境” always means `dev`,
  never `chess`. This is optional only for a code-only local development dependency analysis, where
  omission defaults GitHub code reads to `main`;
- timezone and approximate occurrence time or an explicit time range;
- tenant when the request queries or exports tenant-scoped business data;
- investigation focus: full chain, knowledge-card selection, missing AI reply, tool execution, or
  standalone Sentry error analysis;
- conversation focus: the complete conversation ID;
- Sentry focus:
  - project inventory needs no environment or service;
  - a top-Issue ranking needs environment, service, status/sort intent, and a bounded time window;
  - an Issue lookup needs environment, service, and Issue ID;
  - an Event lookup needs environment, service, and Event ID;
  - a targeted search needs environment, service, and at least one trace ID, request ID,
    conversation ID, UID, or short exact error phrase.

After authentication succeeds, treat an explicit skill invocation with no incident details as a
request to start the preset intake. Do not replace it with a generic sentence asking the user to
provide an ID, environment, and time. Present the following interaction immediately:

```text
请选择或填写排查条件（可按“环境 / 目标 / 时间 / 会话ID”回复）：

1. 环境
   1) dev（开发环境，devnet 分支）
   2) chess（测试环境，testnet 分支）
   3) uat（UAT 环境，main 分支）
   4) production（正式环境，main 分支）

2. 排查目标
   1) 完整会话链路
   2) 知识卡片选择流程
   3) AI 未自动回复
   4) Tool 调用、拦截与返回
   5) Sentry 异常查询

3. 时间范围
   1) 最近 15 分钟
   2) 指定发生时间
   3) 自定义起止时间

4. 定位线索
   会话类请输入完整会话 ID；Sentry 类可填写项目列表、Top 错误，或服务/项目及 Issue ID、
   Event ID、trace ID、request ID、UID、错误关键字
```

Use the host's structured choice controls when they are available and can collect these values in
one interaction. Otherwise render the numbered text exactly as above. When the request contains
only some scope values, retain those values and present only the missing groups using the same
labels and options. Never fall back to an unstructured “请提供...” prompt.

For a recent incident without a range, offer the last 15 minutes as the first choice rather than
silently assuming it. For a historical incident, require the approximate occurrence time. Treat
“test environment” as ambiguous and ask the environment choice.

Treat IDs and UIDs as strings, preserving leading zeros and hyphens. Never substitute an admin ID,
workspace ID, assignee ID, or prior-conversation UID for the incoming user's UID.

## Investigate

Start with `investigate_conversation` for a full-chain request. For a standalone Sentry request, do
not call conversation tools merely to satisfy a conversation-shaped workflow: route directly by the
provided correlation. Use targeted tools when the user asks one precise question or when the
aggregate result exposes a gap:

- `logs_search_conversation` for the message and routing timeline;
- `logs_get_tool_calls` for model tool intent, actual invocation, interception, and response;
- `logs_get_uid_validations` for supplied UID, expected incoming-user UID, and validation outcome;
- `logs_query` for a bounded combination of allowlisted log labels and text filters when the fixed
  conversation log tools cannot express the request. Preserve `nextCursor` and request another page
  only when it is materially needed;
- `logs_discover` when the service or useful label is unknown. Search one exact runtime clue across
  the authorized time window, inspect the returned service/label facets, then narrow subsequent
  `logs_query` calls. Treat unsearched pages as incomplete;
- `database_get_conversation_user` to establish the authoritative incoming user;
- `database_get_tool_execution` to distinguish emitted calls from persisted executions;
- `database_describe_views` before the first open-ended database request, then
  `database_query_view` with an explicit tenant, selected fields, structured filters and bounded
  result limit. Never invent a view or field and never request arbitrary SQL;
- `database_catalog_search` when no approved logical view is known. Search by business concept,
  database, schema, or table, then call `database_describe_object` for the chosen `objectId` before
  `database_query_object`. The catalog may enumerate any live table inside a deployment-authorized
  database/schema scope. Require `queryable=true`, use only returned live fields, always pass the authoritative tenant, and
  never guess an object ID, table, column, tenant column, or SQL fragment;
- `data_export_create` when the user asks for a CSV/XLSX artifact rather than an in-chat sample. Use
  `source=logs` for a complete bounded log filter, `source=database_view` for one approved logical
  view, `source=database_object` for one dynamically discovered and described object, and
  `source=conversation` for the staged KC→Scene→ACK→complete-message workflow. Always
  supply environment, timezone, exact time bounds when applicable, tenant for business data,
  requested format, and a deliberate `maxRows` safety bound;
- `data_export_status` after creation until the job reaches `completed` or `failed`. On completion,
  report the returned validation counts, `incomplete` flag, expiry, SHA-256, and owner-bound MCP
  resource link. Never decode or paste a large export into model context merely to deliver it;
- `sentry_list_projects` when the user asks which mapped Sentry projects are visible. This inventory
  does not require an environment or service and must remain bounded to at most 100 results;
- `sentry_discover_issues` when the error clue is known but the owning service/project is not. Search
  visible mapped projects with one bounded structured query, then use the matched service with the
  targeted Issue/Event tools. Report unsearched services and per-service failures;
- `sentry_list_top_issues` for an explicit recent/top/error-ranking request. Supply one service and
  environment, a bounded time range, fixed status (`unresolved`, `resolved`, `ignored`, or `all`),
  fixed sort (`frequency`, `lastSeen`, `firstSeen`, or `users`), and at most 20 results;
- `sentry_search_issues` only after a relevant service and correlation value are known, to find
  bounded runtime-error summaries in the incident window. Prefer an evidenced OpenTelemetry trace
  ID or request ID; use conversation ID or UID only when runtime evidence shows the service sends
  that value as an indexed Sentry Tag;
- `sentry_query_issues` when fixed Sentry tools cannot express a bounded release, level, tag or text
  query. Use only `text/all/any`; Issue `any` filters must use the same field;
- `sentry_query_events` only for a materially necessary compact Event search and only when the
  signed-in role permits Event access. It is not a bulk export tool;
- `sentry_get_issue` for one sanitized Issue summary after its service, environment, and Issue ID
  are known;
- `sentry_get_issue_events` for at most a few compact samples from a confirmed Issue;
- `sentry_get_event` only when one specific event is materially needed and the signed-in role permits it;
- `github_discover_code` when an exact runtime term, route, symbol, error string, or tool name is
  known but the owning repository is not. It searches bounded environment-correct branches across
  authorized repositories and reports unsearched repositories. Once a repository is identified,
  narrow with `github_search_code`, `github_list_tree`, and `github_read_file`;
- repository-specific GitHub tools directly when logs already identify the owning service.

For local-to-remote dependency analysis, inspect the local repository first and extract the strongest
dependency clues: repository or package name, imported symbol, route, RPC method, event/topic, schema,
or exact error text. If the authorized dependency repository is known, use its repository-specific
GitHub tools directly; otherwise use `github_discover_code` with repository hints, then narrow with
tree, search, and bounded file reads. When the user has not selected a runtime environment, omit the
code-tool environment so the MCP uses `main`; do not invent `production` merely to obtain the same
branch. Continue recursively when the remote implementation calls another authorized repository,
and distinguish local-file evidence from remote `main` evidence in the conclusion. Never describe a
bounded or incomplete search as a full repository scan.
If an older connected MCP schema still requires `environment`, use an environment already authorized
for the account that maps to `main` (`uat` when available, otherwise `production`) only as a
compatibility fallback, and label that fallback explicitly. Do not use the selected name as runtime
evidence.

For a Sentry-only request:

- use `sentry_list_projects` directly for project-visibility or connected-project requests;
- use `sentry_list_top_issues` directly for explicit top-error, frequency, recency, new-Issue, or
  affected-user rankings. Use the requested window, or the tool's 24-hour default when the user asks
  for a recent ranking without another range, and report the exact returned boundaries;
- use `sentry_get_issue` when service, environment, and Issue ID are known, then retrieve up to a few
  samples with `sentry_get_issue_events` only when event-level evidence is materially needed;
- use `sentry_get_event` directly when service, environment, and Event ID are known and the role
  permits event detail;
- otherwise use `sentry_search_issues` with the supplied trace ID, request ID, conversation ID, UID,
  or short exact error phrase, then retrieve samples only from a relevant Issue;
- if the service is unknown, use `sentry_discover_issues` before any service-specific lookup;
- when those fixed fields cannot express the user's release, level, custom Tag, or combined filter,
  use `sentry_query_issues`; use `sentry_query_events` only when compact event-level rows are
  materially required and the role permits it;
- correlate the selected Issue/Event back to logs or code only when the user requests root-cause
  analysis or the Sentry evidence alone cannot establish the failing component.

Do not use a top-Issue ranking as a substitute for a targeted lookup when the user supplied a
correlation or Issue/Event ID. For an open-ended request, call `query_capabilities` only when the
tool contract is not already known, then use the appropriate structured query tool. Never request
raw Sentry search syntax, paginate automatically, or retrieve full events before issue summaries
establish relevance. Preserve service, environment, time boundaries, filters, returned limits/cursor
state, Issue ID, Event ID, timestamp, and source URL for every Sentry fact used in the conclusion.

For knowledge-card selection, reconstruct the observable path from the incoming message through
query rewrite, retrieval, candidate scoring or filtering, selected card IDs, card metadata, and any
bound action or tool. Use only stages supported by runtime or code evidence and mark code-only paths
as unverified at runtime.

For an export request, prefer the server export contract over manually auto-paging query tools. A
completed XLSX is already reopened and checked by the server; still inspect its validation manifest
for tenant boundary, candidate/final conversation counts, message counts, unmatched ACK mappings,
excluded Scene IDs, truncation, and row-limit completeness. `completed` does not mean complete when
`incomplete=true`. Export resources are encrypted at rest, bound to their creator, and expire; they
contain no upstream database, log, Sentry, SSH, or MCP credential.

After a production-data task, identify any reusable, evidence-backed schema or identifier lesson
that was not already in the references. Include it as a short “可沉淀规则” note in the answer. Do
not automatically edit an application's `AGENTS.md` or repository; persist the rule only when the
user explicitly requests that repository change. MCP audit records and the export validation
manifest remain the automatic, non-mutating task history.

Correlate by `conversationId`, `traceId`, `requestId`, `toolCallId`, UID, timestamp, service, and Git
commit SHA. Do not infer that a tool ran merely because the assistant response mentions it.

Map the selected focus to exactly one focus-specific output template in
`references/output-format.md`. “Detailed decision process” means observable events, evaluated rules,
and code conditions supported by evidence; never provide or invent hidden model chain-of-thought.

Read [references/investigation-workflows.md](references/investigation-workflows.md) when selecting a
diagnostic path. Read [references/output-format.md](references/output-format.md) before presenting a
completed investigation. Read [references/environment-and-tools.md](references/environment-and-tools.md)
when choosing an environment, branch, or MCP tool. Read
[references/production-data-query.md](references/production-data-query.md) for any knowledge-card,
FAQ, KC, Scene, conversation, `message_hub`, or export request.

## Evidence rules

- Distinguish “model planned a tool,” “tool request emitted,” “request intercepted,” “upstream tool
  executed,” and “tool result returned to the model.”
- Give database conversation-user evidence precedence over a UID guessed from message text.
- Treat `partial: true`, `incomplete: true`, timeouts, unavailable gateways, and truncated code search
  as incomplete evidence, not proof of absence.
- Cite exact timestamps and source identifiers. For code findings include repository, environment
  (or `local/default-main`), ref, commit SHA, file path, and relevant line range.
- For Sentry findings cite service/project, environment, Issue ID or Event ID, occurrence time, and
  source URL. Treat an Issue aggregate as evidence of an error family, not proof that the exact
  conversation produced every sampled event.
- In top-Issue results, `windowEventCount` is the selected-window total while `count` is the Issue's
  lifetime count. Never present one as the other. Treat `incomplete: true` and project `missing`
  entries as explicit coverage limitations.
- State the earliest verified break point, likely cause, confidence, and the missing evidence needed
  to raise confidence.
- Respond in Chinese unless the user explicitly requests another language.
- Remain read-only. Do not change production configuration, data, users, workflows, or repository
  contents unless the user separately requests and authorizes that work.
