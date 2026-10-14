# 18 — Agent Skills & the Blueprint Experience Loop

> How FlashCo blueprints carry portable expertise (Skills) and accumulate
> their own memory across runs (Experience) — the two mechanisms that make
> a blueprint more than a static template.

The previous documents describe a blueprint as a *declaration*: six files
and a topology that render into a company. This document covers the two
mechanisms that make the declaration **alive**: Skills, which let a
designer package specialist know-how per agent, and the Experience loop,
which lets every company teach the blueprint something for the next one.

---

## Part 1 — Agent Skills (SKILL.md)

### The idea

A persona tells an agent who it is. A **skill** tells it how to work:
a checklist, a convention, a domain playbook. FlashCo adopts the open
[Agent Skills format](https://skills.sh) — a `SKILL.md` file with YAML
frontmatter (`name`, `description`) and markdown instructions — so
expertise written anywhere in the ecosystem can be attached to a FlashCo
agent, and skills written for FlashCo work elsewhere.

### Design-time: attaching skills

In the blueprint wizard, each agent card carries a Skills section with
two paths:

- **Import** — paste a skills.sh URL, a GitHub URL, or an
  `owner/repo/skill-name` shorthand. A server-side endpoint resolves the
  reference to the raw `SKILL.md` on GitHub (where all skills.sh skills
  live), parses the frontmatter, and returns it for review. Fetches are
  restricted to skills.sh/GitHub hosts with size caps; the content is
  **data the designer reviews and attaches**, never something fetched or
  executed at runtime.
- **Write inline** — name + markdown instructions, authored directly.

Skills are stored inside the blueprint's `agent_roles` — they are part of
the team declaration, versioned and forked with it.

### The cost contract

Skills are permanent context: they ride along on every turn. Three
guardrails keep that honest:

1. **Caps** — 3 skills per agent in the wizard; the gateway enforces a
   defensive backstop (5 skills / 12k chars each) with annotated
   truncation rather than failure.
2. **Live token estimates** — the wizard shows ~tokens per skill and per
   agent while designing.
3. **Cache placement** — skills are injected into the *stable* segment of
   the agent's system prompt (`## Your Skills`, after identity), so with
   Anthropic prompt caching they read back at ~10% of normal input price
   after the first turn.

```ts
// gateway/src/runtime/agent-context.ts — the stable prompt segment
const skillsSection = skills.length > 0
  ? `\n## Your Skills\nYou have been equipped with the following
     specialist skills. Apply them whenever they are relevant...`
  : "";
```

---

## Part 2 — The Blueprint Experience Loop

### The idea

Inspired by Hermes-style self-improving agent loops (a closed cycle where
completed work is distilled into persistent memory and reusable
procedures), applied at the **team-design level**: the blueprint is the
persistent identity, each company is a session, and lessons flow from
sessions back into the identity.

Without this, every company repeats its predecessors' mistakes — the
fourth team to link an empty repo to Vercel hits the same confusing
failed deploy as the first.

### The loop, in three arrows

**WRITE (in-run).** Every agent has a `record_experience` tool:

```
record_experience(
  lesson:  "Create the Vercel project only after the scaffold
            builds — an empty repo produces a confusing failed
            first deploy.",
  context: "Learned after diagnosing a red first deployment."
)
```

The gateway emits an `experience.recorded` webhook; the engine stamps the
lesson with its provenance — blueprint, source company, orchestration
mode, agent — stores it, and posts a "📚 Lesson recorded" beat to the
company chat so the investor sees learning happen.

**WRITE (at dissolution).** The retrospective judge already reads the
full trajectory to score the run. Its schema-forced output now includes
up to 3 `lessons` — explicitly constrained to *generic process lessons*,
barred from client names, mission specifics, or credentials (essential
for marketplace blueprints run by many customers).

**READ (at deploy).** When a new company launches, the engine renders the
blueprint's freshest active lessons into an `EXPERIENCE.md` workspace
file, and the gateway injects it into every agent's stable system prompt.
The file ends by pointing agents back at `record_experience` — closing
the loop.

### Guardrails

- **Bounded window**: 15 lessons / ~6k chars rendered per launch, newest
  first — old lessons age out instead of growing the prompt forever, and
  the whole section rides the prompt cache.
- **Provenance**: every lesson keeps its source company, agent, and
  orchestration mode, so a bad lesson can be audited and archived
  (status flag) instead of silently steering every future run.
- **Genericity**: both write paths instruct toward short, transferable,
  client-free lessons; the judge's schema enforces it.
- **Visibility**: the creator's Analytics page lists all accumulated
  lessons with provenance; lessons on published blueprints are visible to
  everyone (a quality signal: "this blueprint has learned 23 lessons").

### What is deliberately not built yet

- **Compaction/distillation**: when a blueprint accumulates far more
  lessons than the window, an explicit distillation pass (N lessons → one
  refined principle, Hermes-style) is the natural next step; today, old
  lessons simply age out.
- **Lesson scoring**: lessons are not yet weighted by whether runs that
  saw them scored higher — that correlation is exactly what the
  evaluation loop can measure next.

### The research question this opens

Does the loop work? The methodology is already in place: launch the same
mission from the same blueprint N times in sequence and watch the judge's
scores. If experience transfer is real, later runs should score higher
and burn less budget than earlier ones — and because every lesson is
stamped with its orchestration mode, you can even ask which topology
*learns* fastest.
