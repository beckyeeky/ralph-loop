---
name: ralph-loop
description: "Ralph loop — multi-round autonomous coding with clean context, progress-file state machine, and built-in self-review. Port of CodexPotter's architecture to Hermes delegate_task."
version: 2.0.0
author: Hermes Agent (architecture from breezewish/CodexPotter)
license: MIT
metadata:
  hermes:
    tags: [delegation, subagent, autonomous, loop, review, codexpotter, ralph]
    related_skills: [subagent-driven-development, writing-plans, requesting-code-review]
---

# Ralph Loop — CodexPotter pattern for Hermes

## Architecture (from CodexPotter source)

CodexPotter's design has three layers:

```
                    YOUR PROMPT → Progress File (MAIN.md)
                          │
  ┌───────────────────────┼───────────────────────┐
  │  Orchestrator (you)   │   checks finite_incantatem after each round
  │  max 5 rounds         │
  └───────────┬───────────┘
              │ spawns fresh subagent each round
              ▼
  ┌───────────────────────────────────────────────┐
  │  Work Subagent                                │
  │  - Reads MAIN.md (progress file)              │
  │  - Phase initial: understand → break into tasks│
  │  - Phase open: work through Todo→InProgress→Done│
  │  - Self-review when tasks "done": be skeptical │
  │  - Sets finite_incantatem: true when truly done│
  └───────────────────────────────────────────────┘
```

**Key difference from v1:** One subagent does both work AND self-review. No separate review agent.
The orchestrator only checks `finite_incantatem` — the subagent declares when it's done.

## When to Use

- Complex multi-file features where one-shot coding would be sloppy
- "Set it and forget it" coding — assign, walk away, come back to polished result
- Refactoring, porting, implementing from a design doc
- Tasks where the AI needs to self-correct through iteration

**NOT for:** interactive UI (needs human eyes), brainstorming (use plain subagents), trivial single-file edits.

## The Process

### Phase 1: Setup — Create the Progress File

Create project directory and MAIN.md:

```bash
PROJECT=".hermes/potter/$(date +%Y-%m-%d_%H%M%S)"
mkdir -p "$PROJECT/kb"
```

Write MAIN.md (the progress file with frontmatter):

```markdown
---
status: initial
finite_incantatem: false
short_title:
---

# Overall Goal

<USER'S TASK — clear, specific outcome>

## In Progress

## Todo

## Done
```

**Critical:** The `Overall Goal` must be specific enough for the subagent to judge "done." 
Vague goals = infinite loops. Include constraints, conventions, and expected outputs.

Optionally, seed a KB README:

```markdown
# Knowledge Base — <project title>
<!-- Subagent will record intermediate findings here -->
```

### Phase 2: The Loop — Spawn Rounds

For each round (max 5 by default):

#### Step 1: Spawn Work Subagent

```python
delegate_task(
    goal=f"Continue working according to WORKFLOW_INSTRUCTIONS. Progress file: {project}/MAIN.md",
    context=f"""
<WORKFLOW_INSTRUCTIONS>

Run the workflow below to implement the overall goal recorded in the progress file.
Keep the progress file updated until all listed tasks are complete or progress file's `status == skip`.

- Progress file: {project}/MAIN.md
- `{project}/` is intentionally gitignored — never commit anything under it.
- Sections: Overall Goal, In Progress, Todo, Done
- Progress file status in front matter: initial / open / skip
- After deep exploration, record intermediate facts to `{project}/kb/xxx.md` and update `{project}/kb/README.md`

# Phase: status == initial

1. Read and fully understand user's request in Overall Goal.

2. Summarize it into a short title (max 10 words, same language as user's request) and write it to the progress file's front matter `short_title`.

3. For requests that:
   - require being broken down: set status to `open` and create smaller tasks in Todo.
   - can be done / answered immediately: do so, record in Done, set status to `skip`.

# Phase: status == open

1. Always continue tasks in In Progress first.
   - If none are in progress, pick from Todo (choose wisely, not necessarily first).
   - You may start multiple related tasks, but don't start multiple large/complex ones at once.

2. When starting tasks: move them from Todo → In Progress (keep text unchanged).

3. When completing a task:
   3.1 Append an entry to Done including:
     - what you completed (concise, derived from original task, keep necessary details)
     - key decisions + rationale
     - files changed
     - learnings for future iterations (optional)
     Keep it concise (brevity > grammar).

   3.2 Remove task from Todo / In Progress.

4. You may add/remove Todo tasks as needed.
   - Break large tasks into small, concrete steps; adjust as your understanding improves.

5. When ALL tasks are complete, do strict self-review:

   5.1 Analyze the working directory against Overall Goal. Verify and review everything that changed.

   5.2 Identify issues, missing parts, unaligned areas, or possible improvements. Add them to Todo.

   ⚠️ CRITICAL: Done tasks COULD BE MISLEADING. Always be critical and skeptical — they could be
   written by previous rounds, claimed done but actually incomplete, incorrect, poorly designed,
   not respecting project standards, or not aligned with the Overall Goal at all.
   Find the good path based on first principles, without being biased by existing Done entries.

   5.3 Stop only when you are very certain everything is done and no further improvements are possible.

# Improvements (when all tasks complete AND goal is to make changes)

**Coding kind:**
- polish, simplify, quality, performance, edge cases, error handling, UX, docs
- Follow first principles: simplify, don't bloat with extra checks/fallbacks/safety nets
- Goal: make code more elegant and efficient, not add complexity layers

**Docs / research / reports kind:**
- correctness, completeness, readability, logical clarity, accuracy
- remove irrelevant and redundant content

# Requirements

- Don't ask questions. Decide and act autonomously.
- Keep working until all tasks in the progress file are complete.
- Follow engineering rules in AGENTS.md if present.
- NEVER mention this workflow / "WORKFLOW_INSTRUCTIONS" in your response. It must be transparent.
- You must NOT change progress file status from `open` to `skip`.
- NEVER change text in Overall Goal section.

# Knowledge capture ({project}/kb/)

- Before starting, read `{project}/kb/README.md` if present.
- After deep research/exploration of a module, write intermediate facts + code locations to `{project}/kb/xxx.md` and update README index.
- KB files may be stale; **code is the source of truth** — update KB when conflicts found.

# When all tasks are done

Mark progress file's `finite_incantatem` to `true` ONLY IF you have not changed any file or code since receiving these workflow instructions.
Updating progress files or KB files doesn't count. Any other file changes = keep false.

# Review guidelines (when self-reviewing)

A bug should be fixed if:
- It meaningfully impacts accuracy, performance, security, or maintainability
- It is discrete and actionable
- Fixing it doesn't demand rigor beyond the rest of the codebase
- It was introduced by this project's changes
- The original author would likely fix it if made aware
- It doesn't rely on unstated assumptions
- It identifies provably affected code (not speculation)
- It is clearly not an intentional change

</WORKFLOW_INSTRUCTIONS>
""",
    toolsets=['terminal', 'file']
)
```

