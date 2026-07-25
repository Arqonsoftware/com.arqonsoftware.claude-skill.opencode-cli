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

## Key commands

- `opencode run [message..]` — non-interactive one-shot run. Key flags:
  - `--agent <name>` — which agent drives the run (use the primary/implementation agent for
    real work, e.g. `build`)
  - `--model <provider>/<model>` — provider/model override, from `opencode models`
  - `--auto` — auto-approve permissions (needed for unattended/background runs — dangerous,
    only use when the task and directory scope are already trusted by the user)
  - `--prompt` — alternative to passing the message positionally

## Getting opencode to use its explore/research subagents

You cannot directly invoke a read-only subagent (e.g. `explore`) yourself from the outer
shell — it's internal, only a primary agent (e.g. `build`) can dispatch it. Instead, instruct
this explicitly in the prompt text passed to `opencode run`, e.g.:

> "First, dispatch your explore subagent to survey X, Y, Z. Then, based on that, implement ..."

The primary agent will invoke the subagent on its own as part of executing the run.

## Practical pattern: background + monitor, don't block

`opencode run` on a real implementation task can run for many minutes. A single blocking
foreground call risks hitting your own tool/turn timeout before opencode finishes — and an
empty/rejected tool result in that case does NOT mean opencode failed; it means the result is
unknown. Always run it backgrounded with output to a log file, then watch the log/process
instead of blocking:

```bash
nohup opencode run \
  --agent build \
  --model <provider>/<model> \
  --auto \
  "<full self-contained task prompt — see below>" \
  > /path/to/scratch/opencode_run.log 2>&1 &
echo "PID: $!"
```

Then poll or use a monitoring tool against that PID/log rather than waiting on a single long
blocking call, e.g.:

```bash
while kill -0 <PID> 2>/dev/null; do sleep 5; done; tail -n 100 /path/to/scratch/opencode_run.log
```

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

## Lessons learned

- Treat a blocking foreground `opencode run` that returns an empty/rejected result as
  "unknown outcome, go check the process table and the target repo's git diff" — not as
  confirmation of failure.
- Check the target repo's git history for prior related work before delegating a task that
  might overlap with something already attempted and reverted.
- `--auto` is required for unattended background runs (otherwise it blocks on interactive
  permission prompts), but only use it in directories/tasks the user has already trusted you
  with — it auto-approves file writes and other actions without per-step confirmation.
