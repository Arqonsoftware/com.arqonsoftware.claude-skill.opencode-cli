---
name: opencode-cli
description: |
  Drive the opencode CLI (including kimi). ALWAYS use for ANY UI task — layout, styling,
  components, pages, timelines/steppers, dashboards, forms chrome, Tailwind, React UI,
  polish, redesign, “make it look better”, icons, responsive, journey/step UX — delegate
  to kimi with **direct, concrete task lists** (not vague explore). Also when the user names
  opencode/kimi, or asks how to run opencode subagents.
---

# Driving the `opencode` CLI (kimi)

`opencode` is a separate agentic coding CLI. No MCP — shell out via Bash only.
It has **zero memory** of this conversation; every run needs a full prompt.

Default UI model: **`opencode-go/kimi-k2.7-code`**
(confirm: `opencode models | grep -i kimi`).

---

## UI tasks → always delegate to kimi (mandatory)

**User rule: all UI work must be done by kimi, not by the outer agent.**

When the task is UI-related in any substantial way, you **must**:

1. **Not implement the UI yourself** (no hand-editing TSX/CSS for look-and-feel, layout,
   components, pages, timelines, form chrome, etc.) — even if “faster.”
2. **Delegate immediately** via `opencode run` with **`opencode-go/kimi-k2.7-code`**.
3. Give kimi a **direct task prompt** (see below) — not a research brief.
4. Use `--agent build --auto`, a **git worktree** when a live dev server watches the repo,
   prompt file + positional message, background + monitor.
5. After kimi finishes: review the diff, typecheck/lint, then commit/merge/push only when
   the user wants it (or already said “push”).

### What counts as UI (delegate)

- Pages, layouts, navigation, sidebars, headers, footers, timelines, steppers, wizards
- Styling / Tailwind / CSS / themes / spacing / typography / colors
- React presentation components (tables chrome, empty states, modals, buttons rows)
- Responsive / polish / redesign / “make it professional”
- Form chrome and field arrangement (not pure backend validation alone)
- Icons, badges, chips, progress as visual design

### What is not UI (you may implement)

- Pure backend, API, workers, DB, PDF HTML generation, maps/files upload logic
- One-line non-visual bugfixes (wrong API field name)
- i18n key strings only when not part of a broader UI redesign
- Git/ops/deploy, skill/docs updates

### Hybrid tasks

**You** do non-UI (API/backend). **Kimi** does all UI file edits. Do not “just fix the
CSS yourself” because kimi is slow. Do not ship functional UI wiring yourself and skip
kimi — if UI files need to change, kimi owns those files.

---

## Direct tasks only (critical — user rule 2026-07)

**Do not give kimi vague, exploratory, or “figure it out” prompts.** Those cause
multi-minute read loops (explore subagent, grepping `node_modules`, reading READMEs)
with **zero edits**.

### Outer agent responsibilities before launch

You prepare the task; kimi executes it. Before `opencode run`:

1. **Locate the files yourself** (grep/read). Put exact paths in the prompt.
2. **State current behavior vs desired behavior** in 2–6 bullets.
3. **List product rules** as hard constraints (e.g. “no password on invite form”).
4. **Name out-of-scope paths** (`src/`, `migrations/`, etc.).
5. Prefer **numbered edit steps** (“1. Edit X… 2. Edit Y… 3. Run tsc”) over essays.
6. If backend already shipped, say **“backend DONE — polish/wire UI only.”**
7. Sync the worktree to the correct base commit/branch **before** launch.

### Prompt shape (use this template)

Keep prompts short and imperative. Prefer this structure:

```text
# Direct task: <one-line goal>

Work only in: <absolute worktree path>
Model should EDIT immediately — do NOT run explore/general subagents first.
Do NOT read node_modules, READMEs, or unrelated pages.

## Product rules (hard)
- …
- …

## Files (only these)
1. path/A.tsx — do: …
2. path/B.tsx — do: …
3. path/i18n.tsx — only if new strings needed (all locales)

## Already done (do not re-implement logic)
- Backend: …
- API client: …

## Reference only (read if needed for style match, max 2 files)
- LoginPage.tsx / AuthShell.tsx

## Forbidden
- Backend, migrations, inventing features, password-on-create, unsolicited chrome

## Done
1. Edit the files above
2. cd web && npx tsc --noEmit
3. Summarize diff — do not commit/push
```

### Direct vs bad prompts

| Bad (causes wander) | Good (direct) |
|---------------------|---------------|
| “Implement invite UI; survey the codebase first” | “Edit TeamPage.tsx: remove password fields; CTA Invite; call inviteTeamMember without password” |
| “Make Team page professional” | “TeamPage: invite form spacing like LoginPage Card; button Invite; modal title Reset password” |
| “First dispatch explore subagent…” | **Never** for known-file UI — start with Edit |
| Optional password / dual flows | One clear product rule |
| Long backend API essays | One line: “API already in api.ts: inviteTeamMember / acceptTeamInvite” |

