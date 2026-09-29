# Investigation output formats

Choose exactly one template from the investigation focus. Do not append every available section to
each answer.

## Shared evidence rules

- Begin with `查询范围`: environment, timezone, exact time window, focus, and the applicable
  conversation ID, tenant, service/project, Issue/Event ID, trace ID, or request ID.
- Separate verified runtime facts, code-path evidence, and inference.
- Include exact timestamps and identifiers when available. Use `未查询到` instead of inventing data.
- End with `置信度与证据缺口` using `high`, `medium`, or `low`.
- Redact credentials, tokens, passwords, cookies, and unrelated personal data from tool arguments or
  results.
- Cite Sentry facts with service/project, environment, Issue ID or Event ID, event time, and source
  URL. Summarize only the stack frames or request fields needed to explain the failure; do not dump a
  complete event payload.
- A detailed decision process is the sequence of observable events and evaluated conditions. Never
  provide or claim hidden model chain-of-thought.
- In Mermaid diagrams, use solid arrows for evidenced transitions and dotted arrows for expected or
  code-derived transitions without runtime confirmation. Label dotted paths `未证实`.

## 1. 完整会话链路

Return these sections in order:

1. `查询范围`
2. `结论`: earliest verified break point and whether a final reply was produced.
3. `完整链路流程图`: a Mermaid `flowchart LR` beginning with the inbound user message, then every
   evidenced service, knowledge-card selection, action, tool, and final reply. Show skipped,
   intercepted, and missing stages explicitly.
4. `服务与证据时间线`:

   | 时间 | 服务/来源 | 事件 | traceId/requestId | 证据解释 |
   |---|---|---|---|---|

5. `知识卡片`: selected card ID, title, type/source, selection evidence, and binding.
6. `执行结果`: action name and status; tool name, execution stage, arguments, result or interception;
   final user-facing reply and timestamp.
7. `置信度与证据缺口`.

Do not collapse several services into one generic node when the logs identify them separately.

## 2. 知识卡片选择流程

Return only the card-selection investigation:

1. `查询范围`.
2. `选择结论`: which card or cards were ultimately selected, or the earliest selection stage with
   no result.
3. `卡片选择流程图`: a Mermaid `flowchart LR` from inbound message through every evidenced query
   rewrite, retrieval, candidate set, score/rank/threshold/filter decision, selected card, and bound
   action or tool.
4. `决策证据`:

   | 顺序 | 决策阶段 | 输入/候选 | 规则或分数 | 结果 | 证据来源 |
   |---|---|---|---|---|---|

5. `命中卡片基本信息`:

   | 卡片ID | 名称/标题 | 类型/来源 | 状态 | 分数 | 选中原因 | Action/Tool 绑定 |
   |---|---|---|---|---|---|---|

6. `未选中候选`: include only candidates and rejection reasons actually present in evidence.
7. `置信度与证据缺口`.

If a card field is unavailable, write `未查询到`; never derive its name from the assistant reply.

## 3. AI 未回复原因

Keep the answer limited to the non-reply diagnosis:

1. `查询范围`.
2. `最终原因总结`: state the verified or most likely reason and earliest failed stage.
3. `可观察决策过程`: ordered events and evaluated routing, eligibility, assignment, suppression,
   handoff, model, tool, and reply-publication conditions that materially explain the non-reply.
   Include a compact Mermaid flow only when it makes the failure branch clearer.
4. `关键证据`:

   | 时间 | 决策点/服务 | 实际值 | 期望值 | 结果 | 证据来源 |
   |---|---|---|---|---|---|

5. `修复方案`: smallest concrete fix first, then verification steps. Do not perform the fix in an
   analysis-only request.
6. `置信度与证据缺口`.

Do not add unrelated card inventories, UID tables, or source-code summaries unless they directly
explain the missing reply.

## 4. Tool 调用、拦截与返回

Return these sections in order:

1. `查询范围`.
2. `结论`: deepest verified tool stage: planned, emitted, intercepted, executed, result returned, or
   final reply generated.
3. `命中知识卡片`: card ID, title, selection evidence, and bound action/tool.
4. `Tool 调用明细`:

   | toolCallId | Tool | 阶段 | 调用时间 | 执行服务 | traceId/requestId |
   |---|---|---|---|---|---|

5. `调用入参`: show the evidence-backed JSON arguments, preserving IDs and UID types while redacting
   secrets.
6. `Tool 返回`: show the evidence-backed result or error payload; state explicitly when no upstream
   execution occurred and therefore no result exists.
7. `拦截条件判断`, when intercepted:

   | 条件/代码位置 | 实际值 | 要求值 | 是否成立 | 不成立原因 | 证据来源 |
   |---|---|---|---|---|---|

   Include supplied UID, tool-argument UID, validator-expected UID, and authoritative incoming-user
   UID when UID validation participates in the guard.
8. `置信度与证据缺口`.

Never infer tool execution from assistant wording. Distinguish model intent, emitted request,
interception, upstream execution, result returned to the model, and final response.

## 5. Sentry 异常查询

Return these sections in order:

1. `查询范围`: query mode, environment when applicable, service/project when applicable, timezone,
   exact time boundaries, status/sort or search correlation, and returned limit.
2. `结论`: for inventory, state visible and missing coverage; for rankings, state the leading Issue
   families and selected-window counts; for incident analysis, state the failing component,
   earliest verified failure, impact, and whether the match is an exact Event, an Issue family, or
   an inference.
3. `异常链路流程图`: include a compact Mermaid `flowchart LR` only for an incident path where it
   materially clarifies the failure. Do not create a synthetic flow for project inventory or a
   simple ranking.
4. `Sentry 证据`: choose the one table matching the query mode.

   Project inventory:

   | 服务 | 项目 slug | 项目状态 | 首次事件 | 可见性/映射结果 |
   |---|---|---|---|---|

   Top-Issue ranking:

   | 排名 | 服务/项目 | Issue ID | 状态 | 窗口内事件数 | 生命周期事件数 | 影响用户数 | 首次/最近发生 | 来源 |
   |---|---|---|---|---:|---:|---:|---|---|

   Targeted Issue/Event evidence:

   | 发生时间 | 服务/项目 | Issue ID | Event ID | 异常类型/摘要 | release | traceId/requestId | 来源 |
   |---|---|---|---|---|---|---|---|

5. `根因定位`: include only when the request asks for diagnosis. Separate verified runtime facts,
   code-path evidence, and inference. For sampled Issue events, state why each sample is relevant and
   never claim that all samples belong to the same conversation.
6. `修复与验证建议`: include for diagnosis requests; give the smallest corrective action first,
   followed by a bounded verification query. For inventory or ranking-only requests, replace this
   with a concise recommended drill-down only when useful. Do not modify production in an
   analysis-only request.
7. `置信度与证据缺口`.

Do not dump complete Sentry event payloads, breadcrumbs, headers, cookies, or unrelated personal
data. Keep `windowEventCount` distinct from lifetime `count`. Summarize only the fields and stack
frames required for the diagnosis.
