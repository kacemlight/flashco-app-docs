# 7. The Integration Layer (MCP)

Agents are only as useful as the systems they can act in. FlashCo's
integration layer is built on the **Model Context Protocol (MCP)** — used
in production across two transports, with two self-hosted tool servers and
a set of design rules that keep runtime-discovered tools from becoming a
hallucination factory.

## Two transports, one abstraction

| Transport | Used for | Mechanics |
|---|---|---|
| **stdio (child process)** | Community MCP servers (GitHub, Notion, Slack, SharePoint, Google Drive) | The runtime spawns the server as a subprocess per company with decrypted credentials in its environment; JSON-RPC over stdio; tool discovery via `tools/list` at startup |
| **Streamable HTTP** | Self-hosted servers (CMS publishing, document suite) and credential-based remote servers | JSON-RPC 2.0 over HTTP POST with SSE response parsing, session IDs, Bearer auth per request |

Above the transport, everything is uniform: one manager owns connections,
discovers tools, namespaces them, and dispatches calls by prefix.

## Stateless self-hosted servers

The self-hosted servers follow one strict rule: **the server holds zero
credentials.** Every request carries the tenant's token
(`Authorization: Bearer …`) plus tenant context headers (e.g. the target
CMS author instance URL). Consequences:

- **One deployment serves every tenant** — no per-tenant instances, no
  server-side token vault to breach.
- **Scale-to-zero** — the servers are tiny, stateless, and sleep when idle;
  the marginal cost of an integration approaches zero.
- **No confused deputy** — the server can never be tricked into using
  *its own* authority on a tenant's behalf, because it has none. The
  gateway is the sole credential holder, and it forwards per-request.

## Runtime discovery and the anti-hallucination rule

Tool surfaces come in two kinds, and conflating them is a classic failure:

- **Statically known** — integrations whose tool lists are fixed and
  registered (either documented community servers or servers we build).
  These names may appear in generated documents.
- **Runtime-discovered** — tool names that only exist after a live
  `tools/list` handshake.

The rule: **never enumerate runtime-discovered tool names at
design time.** An LLM asked to write documentation for tools it hasn't
seen will confidently invent plausible-but-wrong names
(`createDocument` vs. `create-document`), and agents will then call
functions that don't exist.

Instead:

1. At blueprint time, generated docs describe runtime-discovered
   integrations by **capability** ("create and publish content fragments"),
   sourced from the integration registry's human-written capability lists.
2. At runtime, the **live tool catalog** — exact names, descriptions,
   schemas from discovery — is injected into every agent's system prompt
   under an explicit "ready to use" section, together with per-integration
   connection status (so agents can distinguish *not configured* from
   *configured but currently broken*).
3. Every generated TOOLS.md carries a verbatim paragraph instructing agents
   to consult the live catalog before calling anything.

Documents teach intent; the runtime is the authority on names.

## Namespacing and routing

Discovered tools are prefixed with their integration ID
(`github_create_issue`, `aem_publishContentFragments`). Prefix resolution
is longest-first so multi-segment IDs resolve correctly. The prefix does
triple duty: routing (which connection handles the call), filtering (the
three-layer allowlist keys on integration ID —
[doc 5](./05-security-model.md)), and legibility (agents and audit logs
see at a glance which system a tool touches).

## Resilience

- **OAuth token refresh** happens in the runtime with an expiry buffer;
  refreshed tokens are re-encrypted and persisted. A 401 during the MCP
  handshake triggers a forced refresh and one retry.
- **Auto-reconnect on failure**: when an MCP call errors, the runtime
  attempts to reconnect that integration and retries the call once —
  absorbing the most common transient failures (expired sessions, server
  cold starts) without surfacing them to the agent.
- **Hot reinitialization**: integrations can be re-initialized for a
  running company (after credential rotation) without restarting agents;
  agent prompt caches are invalidated so the new tool catalog is picked up.

## Choosing MCP (and what about A2A)

MCP fits FlashCo's shape: **composed orchestration** — one runtime
composing many granular capabilities, with full visibility into each call
(which the security guard requires; an opaque delegated black box could
not be policy-checked per call).

Agent-to-Agent protocols (A2A) solve a different problem — delegating
whole tasks to external, opaque agents across organizations. FlashCo is a
closed orchestration domain today, so A2A is deliberately absent. The
natural future move is the reverse direction: exposing each FlashCo
company *as* an A2A server (an agent card at a well-known URL), so
external agents can hire a company. Tracked on the
[roadmap](./09-roadmap.md).
