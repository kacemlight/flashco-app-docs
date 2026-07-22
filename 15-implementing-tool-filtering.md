# 15. Implementing Per-Agent Tool Filtering

*Implementation deep-dive. Design context: [doc 5 — Security
Model](./05-security-model.md) (least privilege) and
[doc 2 — Blueprint Notation](./02-blueprint-notation.md) (capability
gating). Snippets from the production runtime, lightly trimmed.*

Least privilege for agents sounds like policy work; in practice it is one
pure function in the runtime and one carefully-designed sentinel in the
data model. This doc covers both — including the allowlist bug the
sentinel exists to fix.

## The three-layer filter

Computed once per execution, before the model ever sees a tool menu:

```ts
function filterMcpToolsForAgent(
  allTools: ToolDefinition[],
  definition: AgentDefinition
): ToolDefinition[] {
  const agentRole = definition.role.toLowerCase();
  const allowedIntegrations = definition.allowed_integrations ?? [];
  const hasIntegrationAllowlist = allowedIntegrations.length > 0;
  const allowedTools = new Set(definition.allowed_tools ?? []);
  const hasToolAllowlist = allowedTools.size > 0;

  return allTools.filter((tool) => {
    const integrationId = findIntegrationIdForToolName(tool.name);
    if (!integrationId) return true;   // unknown prefix → pass through

    // Layer 1: per-agent integration allowlist (from the blueprint)
    if (hasIntegrationAllowlist && !allowedIntegrations.includes(integrationId)) {
      return false;
    }

    // Layer 2: per-agent fine-grained tool allowlist. Only enforced for
    // integrations with a statically-known tool list — runtime-discovered
    // tools can't be individually gated at blueprint time and are governed
    // by layer 1 only.
    const def = INTEGRATION_REGISTRY[integrationId];
    const hasStaticTools = (def?.tools.length ?? 0) > 0;
    if (hasToolAllowlist && hasStaticTools && !allowedTools.has(tool.name)) {
      return false;
    }

    // Layer 3: per-integration role allowlist from the registry
    const rolesAllowed = def?.allowed_roles ?? [];
    if (rolesAllowed.length > 0 && !rolesAllowed.includes(agentRole)) {
      return false;
    }
    return true;
  });
}
```

### The Layer-2 subtlety that prevents a nasty bug

Tool names for some integrations only exist after runtime discovery
([doc 7](./07-integration-layer-mcp.md)) — so blueprint-time checkboxes
can never include them. If Layer 2 applied to *all* tools, unchecking one
GitHub tool would materialize a tool allowlist that silently hides **every
runtime-discovered tool** (their names aren't in the list). Hence the
`hasStaticTools` guard: fine-grained gating where names are known,
integration-level gating where they aren't. A one-line guard against an
entire class of "why did AEM disappear?" support tickets.

### Prefix resolution, longest-first

```ts
function findIntegrationIdForToolName(toolName: string): string | null {
  const ids = Object.keys(INTEGRATION_REGISTRY)
    .sort((a, b) => b.length - a.length);   // "google_docs" before "google"
  for (const id of ids) {
    if (toolName === id || toolName.startsWith(`${id}_`)) return id;
  }
  return null;
}
```

Namespaced tool names (`github_create_issue`) make the integration ID
recoverable from the name alone — which is what lets routing, filtering,
and audit logs all key off the same prefix.

## The allowlist data model — and the deny-all sentinel

The persistence semantics look trivial and hide a trap:

```
allowed_integrations: []            → ALL integrations allowed (the default)
allowed_integrations: ["github"]    → only GitHub
allowed_integrations: ["__none__"]  → NOTHING (deny-all sentinel)
```

The trap: with "empty means all", a UI where the user unchecks every
integration would produce `[]` — which silently means *all again*. The
round trip "restrict everything" → "everything allowed" is exactly the
kind of security UI bug that passes every casual test. The fix is a
non-empty sentinel that matches no real integration:

```ts
function toggleIntegration(id: string) {
  // Materialize the allowlist on first deselection — [] means "all",
  // so we can't just remove one item from it. The ["__none__"] deny-all
  // sentinel materializes to an empty set.
  const current = usingAllIntegrations
    ? [...availableIntegrations]
    : allowed.filter((x) => x !== "__none__");

  const next = active ? current.filter((x) => x !== id) : [...current, id];

  onChange(index, {
    ...agent,
    allowed_integrations:
      next.length === availableIntegrations.length
        ? []                    // everything re-checked → collapse to "all"
        : next.length === 0
        ? ["__none__"]          // everything unchecked → deny-all, NOT "all"
        : next,
  });
}
```

The runtime needs no special case: `["__none__"]` is a non-empty allowlist
that matches nothing, so Layer 1 filters everything out naturally. The
sentinel lives purely in the data; the filter stays a pure function.

## Keeping two allowlists consistent

The per-tool checkboxes (Layer 2) must stay coherent when integrations
toggle (Layer 1). Deactivating an integration drops its tools from the
checked set; **re-activating must add them back** — a materialized tool
allowlist that doesn't include the newly-activated integration's tools
would hide them at runtime:

```ts
let nextTools = agent.allowed_tools ?? [];
if (nextTools.length > 0) {
  const toolsOfId = new Set(staticToolsOf(id));
  nextTools = active
    ? nextTools.filter((t) => !toolsOfId.has(t))     // deactivate: drop
    : [...nextTools, ...staticToolsOf(id)];          // reactivate: re-check all

  // Collapse back to "all tools" when every static tool is checked
  const fullSet = allActiveStaticTools(nextIntegrations);
  if (fullSet.length > 0 && fullSet.every((t) => nextTools.includes(t))) {
    nextTools = [];
  }
}
```

## One source of truth, three projections

The same two lists per agent drive:

1. **Runtime enforcement** — the filter above, computed fresh each
   execution;
2. **Documentation** — the TOOLS.md agent×integration matrix is generated
   from these lists and marked *authoritative* in the generation prompt
   ([doc 2](./02-blueprint-notation.md));
3. **Consent** — the launch flow shows investors exactly which agent may
   touch which system before they fund it.

When enforcement, documentation, and consent are projections of one data
structure, they cannot disagree — which is the actual definition of a
capability *contract* rather than a capability *hope*.
