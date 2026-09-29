# Investigation workflows

## Missing AI automatic reply

1. Query the conversation timeline for the selected environment and occurrence window.
2. Establish whether the inbound message reached message-hub and downstream routing services.
3. Identify assignment, workflow, suppression, handoff, or eligibility events before looking at code.
4. If a tool was expected, inspect the tool-call timeline and persisted execution separately.
5. If logs identify a failing service, trace ID, or exact error phrase, search bounded Sentry Issue
   summaries in the same environment and time window. Retrieve samples or one event only if needed.
6. Use GitHub search for exact error text, event name, handler, or configuration key found in logs or
   the confirmed Sentry event.
7. Report the earliest stage with evidence of divergence.
8. Keep the answer scoped to the non-reply cause, observable decision path, final cause summary, and
   repair options. Do not append unrelated full-chain or UID sections.

## Knowledge-card selection

1. Start from the exact inbound message and trace every evidenced rewrite, intent, retrieval, rank,
   threshold, filter, and selection event in timestamp order.
2. Record all candidate and selected card IDs exposed by logs, including scores and rejection reasons
   when present.
3. Resolve each selected card's basic metadata from logs or read-only data sources: card ID, title,
   type or source, status, score, selected reason, and bound action or tool.
4. When runtime records omit a decision stage, inspect the environment-mapped code to identify the
   applicable rule, but label it as code-path evidence rather than proof that it ran.
5. Produce a Mermaid decision flow plus a card summary table. Do not invent card names or scores.

## Tool did not run

Classify the deepest verified stage:

1. assistant text only;
2. model produced tool intent;
3. tool request was emitted;
4. pre-execution validation intercepted it;
5. upstream tool executed;
6. result returned to the model;
7. model produced the final user response.

Missing evidence at a later stage must not erase verified evidence at an earlier stage.

For every intercepted call, identify the exact guard or predicate, actual value, required value, and
why the condition evaluated false. Include the matched knowledge card, tool name, call arguments,
and returned result when those records exist.

## UID validation or interception

1. Fetch the authoritative incoming user for the conversation.
2. Fetch UID-validation logs for the same environment and time range.
3. Compare these values without type coercion:
   - UID written by the user;
   - UID parsed or injected into tool arguments;
   - UID expected by the validator;
   - authoritative incoming-user UID.
4. Determine whether the mismatch began during extraction, context reuse, argument construction, or
   validator lookup.
5. If a constant or unrelated ID appears, search the named service and exact ID/validation message in
   the environment-mapped code branch.

## Full-chain investigation

Run `investigate_conversation` first, then fill only material gaps with targeted tools. Build a single
ordered timeline rather than separate uncorrelated log dumps. Inspect source code after the runtime
break point is known so code explains observed behavior instead of replacing runtime evidence.
The final chain must begin with the inbound user message and follow evidenced services through card
selection, action execution, tool execution, and the final reply. Show the chain as Mermaid and
separately list the selected card, action, tool, and reply; mark absent stages explicitly.
Use Sentry only as a targeted error branch from a service already present in this chain, not as a
replacement for the conversation timeline.

## Open-ended log query

1. Fix environment, timezone, exact start/end, services, label filters, text filters, and page limit.
2. Use a fixed conversation log tool when a conversation ID is available; otherwise call
   `logs_query` with allowlisted labels and structured `all/any/none/regex/notRegex` text filters.
3. Start with counts or the smallest useful page. Follow `nextCursor` only when another page is
   materially needed, and preserve the original time range and filters on every page.
4. Report returned rows, limit, cursor/`hasMore`, and whether the evidence is incomplete. Never call
   the cursor a platform token and never ask for a Loki/Grafana credential.

## Sentry project inventory or issue ranking

1. For connected-project or visibility requests, call `sentry_list_projects` without inventing an
   environment. Report configured, visible, missing, returned, and incomplete counts separately.
2. For a top-Issue request, fix the environment, service/project, timezone, exact bounded window,
   status, sort, and result limit. Use the tool's 24-hour default only when the user asks for a recent
   ranking without another range, then report the exact returned boundaries.