### Do NOT tell kimi to explore first (UI tasks)

For UI with known files, **omit** any “dispatch explore/general subagent” instruction.
That pattern was useful for unknown codebases; for UI polish it burns tokens reading
unrelated files. Kimi should open the listed files and edit.

Only ask for a subagent survey when:
- The user asked for a **read-only audit/report**, or
- You genuinely do not know which files to touch (you should usually find them first).

### Kill and re-prompt if not writing

- After ~2–3 min with **zero** `git status` changes and log still on Read/Explore → **kill**.
- Relaunch with a **shorter, more direct** prompt (exact files + exact edits).
- Do **not** re-launch the same vague prompt.
- One-file CSS with a known fix: if kimi still doesn’t write in ~1–2 min, outer agent may
  finish that single known fix only when the user is blocked — prefer re-prompt first.

---

## Before starting

1. `opencode --version` — must be on `PATH`.
2. `opencode models | grep -i kimi` — confirm model id.
3. `opencode agent list` — usually `build` for implementation.

---

## Picking a model

- **UI (any substantial UI)** → **`opencode-go/kimi-k2.7-code`** always.
- **Non-UI implementation** → prefer `opencode-go/minimax-m2.7` unless user names another.
- **Read-only audit** → free/flash tier (e.g. `opencode/deepseek-v4-flash-free`).
- User names a model → use that exact id if listed; else say unavailable.

Related kimi ids seen: `opencode-go/kimi-k2.6`, `opencode-go/kimi-k2.7-code`, `opencode-go/kimi-k3`.
Default UI is **k2.7-code**, not k3 (k3 wandered more on polish tasks).

---

## Launch pattern (background + worktree)

```bash
# Prefer isolated worktree so live dev server is not half-broken mid-run
git worktree add -b kimi/<task-slug> /tmp/<repo>-kimi-<task> origin/main

PROMPT_FILE=/tmp/opencode/<task>-prompt.txt
LOG=/tmp/opencode/<task>-kimi.log
# write DIRECT task prompt into $PROMPT_FILE

nohup opencode run \
  --agent build \
  --model opencode-go/kimi-k2.7-code \
  --auto \
  --dir /tmp/<repo>-kimi-<task> \
  --title "<task-slug>" \
  "$(cat "$PROMPT_FILE")" \
  > "$LOG" 2>&1 &
echo "PID=$! LOG=$LOG"
sleep 3
head -25 "$LOG"   # must show: > build · kimi-k2.7-code  — not yargs help
ps -p $! -o pid,etime,cmd
```

**Smoke-check:** log has `> build · …`, process alive. If only help text → fix argv and relaunch.

**Do not** use `--prompt "$(cat <<'EOF' …)"` under `nohup` — dumps help and exits.
Always: write file → `"$(cat "$PROMPT_FILE")"` as **positional** message → `--dir` + `--title`.

### Monitor (don’t block forever)

```bash
while kill -0 <PID> 2>/dev/null; do
  sleep 30
  git -C /tmp/<repo>-kimi-<task> status --short | head
  tail -n 20 "$LOG"
done
```

Kill by **PID only** (`kill <PID>`). Avoid `pkill -f opencode` from a command line that
itself contains the string `opencode` (self-match can kill the wrapper).

---

## Worktree rules

1. `git worktree add -b kimi/<slug> <path> <base>` (usually latest `main` / task base).
2. Point `--dir` at that path.
3. After success + typecheck: review diff → merge/cherry-pick to main only when user wants.
4. `git worktree remove <path>` when done.
5. If an old kimi is stuck on a bad prompt: **kill PID → remove/recreate worktree → new direct prompt**.

Skip worktree only for pure read-only audits (no writes).

---

## After kimi finishes

1. `git -C <worktree> diff --stat` and spot-check key files.
2. Confirm product rules held (e.g. no password-on-invite reintroduced).
3. `cd web && npx tsc --noEmit` if kimi didn’t cleanly.
4. Merge/push only when user asks (or already said push).
5. Tell the user: branch, files changed, anything left for them.

---

## Lessons learned (keep updating this file)

- **Direct tasks only (2026-07):** User wants kimi given **direct task lists**, not
  explore-first briefs. Vague “implement invite UI / survey codebase” → stuck reading,
  no commits. Tight “edit these N files: do X/Y/Z” → actual diffs.
- **Never skip kimi for UI** to ship faster; do backend yourself, UI via kimi.
- **Kill stuck runs** that only Read/Explore for minutes; re-prompt shorter.
- Stale worktrees on old commits cause wrong code; always base on current main/task HEAD.
- Prompt must include product constraints (invite-only, no invent features) or kimi may
  restore password-on-create from older patterns in git history.
- Blocking foreground `opencode run` → treat empty result as unknown; check PID + diff.
- `--auto` required for unattended runs; only in trusted dirs.
- When this skill gains new facts, **update SKILL.md in the same turn**.
