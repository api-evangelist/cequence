---
name: Onboard a least-privilege agent on Cequence AI Gateway
description: Create an MCP server from a spec, compose a least-privilege agent persona, attach a DLP policy, and hand back a connect URL — entirely over the Cequence AI Gateway MCP server.
api: mcp/cequence-mcp.yml
endpoint: https://mcp.aigateway.cequence.ai/mcp
operations: [list_third_party_apps, get_third_party_app, list_api_specs, get_api_spec, list_mcp_specs, get_mcp_spec, list_pools, list_teams, create_mcp_server, list_skills, create_agent_persona, recommend_dlp_policy, attach_dlp_policy, get_agent_persona]
generated: '2026-09-05'
method: generated
source: https://docs.aigateway.cequence.ai/docs/remote-mcp-servers/cequence-ai-gateway
grounding: >-
  Every tool name below is transcribed from the provider's published tools reference. Cequence
  publishes no OpenAPI, and live tools/list is auth-gated, so no input schema is asserted here —
  read each tool's real arguments from the server after you connect.
---

# Onboard a least-privilege agent on Cequence AI Gateway

Use this when someone asks for "an agent that can do X in system Y" and it has to be governed
rather than handed a shared API key.

## Before you start

Connect to the Cequence AI Gateway MCP server. It is provisioned automatically for every tenant
— you do not create it from a vendor URL. Copy the per-tenant URL from the server's **MCP
details page** in the portal; do not hardcode `https://mcp.aigateway.cequence.ai/mcp`.

Authentication is OAuth 2.1 authorization-code with PKCE and dynamic client registration
(`/register`, `/authorize`, `/token`), fronted by RFC 8414 and RFC 9728 discovery documents. You
complete a browser sign-in on first connect. **Your MCP permissions equal your portal role.**
Read tools work for any tenant member; every `create_*` / `update_*` / `attach_*` tool needs an
operator role (PlatformOperator or TenantUser) and otherwise returns an authorization error.

## Steps

1. **Find the source.** Call `list_third_party_apps` to browse the Cequence-curated catalog,
   then `get_third_party_app` on the candidate to see its endpoints (for an API) or its tools
   (for a remote MCP server). If the target is the tenant's own upload, use `list_api_specs` /
   `get_api_spec` or `list_mcp_specs` / `get_mcp_spec` instead.

2. **Pick the grants before you create anything.** Name the specific endpoints or tools the
   agent needs. Never "expose everything" — least privilege is the point of the gateway, and a
   persona that starts wide is not narrowed later.

3. **Decide where it runs.** `list_pools` shows private deployment pools; omit a pool for the
   Cequence cloud. `list_teams` shows the SSO-mapped groups that will gate access.

4. **Create the MCP server.** Call `create_mcp_server` with the chosen spec, the selected
   endpoints or tools, the inbound auth method, the upstream auth method, and the deployment
   target. **Call it with `dryRun: true` first.** The dry run validates every referenced tool,
   API and skill and surfaces warnings without creating anything. Show the caller the result and
   get approval before the real call.

5. **Compose the persona.** `list_skills` to find reusable runbooks, then `create_agent_persona`
   with the mix of MCP tools, REST API operations and skills, the inbound auth method, the rate
   limit, and the teams that may use it. Again: `dryRun: true` first.

6. **Protect the data.** Call `recommend_dlp_policy` for the persona — the recommendation is
   deterministic, derived from the persona's tools, endpoints and job description — then
   `attach_dlp_policy` so the policy follows the agent across every tool, API and model it uses.
   `list_sdp_categories` and `list_dlp_policies` show what is available and what is already on.

7. **Hand back the connect URL.** `get_agent_persona` returns the persona including its connect
   URL. That URL, not the admin endpoint, is what the requester puts in their AI client.

## Rules that will bite you

- **There is no undo.** No `delete_agent_persona`, no `delete_mcp_server`, no `detach_dlp_policy`
  on the MCP surface, and no documented restore window. `dryRun` is the only pre-commit safety
  net Cequence ships. Rehearse; do not experiment.
- **There is no idempotency key.** Retrying a `create_*` call that may have succeeded can create
  a second persona or server. If a call times out, call the matching `list_*` tool and check
  before retrying.
- **Rate limits are per tool, and they default by HTTP method** — 1,000/hour for GET/HEAD/OPTIONS,
  100/hour for POST/PUT/PATCH, 10/hour for DELETE, in a *rolling* window. Set the persona's limit
  deliberately: interactive users and CI/CD pipelines need different numbers, and limits are
  configured per MCP server, so two use cases usually mean two servers.
- **A 429 carries no budget headers.** Cequence documents no `RateLimit-*` or `Retry-After`
  header, so you cannot pace ahead of the limit — back off on the 429 itself.
- **Responses may arrive redacted.** A DLP policy in `redact` mode masks fields in flight. Plan
  for a masked value, not an absent one.
- Errors map to the enforcement pipeline: 404 routing, 401 `auth_denied`, 403 `authz_denied`,
  429 `rate_limit_exceeded`, 502/503 `upstream_error` / `circuit_breaker_open`. See
  `errors/cequence-problem-types.yml`.