3. Rank only by the tool-selected fixed sort. Use `windowEventCount` for the requested-window volume;
   keep lifetime `count`, affected `userCount`, first seen, and last seen as separate fields.
4. Do not retrieve samples for every ranked Issue. If root-cause analysis is requested, select only
   the materially relevant Issue, call `sentry_get_issue`, then retrieve a few event samples if needed.
5. Treat `incomplete: true`, rate limits, missing mappings, and projects outside the signed-in
   principal's repository range as coverage gaps rather than proof of absence.

## Standalone Sentry runtime error

1. Fix the environment, service/project, timezone, and exact time window. Preserve any Issue ID,
   Event ID, trace ID, request ID, conversation ID, UID, release, and exact error phrase as strings.
2. If an Issue ID is known, retrieve its sanitized summary first and fetch a small set of Issue
   events only when event-level evidence is needed. If an Event ID is known and the role permits
   event detail, retrieve that event directly. Otherwise search bounded Issue summaries by the
   strongest available correlation value.
3. If fixed correlation fields cannot express a release, level, custom Tag, or combined text query,
   use `sentry_query_issues`. Use `sentry_query_events` only for compact rows that materially close
   an evidence gap. Never pass raw Sentry syntax, and follow a returned cursor only when another page
   is required.
4. Confirm that the returned service/project and environment match the request. Do not treat a
   same-text error from another environment as the same incident.
5. Extract only the evidence needed to explain the failure: first/last seen, event time, exception
   type/message, relevant stack frames, release, trace/request correlation, and source URL.
6. If root cause is requested, correlate the selected event to bounded logs by trace/request ID and
   inspect the environment-mapped repository for the implicated handler or exact error. Label code
   paths without runtime confirmation as `代码路径证据`.
7. State whether the finding identifies the exact event, only an Issue family, or a probable match.
   Never attribute every Issue sample to the user's conversation.

## Production data query or export

1. Fix environment, tenant, timezone, exact start/end, filters, fields, and requested format.
2. Establish identifier semantics before correlation, especially KC versus Scene ID and
   conversation ID versus message `part_id`.
3. Use the smallest read-only MCP query that can answer the request. Validate counts and unmatched
   mappings before drawing a conclusion.
4. When fixed database tools cannot express an in-chat sample request, call `database_describe_views` and then
   `database_query_view` with the authoritative tenant, selected fields, structured filters, bounded
   row limit, and explicit offset. Do not request arbitrary SQL.
5. For knowledge-card and `message_hub` requests, apply the rules in
   [production-data-query.md](production-data-query.md).
6. For a file request, call `data_export_create` instead of manually collecting pages. Choose
   `logs`, `database_view`, or `conversation`; the conversation source performs the staged
   KC→Scene→ACK→complete-message workflow.
7. Call `data_export_status` with the returned `exportId` until it completes or fails. Deliver the
   returned MCP Resource and report validation counts, expiry, checksum, and `incomplete` state.
8. If a requested relationship is not among the approved views, report the missing server contract;
   do not bypass the service with SSH, PEM, arbitrary SQL, or locally supplied credentials.
9. End with any genuinely reusable, evidence-backed rule as a “可沉淀规则” note. Updating a target
   repository's `AGENTS.md` still requires an explicit user request; audit and export manifests are
   recorded automatically without mutating application repositories.

## Tool or source failure

- `DATA_SOURCE_NOT_CONFIGURED`: report that the MCP route exists but its upstream gateway is not
  connected.
- `PERMISSION_DENIED`: identify the missing source or environment scope; do not retry another
  environment to bypass it.
- `NOT_FOUND`: verify environment, ID, and time range before concluding the record does not exist.
- `RATE_LIMITED` or timeout: preserve completed evidence and report the investigation as partial.
- `UPSTREAM_AUTHORIZATION_FAILED` from Sentry: distinguish an invalid or revoked Token (`401`) from
  insufficient read permission (`403`); do not ask the user to paste a Token into the conversation.
- Empty result: widen the time range only when consistent with the user-provided occurrence time.