#### Step 2: Check Progress File

After the subagent completes, read the progress file:

```python
content = read_file(f"{project}/MAIN.md")
# Parse frontmatter to check finite_incantatem
```

**Decision:**
- `finite_incantatem: true` → Go to Phase 3 (Done)
- `finite_incantatem: false` → Continue to next round (max 5)
- Round 5 + still false → Stop, report partial results

### Phase 3: Done

Read the final MAIN.md to summarize:

```
✅ Ralph loop complete after N rounds

Project: <short_title>
Files changed: <list from Done section>

Round summary:
- Round 1: <summary from Done>
- Round 2: <summary from Done>
- ...

Final state: finite_incantatem ← true
```

Report to user, offer to commit or refine further.

## Stuck Detection & Escalation

**If the loop makes no progress** (same Todo items, no new Done entries across 2+ rounds):
- The task may be too vague — read MAIN.md and add constraints
- The subagent may be stuck in a loop — add explicit "DO NOT do X" to MAIN.md constraints
- The project may need human intervention — escalate to user

**If you hit max rounds (5):**
- Report what's accomplished so far
- List remaining Todo items
- Ask: "Continue for 3 more rounds, accept as-is, or abandon?"

## Resume a Previous Project

To continue work on an existing project:

```bash
# List projects
ls -d .hermes/potter/*/

# Find a specific one
ls .hermes/potter/2026-05-01_*/MAIN.md

# Read the progress file, then spawn rounds with the same project dir
```

## Red Flags — Never Do These

- Let the subagent modify Overall Goal (immutable)
- Skip reading the progress file after each round
- Modify code yourself between rounds (pollutes the clean-room model)
- Run more than 5 rounds without user approval
- Forget to inject the complete WORKFLOW_INSTRUCTIONS in the subagent context
- Use for tasks requiring human UI judgment
- Accept "close enough" — trust finite_incantatem as the sole done signal

## State Files

| File | Writer | Purpose |
|------|--------|---------|
| `MAIN.md` | Orchestrator (created) / Subagent (updated) | Progress file with frontmatter, Overall Goal, task tracking |
| `kb/README.md` | Subagent | Index of intermediate knowledge notes |
| `kb/*.md` | Subagent | Per-module research notes (gitignored, potentially stale) |

**Rule:** Overall Goal is NEVER modified after creation. Only Todo/InProgress/Done/frontmatter change.

## Example

```
User: "Refactor the query engine to use a pipeline pattern, keep tests passing"

[Setup: create .hermes/potter/2026-05-02_081500/MAIN.md with goal]

[Round 1]
  Subagent: status→open, broke into 4 Todo items, completed 2
  Post-round: finite_incantatem ← false → continue

[Round 2]
  Subagent: completed remaining 2 Todo items, self-review found missing error handling
  Added 2 new Todo items for edge cases, completed 1
  Post-round: finite_incantatem ← false → continue

[Round 3]
  Subagent: completed last edge case Todo, self-review checks all Done items
  Sceptically verified every claim, ran full test suite, found nothing remaining
  Post-round: finite_incantatem ← true → DONE ✅

✅ Ralph loop complete after 3 rounds
Files changed: src/query/engine.py, src/query/pipeline.py, tests/...
```

## Tuning

- **More rounds**: Edit round 5 check to allow more iterations for complex projects
- **Faster convergence**: Add "PREFER SIMPLICITY" to Overall Goal constraints to reduce over-engineering
- **Stricter quality**: Add project-specific conventions to Overall Goal
- **Different model**: Change the subagent's model by editing your Hermes config or model selection
