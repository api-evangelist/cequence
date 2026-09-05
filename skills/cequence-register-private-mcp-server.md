---
name: Register a private-network MCP server with Cequence AI Gateway
description: Use the first-party CLI to introspect an MCP server the control plane cannot reach, then register it with its tools intact instead of with zero tools.
api: cli/cequence-cli.yml
package: "@cequenceai/mcp-cli"
operations: [introspect, create_mcp_server, list_mcp_specs, get_mcp_spec, list_mcp_servers, list_tools_for_mcp_server]
generated: '2026-09-05'
method: generated
source: https://www.npmjs.com/package/@cequenceai/mcp-cli
grounding: >-
  Commands and flags are transcribed verbatim from the published @cequenceai/mcp-cli README
  (version 2.0.1, 2026-08-19). MCP tool names come from the provider's tools reference.
---

# Register a private-network MCP server with Cequence AI Gateway

## The problem this solves

When an MCP server sits on a private or internal network, the Cequence control plane cannot
reach it to auto-discover its tools — so it registers with **0 tools** and is useless to an
agent. `introspect` runs from a machine that *can* reach the server, does the MCP handshake,
lists every tool, and writes a bundle you register.

## Steps

1. **Run introspect from inside the network.**

   ```bash
   npx @cequenceai/mcp-cli@latest introspect \
     --url "https://internal-mcp.corp.local/mcp" \
     --name "Internal Widgets" --primary-category "Developer Tools"
   ```

   Auth variants, all documented:
   - bearer: `--auth bearer --token "$TOKEN"`
   - API key in a custom header: `--auth api-key --api-key "$KEY" --header-name "X-API-Key"`
   - basic: `--auth basic --basic-user … --basic-pass …`
   - OAuth (opens a browser for the authorization-code flow): `--auth oauth --scope "read"`,
     local redirect port `--callback-port` (default 8765)
   - SSE instead of Streamable HTTP: `--transport sse`
   - extra headers: `--header "X-Tenant: acme"`, repeatable

2. **Take the bundle.** By default you get one archive, `<out-dir>/<slug>.tar.gz`, containing
   `create-spec.json` plus informational `tools.json` and `manifest.json`. Use `--format dir` to
   write the three files loose for inspection or diffing. If the working directory is not
   writable — common in locked-down environments — point `--out` somewhere you can write.

3. **Register it.** Either upload the `.tar.gz` in the AI Gateway UI (add a remote MCP server →
   the CLI option), or skip the UI entirely:

   ```bash
   npx @cequenceai/mcp-cli@latest introspect \
     --url "https://internal-mcp.corp.local/mcp" --name "Internal Widgets" \
     --push --api-url "https://api.your-gateway.example" \
     --tenant "my-company" --api-token "$CP_BEARER"
   ```

   `--push` posts to `POST /api/v2/mcp-specs` on the control plane. Env fallbacks:
   `CEQUENCE_API_URL`, `CEQUENCE_TENANT_ID`, `CEQUENCE_API_TOKEN`.

4. **Refresh, don't duplicate.** To update an existing catalog spec's tool list in place, add
   `--spec-id "<id>"` to the `--push` call. Re-introspecting without `--spec-id` creates a new
   spec — and nothing in this pipeline is idempotent.

5. **Build the server from it.** Back on the MCP surface, `list_mcp_specs` / `get_mcp_spec`
   confirm the spec and its tool inventory, then `create_mcp_server` (with `dryRun: true` first)
   builds the gateway server from it. Verify with `list_mcp_servers` and
   `list_tools_for_mcp_server`.

## The caveat the docs put in bold, and you should too

`introspect` removes the **tool-discovery** blocker only. For a **cloud-hosted** Cequence
gateway the data plane must still reach the internal endpoint at runtime — a network route or an
IP allowlist. The Cequence-hosted SaaS gateway egresses from `34.132.39.216` and `34.170.84.5`;
allowlist **both**, because traffic can leave from either and permitting one produces
intermittent failures that are hard to diagnose. For a private or hybrid deployment the data
plane already runs inside your network and runtime reachability exists.

## Credential hygiene

`--api-token` authenticates to the **Cequence** control plane. `--token` authenticates to the
**remote MCP server**. They are different credentials and swapping them produces a confusing
401 from the wrong side.
