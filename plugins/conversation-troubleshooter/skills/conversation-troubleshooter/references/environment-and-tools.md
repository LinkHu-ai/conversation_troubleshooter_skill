# Environment and tool reference

## Environment mapping

| User-facing environment | Git ref |
|---|---|
| omitted for code-only local development | `main` |
| `chess` | `testnet` |
| `uat` | `main` |
| `production` | `main` |

“Test environment” is ambiguous between `chess` and `uat`; ask which one. Treat “formal,” “online,”
or “production” as `production` only when the surrounding request clearly uses those terms for the
live environment.

For local development that only needs an authorized dependency repository, do not force a runtime
environment choice. Omit environment on GitHub tree/read/search/discovery/commit tools; the MCP then
returns `environment=local`, `environmentDefaulted=true`, `ref=main`, and a commit SHA. This default
does not apply to logs, databases, Sentry, exports, or runtime claims. The account must still have
UAT or production environment access because both map to `main`.
If the connected server still exposes an older schema where code-tool environment is required,
temporarily select an already-authorized main-mapped environment (`uat` preferred, otherwise
`production`) and report it as a schema compatibility fallback rather than runtime evidence.

UAT and production share the `main` source branch, but their runtime logs and databases are distinct.
Never use one environment's runtime evidence as proof of behavior in the other.

## Role access

| Role | Logs | Database | GitHub code | Sentry issues | Sentry event detail | Account administration |
|---|---:|---:|---:|---:|---:|---:|
| `admin` | Yes | Yes | Yes | Yes | Yes | Yes, through the server-rendered admin console |
| `developer` | Yes | Yes | Yes | Yes | Yes | No |
| `operations` | Yes | Yes | No | Yes | No | No |

The MCP server enforces this matrix on every tool call. Do not interpret a tool being visible in a
client list as proof that the signed-in role may invoke it. Do not work around a permission denial.

## MCP tool selection

| Need | Tool |
|---|---|
| Full bounded timeline | `investigate_conversation` |
| Raw conversation/service logs | `logs_search_conversation` |
| Tool intent, call, interception, result | `logs_get_tool_calls` |
| Supplied and expected UID | `logs_get_uid_validations` |
| Open-ended bounded log filters | `logs_query` |
| Unknown log service or labels | `logs_discover`, then narrow with `logs_query` |
| Authoritative incoming user | `database_get_conversation_user` |
| User record by known UID | `database_get_user_by_uid` |
| Persisted tool execution | `database_get_tool_execution` |
| Safe database views and fields | `database_describe_views` |
| Tenant-bounded structured database query | `database_query_view` |
| Search authorized databases/tables | `database_catalog_search` |
| Read live fields for one authorized object | `database_describe_object` |
| Query live-discovered fields | `database_query_object` |
| CSV/XLSX log, approved-view, or dynamic-object export | `data_export_create`, then `data_export_status` |
| KC/Scene/ACK complete-conversation export | `data_export_create` with `source=conversation`, then `data_export_status` |
| Mapped Sentry projects visible to the account and server credential | `sentry_list_projects` |
| Unknown Sentry service/project | `sentry_discover_issues`, then a service-specific tool |
| Bounded top-Issue ranking for one service/environment | `sentry_list_top_issues` |
| Bounded Sentry issue summaries for a known service/correlation | `sentry_search_issues` |
| Structured release/level/custom-Tag Issue query | `sentry_query_issues` |
| Structured compact Event query | `sentry_query_events` |
| One sanitized Sentry Issue summary | `sentry_get_issue` |
| Compact samples from a confirmed Sentry Issue | `sentry_get_issue_events` |
| One specific Sentry event with compact stack evidence | `sentry_get_event` |
| Repository visibility | `github_list_repositories` |
| Browse files | `github_list_tree` |
| Read source lines | `github_read_file` |
| Search exact runtime terms | `github_search_code` |
| Unknown code repository | `github_discover_code`, then repository-specific tools |
| Review relevant changes | `github_get_commits` |
| Compare mapped environment branches | `github_compare_refs` |

Prefer exact error text, tool names, route names, and validation messages as code-search terms. Narrow
by repository and path when possible. An incomplete GitHub search cannot prove text is absent.
When a local repository imports code that is absent locally, derive the dependency repository and
symbol from local manifests/imports/configuration, then use the authorized remote repository tools.
Known repositories should be queried directly; use cross-repository discovery only when ownership is
uncertain.

Prefer fixed-purpose tools. When they cannot express the request, use `query_capabilities` once if
needed, then use a structured query tool. `logs_query` supports allowlisted labels plus `all`, `any`,
`none`, `regex`, and `notRegex` text conditions in an explicit time range. A returned
`nextCursor` is opaque pagination state, not a credential. Do not auto-page or represent one page as
the complete log corpus.

