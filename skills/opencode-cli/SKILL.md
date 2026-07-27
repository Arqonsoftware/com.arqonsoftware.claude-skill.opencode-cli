---
name: opencode-cli
description: Use when the user explicitly asks to delegate a coding task to "opencode" (the opencode CLI) or a specific opencode-hosted model, or asks how to drive opencode subagents from Claude Code. Not a general coding tool — only invoke when the user names opencode by name.
---

# Driving the `opencode` CLI from Claude Code

`opencode` is a separate agentic coding CLI, unrelated to Claude Code. There is no MCP
connector/integration for it — the only way to use it from Claude Code is shelling out via
Bash. It is not aware of this conversation; every invocation needs full, self-contained
context in the prompt.

## Before starting

1. Confirm the binary is installed and on `PATH`: `opencode --version`. If missing, tell the
   user rather than trying to install it yourself.
2. Discover what's actually available on this install rather than assuming — model and agent
   names vary by user/config:
   - `opencode models` — lists all available `provider/model` ids. Grep for what the user
     asked for, e.g. `opencode models | grep -i <model-name>`.
   - `opencode agent list` — lists configured agents. Typically includes a primary
     implementation agent (commonly named `build`), a `plan` agent, and read-only subagents
     such as `explore` or `general` that a primary agent can dispatch internally.

## Picking a model

`opencode models` output typically includes a mix of free-tier and paid models — the split
usually shows up as a `-free` suffix on the model id (e.g. `provider/foo-flash-free` vs.
`provider/foo-flash`). Match the model to the task, don't default to the biggest/priciest one:

- **Read-only investigation / exploration / "just look into X and report back"** — use a free
  or "flash"/"mini" tier model. These tasks are grep-and-read heavy, not reasoning-heavy, so a
  cheap model does the job. Don't spend a premium model's budget on a task that's mostly file
  reads.
- **Real implementation work** (writing/editing code that needs to be correct, non-trivial
  logic, cross-file consistency) — use a stronger paid model. Still pick the more
  cost-effective option among the strong ones rather than reaching for the most expensive
  model by default — check `opencode models` for what's available and ask the user if unsure
  which paid tier they want to spend on, especially if one of the options is known to be
  notably more expensive (e.g. a "pro"/flagship-labeled variant, or a model the user has
  previously flagged as costly).
- If the user names a specific model, use exactly that one — don't substitute your own
  judgment for an explicit instruction. If they instead just say "use opencode" with no model
  named, pick based on the task type above and briefly say which one and why.
- If the user pushes back on a model choice as too expensive, swap to a cheaper capable
  option for the rest of the task and remember that preference for future picks in the same
  session.

### Models observed to work well (update this list as you learn more)

