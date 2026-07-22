# 11. Implementing Three-Tier Memory

*Implementation deep-dive. Design context: [doc 4 — Memory & Context
Engineering](./04-memory-and-context.md). Snippets from the production
runtime, lightly trimmed.*

The memory system's job: let an agent work for days inside a context
window built for minutes. The implementation is three data structures, two
budgets, and a handful of carefully boring rules.

## Budgets first

```ts
const WORKING_MEMORY_TOKEN_BUDGET = 12_000;  // tier 1: verbatim recent turns
const SUMMARY_MEMORY_TOKEN_BUDGET = 2_000;   // tier 2: compressed past
const MAX_EPISODIC_TOKENS = 1_000;           // tier 3: structured facts

/**
 * Older turns get observation masking (tool_result → short stub).
 * JetBrains Research found 10 is optimal; we use 6 for tighter budget.
 */
const RECENT_UNMASKED_TURNS = 6;
```

Token counting is deliberately approximate — `Math.ceil(text.length / 4)`
— because memory decisions need to be *cheap and directionally right*, not
exact. Calling a tokenizer on every message on every turn would cost more
than the imprecision does.

## Observation masking

The single highest-leverage function in the file. Old tool results are
replaced with stubs at *recall* time (storage keeps the original until
summarization claims it):

```ts
private maskObservations(msg: Message): Message {
  if (typeof msg.content === "string") return msg;

  const blocks = msg.content.map((block) => {
    if (block.type !== "tool_result" || !block.content) return block;

    // Only mask substantial results (>200 chars)
    if (block.content.length <= 200) return block;

    // Keep a very short snippet for context
    const snippet = block.content.slice(0, 100).replace(/\n/g, " ");
    return {
      ...block,
      content: `${snippet}... [observation masked — see episodic memory for key facts]`,
    };
  });
  return { role: msg.role, content: blocks };
}
```

Three details that matter:

1. **Small results survive.** A 150-char tool result ("file written OK")
   is cheaper to keep than to stub — masking it would *lose* information
   per token.
2. **The stub keeps a 100-char snippet.** The agent retains the *shape* of
   what it saw ("Market Report 2025: The eco segment gr...") — enough to
   know whether it needs to re-fetch, never enough to flood context.
3. **The stub says where the facts went.** "See episodic memory" is a real
   pointer — the important content of that observation was extracted into
   tier 3 when the tool ran.

## Episodic facts — memory that survives forgetting

Facts are extracted *at tool-call time*, not by a later analysis pass:

```ts
export interface EpisodicFact {
  type: "decision" | "deliverable" | "delegation"
      | "task_completed" | "approval" | "key_info";
  timestamp: number;
  summary: string;   // one line: "Wrote file: market-overview.md"
}
```

When the agent calls `write_deliverable`, `complete_task`, or
`delegate_to_agent`, the memory records a one-liner. The list is capped;
when it overflows, oldest facts drop first (but never below a floor). The
result: an agent can lose every conversational detail to summarization and
still answer "what have you produced so far?" correctly — because *what it
did* is stored separately from *how the conversation went*.

## Summarization with safe cut points

When working memory exceeds budget, the oldest half is compressed by a
cheap model. The subtle part is where you're allowed to cut:

```ts
applySummarization(newSummary: string): void {
  const keepCount = Math.max(
    MIN_WORKING_MESSAGES,
    Math.min(8, Math.floor(this.workingMemory.length / 2))
  );
  let cut = this.workingMemory.length - keepCount;

  // A cut must land on a plain user message — never between an
  // assistant tool_use and the user tool_result that answers it.
  while (cut < this.workingMemory.length) {
    const msg = this.workingMemory[cut];
    if (msg.role === "assistant") { cut++; continue; }
    if (msg.role === "user" &&
        Array.isArray(msg.content) &&
        msg.content.some((b) => b.type === "tool_result")) { cut++; continue; }
    break;
  }

  // Merge with the existing summary; if the merge overflows its budget,
  // keep only the new summary — more recent = more relevant.
  if (this.summaryMemory) {
    const merged = `${this.summaryMemory}\n\n--- More recent ---\n\n${newSummary}`;
    this.summaryMemory = estimateTokens(merged) > SUMMARY_MEMORY_TOKEN_BUDGET
      ? newSummary
      : merged;
  } else {
    this.summaryMemory = newSummary;
  }
  this.workingMemory = this.workingMemory.slice(cut);
}
```

Cutting between a `tool_use` and its `tool_result` produces an orphaned
result — which provider APIs reject outright. The safe-cut walk is two
lines of logic that prevent an entire category of 400 errors.

**The failure path is also designed:** if the summarization model call
itself fails, an emergency trim hard-drops the oldest messages. Degraded
recall beats a crashed agent — memory management must never be the thing
that kills the worker.

## Prompt assembly — how the tiers compose

```ts
buildMessages(): Message[] {
  const messages: Message[] = [];

  // Tier 3 + Tier 2 as a context preamble, then a synthetic ack,
  // so the conversation continues naturally from compressed history.
  if (this.episodicFacts.length > 0 || this.summaryMemory) {
    messages.push({ role: "user", content: contextPreamble });
    messages.push({ role: "assistant", content: "Understood — continuing." });
  }

  // Tier 1, with observation masking outside the recent window
  const maskBefore = this.workingMemory.length - RECENT_UNMASKED_TURNS * 2;
  this.workingMemory.forEach((msg, i) => {
    messages.push(i < maskBefore ? this.maskObservations(msg) : msg);
  });
  return messages;
}
```

The synthetic assistant acknowledgment is a small trick with a real
purpose: providers require alternating roles, and a context preamble
followed directly by the next user message would break alternation. The
ack keeps the conversation shape legal while costing four tokens.

## Serialization — memory that survives restarts

The entire structure — all three tiers plus counters — serializes to JSON
on graceful shutdown and restores on boot. This is what lets the runtime
redeploy mid-mission without agents losing their minds. (Crash-safe
*mid-turn* persistence is a roadmap item — see
[doc 9](./09-roadmap.md).)

## The numbers that justify the design

With all three tiers active, long-running agents hold steady around the
working budget (~12k tokens of history per call) regardless of mission
length — versus unbounded growth without it. Combined with provider-side
prompt caching of the static system prompt
([doc 4](./04-memory-and-context.md)), the marginal context cost of turn
*N* is effectively flat in *N*. That flatness is the entire game: it is
what makes multi-day companies economically possible.
