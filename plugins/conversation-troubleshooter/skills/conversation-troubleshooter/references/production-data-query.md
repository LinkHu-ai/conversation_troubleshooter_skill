# Production data query and validation

Use these rules for knowledge-card, FAQ, KC, Scene, conversation, `message_hub`, or export requests.
They describe evidence semantics; they do not authorize arbitrary SQL or direct shared-host access.

## Scope and safety

- Fix the environment, tenant, timezone, exact start and end time, filters, requested fields, and
  output format before retrieving rows. Ask only for a missing value that changes the result and
  cannot be inferred from authoritative evidence.
- Every tenant-scoped query must apply the same tenant boundary at every database or service stage.
  Cross-database correlation is staged by identifiers; never perform an unbounded cross-tenant join.
- Use the server's read-only MCP tools. Do not expose or request PEM files, platform tokens,
  passwords, cookies, connection strings, or raw credential-bearing commands.
- For a request covered by an approved logical view, call `database_describe_views` and use
  `database_query_view`. Otherwise call `database_catalog_search`; it may dynamically enumerate
  any table inside a deployment-authorized database/schema scope. Select only a returned `objectId`,
  call `database_describe_object`, require `queryable=true`, and query only its returned live fields with
  `database_query_object`. Supply the authoritative tenant explicitly and select only fields
  required for the question. Never send SQL, connection details, or a tenant predicate as free
  text; the server owns those boundaries.
- Begin with field/schema meaning, a small sample, identifier types, indexes when exposed, and
  counts. Preserve IDs as strings. Treat partial, truncated, timed-out, or unmatched data as an
  evidence gap rather than absence.

## Knowledge-card semantics

The observed PostgreSQL relationship is:

```text
tags (level 1)
└─ tags (level 2, parent_id -> level 1)
   └─ faq_knowledge_items.issue_tag_id
      └─ faq_knowledge_item_scenes.item_id
         └─ faq_questions.scene_id
```

- A knowledge-card scene is represented by `faq_knowledge_item_scenes`.
- KC is `card_id`; Scene ID is the scene row `id`. Never use them interchangeably.
- `metadata.knowledge.selected_card_ids` has been observed as an array of Scene ID strings, not KC
  IDs. Confirm the JSON path and element type from a sample before treating this as universal.
- Scene question/answer evidence maps to `scene_description` and `reference_reply`; alternative
  phrasings come from `faq_questions.question` ordered by `sort_order`.
- A currently active card requires both scene and knowledge item to be enabled and not deleted, plus
  a valid effective period. Effective-period values are Unix milliseconds. Empty or `always` is
  active; a bounded custom range is inactive before `valid_from` or after `valid_to`, with zero as an
  unbounded side.
- When separating active and expired cards, freeze one `now_ms`. Validate that the groups do not
  intersect and their union equals the pre-split set. Classify future-not-started cards outside the
  current-active set and state that convention.

## Conversation and ACK semantics

The observed MySQL `message_hub` model uses:

- `conversation_informations` for the tenant-scoped conversation set in a time window;
- `conversations` for messages keyed by `conversation_id` and `part_id`, with `created_at` in Unix
  seconds;
- `conversation_ack_metadata` for ACK evidence, where `ack_part_id` maps to a message `part_id` and
  must also be restricted by tenant and conversation ID.

Use these conventions:

- KC matching first maps the KC `card_id` to same-tenant Scene IDs, then compares those Scene IDs to
  ACK `selected_card_ids`.
- “Recent N days” means a rolling `N x 24` hours unless the user gives calendar boundaries. Always
  report the exact start, end, and timezone.
- For a complete-conversation export, select conversation IDs from the scoped overview set, then
  return all messages belonging to those IDs; do not truncate a selected conversation merely
  because some messages lie just outside the overview boundary. State this convention.
- Excluding a KC means excluding the whole conversation when any ACK matches one of the mapped Scene
  IDs.
- User-supplied conversation IDs are parsed as strings, deduplicated, and intersected with tenant and
  time boundaries.

Validate candidate, excluded, and final conversation counts; message count per conversation;
unmatched ACK-to-message mappings; and excluded Scene IDs remaining in the final set.

## Export output

Use `data_export_create` for a bounded export operation and `data_export_status` to retrieve its
validation manifest and owner-bound MCP Resource:

- use Chinese headers, freeze/filter the header row, wrap long text, and size columns deliberately;
- treat Conversation ID, Part ID, KC, and Scene ID as identifiers. Preserve long IDs as text;
- order messages by conversation ID, message time, then message primary key, and keep one
  conversation in one output partition;
- keep only requested fields in data sheets and add a scope/readme sheet with environment, tenant,
  timezone, exact time range, filters, row counts, identifier mapping, and unmatched counts;
- re-open the result and validate sheet names, headers, row counts, long IDs, Chinese text, wrapping,
  and representative values.

The MCP supports `logs`, `database_view`, `database_object`, and `conversation` exports. Use
`database_object` only after catalog search and live description. `conversation` performs the
same-tenant KC→Scene mapping, bounded conversation-overview selection, ACK matching, and full-message
retrieval described above. XLSX files contain Chinese headers plus a query-scope sheet and are
reopened for validation before completion. Artifacts are encrypted at rest, creator-bound, carry a
SHA-256 checksum, and expire automatically.

Exports remain deliberately bounded by `maxRows`, file size, environment, tenant, and time scope;
they are not arbitrary SQL or unlimited dumps. Treat `incomplete=true`, a failed job, unmatched
records, or a row/file limit as an evidence gap. Narrow or split the request rather than bypassing
MCP through SSH or returning an unverified partial file.