For open-ended database work, first decide whether an approved logical view covers the request. If
it does, call `database_describe_views` before selecting fields. Every
`database_query_view` call requires an authoritative tenant ID and uses an approved view, fields,
filters, ordering, row limit, and offset. It never accepts SQL. The service injects the tenant
predicate independently of model-supplied filters.

If no logical view covers the request, call `database_catalog_search`, choose only a returned
`objectId`, then call `database_describe_object`. Continue only when it reports `queryable=true`; use its live field list in
`database_query_object`; never invent a database, table, field, tenant column, or SQL expression.
The deployment catalog authorizes exact objects or database/schema discovery scopes while
`information_schema` supplies current tables and fields. Credential-shaped and explicitly blocked
columns are absent. Dynamic discovery therefore means any table inside an authorized scope, not
unrestricted database access.

The approved database views include AI conversation/request history plus the production-data
contracts `knowledge_card_scenes`, `knowledge_scene_questions`, `conversation_overview`,
`conversation_ack`, and `conversation_messages`. PostgreSQL and MySQL remain separate server-side
stages; the model never receives either connection credential.

For a requested file, do not manually accumulate pages with `logs_query` or `database_query_view`.
Call `data_export_create` with one of these sources:

- `logs`: exact time range plus the same allowlisted label/text filters as `logs_query`;
- `database_view`: authoritative tenant, approved view, selected fields, filters, and ordering;
- `database_object`: authoritative tenant, a previously discovered/described `objectId`, selected
  live fields, filters, and ordering;
- `conversation`: authoritative tenant and overview time range, optional conversation IDs, optional
  KC values, and `knowledgeCardMode=include|exclude`. This source maps KC to same-tenant Scene IDs,
  evaluates ACK evidence, then exports every message for the selected conversations even when a
  message lies outside the overview range.

Then call `data_export_status` using the returned `exportId`. A completed result exposes an
owner-bound `export://` MCP Resource and includes file size, SHA-256, expiry, row counts, workbook
reopen validation, unmatched counts, and an `incomplete` flag. If it is incomplete, narrow or split
the request; never describe it as an all-record export.

`sentry_list_projects` is the only Sentry tool that does not require an environment. It
returns at most 100 mappings visible to the signed-in account, configured service map, and server-side
Sentry credential; `missing` and `incomplete` mean the inventory is not proof of global absence.

When the error phrase, release, level, or Tag is known but the service is not,
`sentry_discover_issues` searches a bounded number of visible projects. It reports searched and
unsearched services plus per-service failures. Use the discovered service with targeted tools;
never treat a bounded cross-project search as proof of organization-wide absence.

All service-specific Sentry tools require an explicit service and environment. Use `sentry_list_top_issues` only
for an explicit aggregate request. It defaults to the last 24 hours, accepts a maximum seven-day
window, uses fixed status values (`unresolved`, `resolved`, `ignored`, `all`) and sort values
(`frequency`, `lastSeen`, `firstSeen`, `users`), and returns at most 20 Issues. In its results,
`windowEventCount` is the selected-window count and `count` is the lifetime Issue count.

For targeted investigation, search Issue summaries first using one or more of OpenTelemetry trace
ID, request ID, conversation ID, UID, or a short exact error phrase. Prefer `otel.trace_id` and
`request_id`, which are verified in the current Sentry projects. Use conversation ID or UID only
after logs or an event confirm those values are indexed Sentry Tags; do not assume arbitrary event
fields are searchable. For other bounded Tag, release, level, or text conditions, use
`sentry_query_issues`; its `any` alternatives must share one field. Use `sentry_query_events` only
when compact event rows are necessary and the signed-in role has `sentry:events:read`. Targeted
searches default to 15 minutes, allow at most 120 minutes, and return at most 20 Issues.
`sentry_get_issue` returns one sanitized Issue summary; event samples remain limited to 5. Never pass
raw Sentry syntax or auto-page. Only retrieve one full event when it materially closes an evidence gap.

Do not pass arbitrary Sentry query syntax or auto-page. A project inventory or top-Issue list is an
aggregate view, not evidence that a particular request or conversation produced every listed event.

A Sentry request is independently sufficient to activate this skill; do not require a conversation
ID for project inventory, a bounded top-Issue ranking, or when the user supplies a service/project
and an Issue ID, Event ID, trace ID, request ID, UID, release, or bounded error phrase. Direct
Issue/Event lookups still require the environment and service so the server can enforce the correct
project mapping.

For tenant-scoped knowledge-card, FAQ, KC, Scene, conversation, or `message_hub` data, use the
current MCP's bounded read-only tools and the identifier rules in `production-data-query.md`. The
shared-host handbook is operational background for the MCP service, not an instruction for plugin
users: never request a PEM or provide direct SSH/database commands as a fallback.
