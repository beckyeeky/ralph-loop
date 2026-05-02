# Codex CLI Reference for Ralph Loop

## Environment

| Detail | Value |
|--------|-------|
| CLI version | `codex-cli v0.125.0` (research preview) |
| Binary path | `/root/.nvm/versions/node/v22.22.0/bin/codex` |
| Default model | `gpt-5.4` |
| Provider | openai |

## Invocation Recipe

### Minimal working invocation (read-only)

```bash
# Must be inside a git repo (or pass --skip-git-repo-check)
cd <project-dir>
echo "<prompt>" | codex exec -c approval_policy=never
```

### Write-enabled (default for Ralph Loop coding tasks)

```bash
# -s workspace-write allows git clone/modify/commit/push
echo "<prompt>" | codex exec -c approval_policy=never -s workspace-write
```

### Full access (trusted environments only)

```bash
echo "<prompt>" | codex exec -c approval_policy=never -s danger-full-access
```

### With model override

```bash
echo "<prompt>" | codex exec -c approval_policy=never -s workspace-write -m gpt-5.5
```

### Sandbox Modes Reference

| Mode | `-s` flag | Write files? | Git operations? | Network? | Use case |
|------|-----------|-------------|-----------------|----------|----------|
| `read-only` | default | ❌ | ❌ | ✅ | Read-only analysis, code review |
| `workspace-write` | `-s workspace-write` | ✅ (workspace only) | ✅ clone/modify/commit/push | ✅ | **Default for coding tasks** |
| `danger-full-access` | `-s danger-full-access` | ✅ (full system) | ✅ | ✅ | Trusted env, system-level ops |

### Examples seen in this env

```bash
# Works: pipe prompt via stdin
echo 'read test.txt and reply with just "OK"' | codex exec -c approval_policy=never

# Output structure:
#   OpenAI Codex v0.125.0 (research preview)
#   --------
#   workdir: /tmp/codex-test
#   model: gpt-5.4
#   approval: never
#   sandbox: read-only
#   session id: 019de702-...
#   --------
#   (agent output follows)
```

## Pitfalls

1. **`--approval-policy` flag doesn't exist on `codex exec`** — use `-c approval_policy=never` instead. Passing `--approval-policy` causes an error.

2. **Must be in a git repo** — `codex exec` refuses to run outside a git directory unless `--skip-git-repo-check` is passed. Always `git init` first if the project dir isn't a repo.

3. **`bubblewrap` missing** — generates a warning on every run. Codex uses a vendored fallback. Non-fatal, but install with `apt-get install bubblewrap` to silence it.

4. **Rollout recording error at end** — `ERROR codex_core::session: failed to record rollout items: thread <id> not found`. Non-fatal, happens at the end of every exec. Ignore.

5. **Stdin mode vs argument mode** — passing the prompt as an argument works too, but piping via stdin is more reliable for multi-line prompts and avoids shell escaping issues.

6. **Default sandbox is `read-only`** — if codex can't write files, verify you're using `-s workspace-write`. Without this flag, git operations and file modifications will fail silently or raise permission errors.

## Integration with Ralph Loop

For Codex engine mode in the ralph-loop skill, the orchestrator replaces `delegate_task` with:

```bash
# Round loop: inject the developer prompt + progress file reference
# Default: workspace-write sandbox for full git workflow support
SANDBOX_MODE=$(grep 'codex_sandbox:' .hermes/potter/<project>/MAIN.md | head -1)

cat <<'DEVENDOF' | codex exec -c approval_policy=never -s workspace-write -m gpt-5.4

<WORKFLOW_INSTRUCTIONS injected here>
Progress file: .hermes/potter/<project>/MAIN.md

</WORKFLOW_INSTRUCTIONS>
DEVENDOF
```

After each round, read the progress file and check `finite_incantatem`. If false, spawn another codex exec round.

### Sandbox Decision Flow

```
Does the task need to modify files?
├─ Yes → codex_sandbox: workspace-write (default) ← use this for all coding tasks
│         Supports: git clone, edit, commit, push
└─ No → codex_sandbox: readonly
          Supports: read files, analysis, code review only
```