- **`opencode-go/minimax-m2.7`** — good default for real implementation work: fast, and
  produced clean, correctly-scoped diffs (targeted fixes across multiple files, passed
  lint/typecheck first try, didn't wander outside the requested scope) on a multi-file
  responsive-design fix task. Cheaper/faster than reaching for a "pro"/flagship model by
  default — try this first for implementation tasks before escalating.
- **`opencode-go/kimi-k2.7-code`** — capable at diagnosing and fixing messy/inconsistent
  existing code (e.g. reconciling a half-applied theming strategy into one consistent
  approach), but flagged explicitly by the user as expensive — don't default to it; use it
  when a task specifically needs stronger reasoning about a tangled existing implementation,
  not for routine implementation work.
- **`opencode/deepseek-v4-flash-free`** — solid for free-tier read-only exploration/audit
  tasks (e.g. a prioritized mobile-responsiveness audit across ~10 files) — thorough and
  correctly scoped for a report-only task.
- **`opencode-go/deepseek-v4-pro`** — capable for implementation work but paid; no strong
  signal yet that it outperforms `minimax-m2.7` for typical tasks, so prefer `minimax-m2.7`
  first unless the user asks for this specifically.
- If the user asks for a model that doesn't appear in `opencode models`' output (e.g. a
  version number that doesn't exist), don't guess a substitute — tell them it's unavailable
  and ask which of the actually-listed variants they want (`AskUserQuestion` if you have it,
  otherwise just ask in text).

## Key commands

- `opencode run [message..]` — non-interactive one-shot run. Key flags:
  - **positional `message` (preferred)** — the task prompt; pass as one argv so shell quoting
    stays reliable (see “Passing the prompt” below)
  - `--agent <name>` — which agent drives the run (use the primary/implementation agent for
    real work, e.g. `build`)
  - `--model <provider>/<model>` — provider/model override, from `opencode models`
  - `--auto` — auto-approve permissions (needed for unattended/background runs — dangerous,
    only use when the task and directory scope are already trusted by the user)
  - `--dir <path>` — project directory for the session (repo root you want edits in). Prefer
    this over relying on the shell’s cwd alone when backgrounding.
  - `--title <string>` — human-readable session title (helps when listing sessions later)
  - `--prompt` — **avoid for long multi-line tasks when backgrounding**; prefer positional
    message from a file (see below). Mis-wired `--prompt` + heredoc under `nohup` has been
    observed to print `opencode run --help` and exit with no session.

### Passing the prompt (critical — easy to get wrong)

Long, multi-line prompts break easily under `nohup` / nested quotes. **Do not** rely on:

```bash
# BAD patterns observed to fail (CLI prints help, no agent starts):
nohup opencode run --prompt "$(cat <<'EOF'
...huge prompt...
EOF
)" ...
```

**Do this instead** — write the prompt to a file, pass it as a single positional argument:

```bash
PROMPT_FILE=/tmp/opencode/task-prompt.txt
LOG=/tmp/opencode/task.log
# write a full self-contained prompt into $PROMPT_FILE (heredoc to the file is fine)
nohup opencode run \
  --agent build \
  --model <provider>/<model> \
  --auto \
  --dir /absolute/path/to/repo \
  --title "short-task-label" \
  "$(cat "$PROMPT_FILE")" \
  > "$LOG" 2>&1 &
echo "PID=$! LOG=$LOG"
```

**Smoke-check within ~3s** that the run actually started:

- Log should show something like `> build · <model-name>` and/or `# Todos`, **not** the full
  yargs help text for `opencode run`.
- `ps -p $PID` should still be alive (or already finished with a real transcript, not help).
- If the log is only help output, the argv never reached the agent — fix quoting and relaunch;
  do not wait for a “stuck” job.

### “Use kimi code / use opencode as a subagent”

- User phrases like **“kimi code”**, **“kimi-k2.7-code”**, or **“opencode-go/kimi-k2.7-code”**
  mean model id **`opencode-go/kimi-k2.7-code`** (confirm with `opencode models | grep -i kimi`).
  Related ids seen on this install: `opencode-go/kimi-k2.6`, `opencode-go/kimi-k2.7-code`,
  `opencode-go/kimi-k3`.
- You still drive the **outer** shell yourself. “As a subagent” means: run `opencode run
  --agent build` and **in the prompt** tell that primary agent to dispatch its **internal**
  `explore` / `general` subagent first. You cannot shell out to `explore` directly.

## Getting opencode to use its explore/research subagents

You cannot directly invoke a read-only subagent (e.g. `explore`) yourself from the outer
shell — it's internal, only a primary agent (e.g. `build`) can dispatch it. Instead, instruct
this explicitly in the prompt text passed to `opencode run`, e.g.:

> "First, dispatch your explore subagent to survey X, Y, Z. Then, based on that, implement ..."

The primary agent will invoke the subagent on its own as part of executing the run. In logs
you may see a line like `• Survey … Explore Agent` once it actually dispatched.

## Practical pattern: background + monitor, don't block

`opencode run` on a real implementation task can run for many minutes. A single blocking
foreground call risks hitting your own tool/turn timeout before opencode finishes — and an
empty/rejected tool result in that case does NOT mean opencode failed; it means the result is
unknown. Always run it backgrounded with output to a log file, then watch the log/process
instead of blocking:

```bash
PROMPT_FILE=/tmp/opencode/task-prompt.txt
LOG=/tmp/opencode/task.log
nohup opencode run \
  --agent build \
  --model <provider>/<model> \
  --auto \
  --dir /absolute/path/to/repo \
  --title "short-task-label" \
  "$(cat "$PROMPT_FILE")" \
  > "$LOG" 2>&1 &
echo "PID=$! LOG=$LOG"
# immediately verify it didn't just dump --help
sleep 2
head -20 "$LOG"
ps -p $! -o pid,etime,cmd
```

Then poll or use a monitoring tool against that PID/log rather than waiting on a single long
blocking call, e.g.:

```bash
while kill -0 <PID> 2>/dev/null; do
  sleep 15
  # detect wandering: no edits after several minutes
  git -C /absolute/path/to/repo status --short | head
  tail -n 30 "$LOG"
done
tail -n 100 "$LOG"
```
## Use a git worktree when the target repo has a live dev server watching it

If the target repo has a running dev server / file watcher (a hot-reload build, `vite build
--watch`, `wrangler dev`, etc.) that a human is actively using, do NOT run `opencode run
--agent build` directly against that same working tree. A write-capable agent edits files
incrementally over its whole run — for a bundled frontend this means the app can be in a
broken, half-edited state (mismatched imports/usages, syntax errors) for the entire duration
of the run, and since bundlers typically ship one JS bundle, a transient break in the file
being edited can crash *every* route the user is looking at, not just the one being changed.

Instead:
1. Create a separate git worktree for the task: `git worktree add <path> -b <branch-name>`.
2. Point `opencode run`'s prompt at that worktree path instead of the live directory (or `cd`
   into it before invoking, whichever fits the task).
3. Once the run finishes and passes lint/typecheck, merge the branch back or `git diff` it
   into the main working tree, rather than letting the agent edit the live tree directly.
4. Clean up the worktree (`git worktree remove <path>`) once done.

This keeps the human's live dev server stable throughout the whole run instead of only
being stable at the very end. Only skip this for read-only `explore`/investigation runs,
which never write and so can't destabilize anything.

## Writing the task prompt

Since opencode has zero memory of your conversation, the prompt must be fully self-contained:

- Exact file/directory paths in the target repo.
- Any relevant history it needs to know (e.g. "a previous attempt at X was reverted because
  Y — check `git log`/`git show` in the target repo before delegating, and bake anything
  relevant into the prompt so it isn't repeated").
- Explicit instruction to dispatch its explore/research subagent first, if the task benefits
  from codebase survey before editing.
- Explicit step-by-step implementation instructions and scope boundaries (what's in/out of
  scope, so it doesn't over-expand the change).
- Instruction to run the target project's lint/typecheck/build scripts at the end and fix any
  issues found.
- Instruction to report a concise summary of every file changed and what was done.

## Watch for wandering — kill it if it's just reading, not editing

Some models (observed with `kimi-k3` on a "polish this page's UI" task) will spend many
minutes reading broadly unrelated files — READMEs, vendored library internals, unrelated
config — without making any actual edits, especially on vaguely-scoped "make this look
better"/"do a professional pass" style prompts. This wastes time and tokens for no output.

- Periodically check `git diff --stat` (or `git status --short`) in the target directory
  partway through a long run. If it's been running for several minutes with zero file changes
  and the log shows it reading files unrelated to the stated task, it's very likely wandering,
  not making progress — don't just wait it out.
- If it's wandering, kill the process, tear down the worktree, and either retry with a more
  tightly-scoped prompt (name the exact file(s) and exact issue(s) to fix, not "do a
  professional pass") or just do the fix yourself directly if it's small enough that you
  already understand exactly what's needed — don't re-delegate the same vague prompt to the
  same or another model and hope for a different outcome.
- Prefer concrete, itemized instructions ("fix X at line ~N: currently does A, should do B")
  over open-ended aesthetic judgment calls ("make this look professional") — the latter is
  exactly what tends to send a model reading through the whole codebase for context it
  doesn't actually need, instead of just making the fix.

## Lessons learned

- Treat a blocking foreground `opencode run` that returns an empty/rejected result as
  "unknown outcome, go check the process table and the target repo's git diff" — not as
  confirmation of failure.
- Check the target repo's git history for prior related work before delegating a task that
  might overlap with something already attempted and reverted.
- `--auto` is required for unattended background runs (otherwise it blocks on interactive
  permission prompts), but only use it in directories/tasks the user has already trusted you
  with — it auto-approves file writes and other actions without per-step confirmation.
- **Prompt argv vs help dump (2026-07):** Passing a huge multi-line string via `--prompt` under
  `nohup` (especially with nested `$(cat <<'EOF')`) caused `opencode run` to print its help
  text and exit without starting a session. **Reliable pattern:** write prompt to a file →
  pass `"$(cat "$PROMPT_FILE")"` as the **positional** message → use `--dir` and `--title` →
  smoke-check the log for `> build · …` within a few seconds.
- **`--dir`:** Set the absolute repo path with `--dir` so background jobs don’t depend on
  which directory the shell happened to be in when `nohup` started.
- **User alias “kimi code”:** Map to `opencode-go/kimi-k2.7-code` after confirming with
  `opencode models | grep -i kimi`. Still expensive — only when the user asked for it (e.g.
  UI inconsistency / messy reconciliation), not as the default implementer.
- **Subagent wording:** “Use opencode as a subagent” from the outer agent means spawn
  `opencode run --agent build …` and tell *that* agent to use its explore subagent; there is
  no separate outer CLI for `explore`.
- When the skill gains new hard-won usage facts (CLI flags, broken quoting, model aliases),
  **update this SKILL.md in the same turn** so the next session doesn’t re-learn them.
