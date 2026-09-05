---
name: Audit and right-size agent activity on Cequence AI Gateway
description: Use query_tool_activity to answer who called what, what was blocked, which grants are unused, and where sensitive data was seen — then tighten the persona.
api: mcp/cequence-mcp.yml
endpoint: https://mcp.aigateway.cequence.ai/mcp
operations: [query_tool_activity, list_mcp_servers, list_tools_for_mcp_server, list_agent_personas, get_agent_persona, list_dlp_policies, update_agent_persona]
generated: '2026-09-05'
method: generated
source: https://docs.aigateway.cequence.ai/docs/guides/observability
grounding: >-
  Tool names and audit event fields are transcribed from the provider's published tools
  reference and observability guide. No input schema is asserted; live tools/list is auth-gated.
---

# Audit and right-size agent activity on Cequence AI Gateway

Use this for "what are our agents actually doing", for a least-privilege review, or when
someone reports that an agent is being blocked.

## The one tool that answers everything

`query_tool_activity` is the observability surface. It returns logs, KPI summaries, facets,
time-series trends, per-call detail, and sensitive-data findings per server. Every other tool
here is context for interpreting it.

## What is in an event

Each tool-activity event is emitted for **every** MCP tool invocation — allowed, blocked or
failed — and carries: `timestamp`, `request.id`, `tool.name`, `mcp.server_name`, `user.email`,
`client.ip`, `status` (`success` | `error` | `blocked` | `rate_limited`), `action` (`allowed` |
`blocked`), `action.reason`, `duration.total_ms`, `duration.upstream_ms`, and
`http.status_code`. There is also an MCP session id.

## Three questions worth asking every review

1. **"Which grants are unused?"** List the persona with `get_agent_persona`, list the server's
   tools with `list_tools_for_mcp_server`, then diff against the distinct `tool.name` values in
   the activity. A tool granted and never called is an over-grant — remove it with
   `update_agent_persona`, which applies partial edits.

2. **"What is being blocked, and why?"** Filter on `action: blocked` and group by
   `action.reason`:
   - `authz_denied` — the persona lacks the tool, or the caller is not in the right team.
   - `rate_limit_exceeded` — the limit is too tight for the real workload, or something is
     looping. Check the trend before raising it.
   - `auth_denied` — an expired or invalid credential; re-authenticate or reissue the access key.
   - `interceptor_denied` — a DLP or behavioral policy stopped it. Cross-check `list_dlp_policies`.

3. **"Where did sensitive data show up?"** Pull the findings view. A finding tells you a
   detector matched on a response, and whether the policy was in monitor, redact or block mode.
   Monitor-mode findings are the backlog: they are the calls that WOULD have been redacted.

## Performance triage

- High `duration.upstream_ms` means the upstream application is slow, not the gateway.
- `circuit_breaker_open` events mean the gateway stopped forwarding to a failing backend and is
  returning 503 immediately. It recovers on its own after a cooldown test request; repeated
  occurrences are an upstream health problem worth escalating.
- Correlate everything by `request.id` — one id ties the whole chain together.

## Where the events go

Audit events also export to Splunk (HEC), Datadog, OTLP over gRPC or HTTP, and syslog
(TCP/UDP/TLS), configured at the pool level, with per-MCP-server overrides. Route tool activity
to the SIEM for compliance and operational events to the monitoring stack. This is an export
surface, not a subscribable webhook API — there is nothing to subscribe to over MCP.

## Rules

- `query_tool_activity` is a read tool: it works for any tenant member.
- `update_agent_persona` is a write tool: operator role required, no dry run documented for it,
  and no undo. Write down the previous grants before you change them.
