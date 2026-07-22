# 10. Implementing the Agent Execution Loop

*Implementation deep-dive. Design context: [doc 3 — Orchestration
Patterns](./03-orchestration-patterns.md). Snippets are taken from the
production runtime (TypeScript, Node.js), lightly trimmed for reading.*

The execution loop is the heart of any agent runtime: build context, call
the model, execute tool calls, feed results back, repeat until done — or
until one of several safety layers says stop. The interesting engineering
is entirely in the "until" clauses.

## The loop skeleton

```ts
const MAX_TURNS = 40;
const TASK_WARN_MS = 6 * 60_000;   // 6 min — inject "wrap up" nudge
const TASK_TIMEOUT_MS = 8 * 60_000; // 8 min — hard stop

async execute(agentContext: AgentContext, userMessage: string) {
  agentContext.status = "working";
  const startedAt = Date.now();
  let warnInjected = false;
  let taskCompleted = false;
  let turns = 0;

  // Compute this agent's tool menu: native tools for its role, plus
  // MCP tools filtered through the three-layer allowlist (doc 15).
  const nativeTools = getToolsForRole(agentContext.definition.role);
  const mcpTools = filterMcpToolsForAgent(
    this.mcpManager?.getAllTools() ?? [],
    agentContext.definition
  );
  const tools = [...nativeTools, ...mcpTools];

  agentContext.addMessage({ role: "user", content: userMessage });

  while (turns < MAX_TURNS) {
    turns++;

    // ── Wall-clock guards ──────────────────────────────────────────
    const elapsed = Date.now() - startedAt;
    if (agentContext.currentTask && elapsed > TASK_TIMEOUT_MS) {
      finalResponse = "Task timed out. Partial work may have been saved.";
      break;
    }
    if (agentContext.currentTask && !warnInjected && elapsed > TASK_WARN_MS) {
      warnInjected = true;
      agentContext.addMessage({
        role: "user",
        content:
          "[System]: You have been working for over 6 minutes. Wrap up now — " +
          "call `complete_task` with whatever you have. If a deliverable is " +
          "partially done, write it as-is and note what remains.",
      });
    }

    // ── Model call ────────────────────────────────────────────────
    const systemPrompt = await agentContext.buildSystemPrompt(
      mcpTools, integrationStatuses      // live tool catalog + status (doc 7)
    );
    const messages = this.sanitizeMessages(agentContext.buildMessagesForLLM());

    const response = await this.sendWithQueue(agentContext.agentId, {
      model: agentContext.definition.model,
      system: systemPrompt,
      messages,
      max_tokens: maxTokens,
      tools,
    });

    // ── No tool calls → the text IS the final answer ──────────────
    const toolCalls = extractToolCalls(response);
    if (toolCalls.length === 0) {
      finalResponse = textContent;
      break;
    }

    // ── Execute tools, guard every result, loop ───────────────────
    const toolResults = [];
    for (const toolCall of toolCalls) {
      let result = this.mcpManager?.isMcpTool(toolCall.name)
        ? await this.callMcpWithPolicy(toolCall)     // Rule of Two (doc 12)
        : await executeTool(toolCall, toolCtx);
      result = this.guardResult(toolCall, result);   // redaction, caps, wrapping
      toolResults.push(result);
      if (toolCall.name === "complete_task") taskCompleted = true;
    }
    agentContext.addMessage({
      role: "user",
      content: toolResults.map(toToolResultBlock),
    });

    if (agentContext.shouldSummarize) {
      await this.performSummarization(agentContext);  // doc 11
    }
    if (taskCompleted && response.stop_reason !== "tool_use") break;
  }
}
```

## The design decisions inside

### 1. Soft timeout before hard timeout

The two-stage timeout is the loop's most human-shaped feature. At six
minutes, the agent gets a *message*, not a kill: "wrap up, ship partial
work." Only at eight minutes does the runtime force-stop. In practice the
nudge works — models are good at concluding when told to — and partial
deliverables beat empty hands. The principle from
[doc 3](./03-orchestration-patterns.md): **terminating early with partial
results beats running forever.**

### 2. Tool errors are messages, never crashes

Every failure path — unknown tool, MCP disconnect, execution exception —
returns a `tool_result` with `is_error: true` and a descriptive message.
The loop never throws on a tool failure; the *model* gets to react to it
(retry, work around, escalate). A runtime that crashes on bad tool calls
trains you to fear autonomy; one that converts failures into context lets
the agent be resilient.

### 3. MCP auto-reconnect, one retry

Transient MCP failures (expired sessions, cold-started servers) are the
most common integration failure. The loop absorbs them:

```ts
result = await this.mcpManager.callTool(toolCall.name, toolCall.id, input);
if (result?.is_error) {
  const integrationId = findIntegrationIdForToolName(toolCall.name);
  const status = await this.mcpManager.reconnectIntegration(integrationId);
  if (status?.state === "connected") {
    agentContext.invalidatePromptCache();   // tool catalog may have changed
    result = await this.mcpManager.callTool(toolCall.name, toolCall.id, input);
  }
}
```

One reconnect, one retry, then the error goes to the model. No exponential
backoff loops inside an LLM turn — latency there is billed thinking time.

### 4. `stop_reason` handling: the truncation case

When a response ends with `stop_reason === "max_tokens"`, the model was cut
off mid-generation — usually mid-deliverable. The loop detects this and
tells the agent to split the work ("write the deliverable in parts"),
rather than silently persisting a truncated artifact.

### 5. Serialized model access

`sendWithQueue` routes every model call through a per-company request
queue (one in flight, minimum gap between calls, priority classes:
investor messages > task execution > heartbeats). Rate limits are a
*shared* resource; without a governor, one busy agent starves the team.

### 6. History sanitization before every call

Provider APIs are strict about message shape: every `tool_result` must
reference a `tool_use` in the immediately preceding assistant message.
After summarization and masking, history can violate this. A sanitize pass
before each call drops orphaned results and repairs alternation —
mechanical, boring, and the difference between a runtime that works for
eight hours and one that 400s at hour three.

## What the loop deliberately does not do

- **No self-extension**: an agent cannot raise its own turn cap or
  timeout. Limits live outside the model's reach.
- **No dynamic tool acquisition**: the tool menu is computed once per
  execution from the allowlists. An agent cannot talk itself into new
  capabilities mid-turn.
- **No hidden retries of the model call itself** beyond rate-limit
  handling — a bad completion is context for the next turn, not a
  do-over. Determinism of *record* matters more than polish of output:
  every turn that happened is in the trajectory
  ([doc 6](./06-evaluation-methodology.md)).
