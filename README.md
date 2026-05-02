# Ralph Loop

> **Multi-round autonomous coding with clean context and built-in self-review.**
>
> *"Tell it what you want. It keeps working until it's right."*

The Ralph Loop is a pattern for autonomous AI coding: the agent works in rounds, each with a **fresh context**, using the **filesystem as memory**. After each round it **self-reviews** — skeptically checking its own work — and iterates until satisfied. No context accumulation. No degradation. Just a progress file and relentless self-correction.

Named after [Ralph Wiggum](https://ghuntley.com/ralph/), the pattern was pioneered by [CodexPotter](https://github.com/breezewish/CodexPotter) for OpenAI Codex. This repo adapts it for **agent-based systems** — currently [Hermes Agent](https://github.com/nousresearch/hermes-agent), with the architecture portable to any agent that can spawn subagents.

---

## How It Works

```
                     YOUR PROMPT → Progress File (MAIN.md)
                           │
   ┌───────────────────────┼───────────────────────┐
   │   Orchestrator        │   Checks finite_incantatem after each round
   │   Max N rounds        │
   └───────────┬───────────┘
               │ spawns fresh subagent each round
               ▼
   ┌───────────────────────────────────────────────┐
   │   Work Subagent (fresh context each round)     │
   │   - Reads MAIN.md                             │
   │   - Breaks goal into tasks                    │
   │   - Works through Todo → InProgress → Done    │
   │   - Self-reviews: be skeptical of Done tasks  │
   │   - Sets finite_incantatem: true when done    │
   └───────────────────────────────────────────────┘
```

The orchestrator does almost nothing — just spawns rounds and checks one flag. All the intelligence is in the subagent's **developer prompt**, which tells it how to use the progress file.

---

## Three Layers

### 1. The Progress File (`MAIN.md`)

A Markdown file with YAML frontmatter that serves as the agent's durable memory:

```yaml
---
status: initial          # initial → open → skip
finite_incantatem: false # set to true when truly done
short_title:             # auto-generated summary
engine: hermes           # hermes | codex
---

# Overall Goal
<your task here — never modified by the agent>

## In Progress
## Todo
## Done
```

The agent moves through two phases:
- **`initial`** — understand the goal, break it into tasks
- **`open`** — work through tasks, self-review, declare done

### 2. The Developer Prompt

The ~1KB instruction set injected into every subagent. It tells the agent:

- How to use the progress file state machine
- To break goals into concrete tasks
- To self-review **skeptically** when tasks are "done"
- To keep polishing until no improvements remain
- To **never ask questions** — decide and act autonomously

The critical line:

> *"Done tasks COULD BE MISLEADING. Always be critical and skeptical — they could be written by previous rounds, claimed done but actually incomplete, incorrect, or not aligned with the Overall Goal at all."*

### 3. The Orchestrator

Minimal. After each round:
1. Read the progress file
2. Check `finite_incantatem`
3. If `false` → spawn another round
4. If `true` → done

---

## Why This Pattern?

| Problem | Ralph Loop Solution |
|---------|-------------------|
| Context degradation over long sessions | Fresh context every round |
| AI claims done but work is incomplete | Built-in skeptical self-review |
| Human needs to review every output | Agent reviews itself, human only sees final result |
| Prompt engineering overhead | One lean developer prompt, reused every round |
| Tool sprawl (separate review agents) | One agent does work + review |

**vs. one-shot coding:** Multiple iterations catch edge cases, missing requirements, and quality issues automatically.

**vs. chat-based iteration:** No context accumulation. Round 5 is as sharp as round 1.

**vs. separate review agents:** Half the cost, and the agent reviewing its own work is naturally more critical — it knows what shortcuts it took.

---

## Dual Engine — Hermes or Codex CLI

The Ralph Loop supports two execution engines, selected via the `engine:` field in MAIN.md frontmatter:

| Engine | Mechanism | Model | Use when |
|--------|-----------|-------|----------|
| `hermes` (default) | `delegate_task` subagent | Any Hermes model | You want to use your own models (deepseek, minimax, kimi...) |
| `codex` | `codex exec` CLI (stdin pipe) | OpenAI gpt-5.4 / gpt-5.5 | You have a Codex subscription and want the original CodexPotter feel |

**Codex engine** mirrors CodexPotter's architecture exactly — each round pipes the developer prompt to `codex exec`:

```bash
cat <<'DEVENDOF' | codex exec -c approval_policy=never -m gpt-5.4
<WORKFLOW_INSTRUCTIONS with progress file path>
</WORKFLOW_INSTRUCTIONS>
DEVENDOF
```

Key details: `-c approval_policy=never` (not `--approval-policy`), prompt via stdin (not CLI argument), must be inside a git repo. Full reference: `hermes/ralph-loop/references/codex-cli.md`.

## Getting Started (Hermes Agent)

### Prerequisites

- [Hermes Agent](https://github.com/nousresearch/hermes-agent) installed
- The `ralph-loop` skill loaded

### Install the Skill

```bash
# Copy the skill into Hermes
cp hermes/ralph-loop/SKILL.md ~/.hermes/skills/software-development/ralph-loop/
```

Or install from this repo:

```bash
git clone https://github.com/beckyeeky/ralph-loop.git
cp ralph-loop/hermes/ralph-loop/SKILL.md ~/.hermes/skills/software-development/ralph-loop/
```

### Usage

Just tell Hermes what you want:

> "Ralph loop: refactor the query engine to use a pipeline pattern, keep all tests passing"

The agent will:
1. Create `.hermes/potter/<timestamp>/MAIN.md` with your goal
2. Spawn a subagent that breaks the goal into tasks and starts working
3. After each round, check if the agent declares done
4. Loop up to 5 rounds until `finite_incantatem: true`

### Tuning

- **More rounds:** Edit the max-round check for complex projects
- **Faster convergence:** Add "PREFER SIMPLICITY" to your goal
- **Different model (Hermes):** Change your Hermes config or model selection
- **Switch to Codex engine:** Set `engine: codex` in MAIN.md to use OpenAI Codex CLI for a round

---

## Adapting to Other Agents

The Ralph Loop is agent-agnostic. To port it:

1. **Progress file** — any Markdown file with the frontmatter + sections
2. **Developer prompt** — copy the `<WORKFLOW_INSTRUCTIONS>` from the skill file
3. **Orchestrator** — a loop that spawns fresh agent sessions and checks `finite_incantatem`

The pattern works anywhere you can:
- Spawn a fresh agent per round
- Have that agent read/write files on disk
- Check a flag after each round

---

## Credits

- **CodexPotter** ([breezewish/CodexPotter](https://github.com/breezewish/CodexPotter)) — the original implementation for OpenAI Codex CLI. All core architecture: progress file state machine, developer prompt design, `finite_incantatem` mechanism, knowledge base conventions.
- **Ralph Wiggum pattern** — named by [Geoff Huntley](https://ghuntley.com/ralph/) after the Simpsons character: "I'm helping!" The pattern of an agent that keeps trying until the job is actually done.

---

## License

MIT — same as the original CodexPotter.
