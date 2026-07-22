# 13. Implementing Schema-Forced Structured Output

*Implementation deep-dive. Design context: docs
[2](./02-blueprint-notation.md) and [6](./06-evaluation-methodology.md).
Snippets from production (TypeScript and Python), lightly trimmed.*

Anywhere an LLM produces data the *system* depends on — task plans,
extracted fields, evaluation scores — free-text-then-parse is a reliability
tax you pay forever. The fix is structural: **force the model through a
tool call whose input the API validates against a JSON schema.** Malformed
output becomes impossible rather than merely handled.

## Before: the parser that admits defeat

The original task generator asked for "ONLY a raw JSON array" and then
defended against everything models actually do:

```ts
export function parseTasksJson(raw: string): GeneratedTask[] {
  let text = raw.trim();

  // 1. Models wrap output in markdown fences despite instructions
  const fenceMatch = text.match(/```(?:json)?\s*\n([\s\S]*?)\n?```/i);
  if (fenceMatch) text = fenceMatch[1].trim();

  try {
    parsed = JSON.parse(text);
  } catch {
    // 2. Models add preambles — slice from first '[' to last ']'
    const first = text.indexOf("[");
    const last = text.lastIndexOf("]");
    parsed = JSON.parse(text.slice(first, last + 1));   // 3. pray
  }
  // ... then per-field validation and coercion
}
```

Every branch of that function is documentation of an instruction that
failed in production. It still exists — as a *fallback* — but nothing
depends on it anymore.

## After: the schema is the contract

```ts
export const EMIT_TASKS_TOOL = {
  name: "emit_tasks",
  description: "Return the generated task blueprint",
  input_schema: {
    type: "object",
    properties: {
      tasks: {
        type: "array",
        minItems: 1,
        items: {
          type: "object",
          properties: {
            title:         { type: "string" },
            description:   { type: "string" },
            assigned_role: { type: "string" },
            priority:      { type: "integer", minimum: 1, maximum: 5,
                             description: "1=highest/run first, 5=lowest/defer" },
            fixed:         { type: "boolean" },
            depends_on: {
              type: "array", items: { type: "integer", minimum: 0 },
              description:
                "Zero-based indices of EARLIER tasks that must finish first. " +
                "Backwards references only; empty for independent tasks.",
            },
          },
          required: ["title", "description", "assigned_role", "priority", "fixed"],
        },
      },
    },
    required: ["tasks"],
  },
} as const;

export async function generateTasksBlueprint(ctx, instructions?, apiKey?) {
  const data = await callAnthropic(resolveApiKey(apiKey), {
    model: MODEL_MAP.tasks_blueprint,
    max_tokens: 2048,
    messages: [{ role: "user", content: buildTasksPrompt(ctx) }],
    tools: [EMIT_TASKS_TOOL],
    tool_choice: { type: "tool", name: "emit_tasks" },   // ← the force
  });

  const toolUse = data.content?.find((b) => b.type === "tool_use");
  if (toolUse?.input && Array.isArray((toolUse.input as any).tasks)) {
    return validateTaskArray((toolUse.input as any).tasks, JSON.stringify(toolUse.input));
  }
  // Legacy fallback for the rare no-tool_use response
  return parseTasksJson(data.content?.find((b) => b.type === "text")?.text ?? "");
}
```

`tool_choice: { type: "tool", name: "emit_tasks" }` is the load-bearing
line: the model *cannot* answer with prose, fences, or a preamble — the
only legal response shape is a validated `emit_tasks` invocation.

## Schema constraints as semantics, not just types

Two constraints in that schema do real orchestration work:

- `priority: { minimum: 1, maximum: 5 }` plus its description encodes the
  scheduler's semantics *in the schema* — the model can't emit priority 0
  or 99, and the description teaches it which direction is "first".
- `depends_on` items with `minimum: 0` plus post-validation clipping of
  forward references makes the task graph **acyclic by construction**:

```ts
// depends_on: silently drop bad entries; clip any forward/self reference —
// dependencies point backwards only, so cycles cannot exist.
const clean = obj.depends_on
  .map((d) => (typeof d === "number" ? d : Number(d)))
  .filter((d): d is number => Number.isInteger(d) && d >= 0 && d < i);
```

Schema for shape, cheap deterministic post-validation for the invariants
schemas can't express. Both together, and the dependency-wave dispatcher
([doc 3](./03-orchestration-patterns.md)) never needs a cycle detector.

## The same pattern in Python — the evaluation judge

The retrospective judge ([doc 14](./14-implementing-the-llm-judge.md))
builds its schema *programmatically* from the dimension table, so scoring
dimensions and their schema can never drift apart:

```python
DIMENSION_WEIGHTS = {
    "deliverable_quality": 0.35,
    "task_completion":     0.25,
    "collaboration":       0.15,
    "efficiency":          0.15,
    "communication":       0.10,
}

EVALUATION_TOOL = {
    "name": "submit_evaluation",
    "input_schema": {
        "type": "object",
        "properties": {
            **{dim: {
                "type": "object",
                "properties": {
                    "score":     {"type": "integer", "minimum": 0, "maximum": 10},
                    "reasoning": {"type": "string"},
                },
                "required": ["score", "reasoning"],
            } for dim in DIMENSION_WEIGHTS},
            "strengths":  {"type": "array", "items": {"type": "string"}},
            "weaknesses": {"type": "array", "items": {"type": "string"}},
            "summary":    {"type": "string"},
        },
        "required": [*DIMENSION_WEIGHTS.keys(), "strengths", "weaknesses", "summary"],
    },
}

response = client.messages.create(
    model=JUDGE_MODEL,
    max_tokens=2048,
    messages=[{"role": "user", "content": prompt}],
    tools=[EVALUATION_TOOL],
    tool_choice={"type": "tool", "name": "submit_evaluation"},
)
evaluation = next(b for b in response.content if b.type == "tool_use").input
# → guaranteed: every dimension present, every score an int in [0,10]
```

Note `"reasoning"` is *required* per dimension: the schema forces the
judge to justify every number, which is what makes scores debuggable and
calibratable later.

## Where to draw the line

Not everything should be schema-forced. The rule used throughout the
platform:

| Output | Treatment | Why |
|---|---|---|
| Task graphs, extracted fields, judge scores | **Schema-forced tool call** | The system executes on this data; malformed = broken |
| Workspace documents (SOUL.md, TOOLS.md…) | Free generation + prompt quality bars | The *reader* is a model; prose fidelity matters more than shape |
| Agent conversation | Free | That's the product |

Structured output where machines depend on it; language where language is
the point. The same split as the blueprint notation itself
([doc 2](./02-blueprint-notation.md)): prose where humans decide, schemas
where machines depend.
