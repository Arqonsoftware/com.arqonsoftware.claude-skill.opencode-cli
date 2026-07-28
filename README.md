# opencode-cli

A Claude Code / Grok skill that teaches the outer agent how to drive the
[opencode](https://opencode.ai) CLI — especially **kimi for all UI work** with
**direct, file-scoped task prompts** (not vague “explore the codebase” briefs).

## Install

As a plugin via a marketplace pointing at this repo, or by copying
`skills/opencode-cli/` into:

- `~/.grok/skills/` or `~/.claude/skills/` (user-level)
- `<repo>/.grok/skills/` or `<repo>/.claude/skills/` (project-level)

## Requirements

The `opencode` CLI must already be installed and on `PATH`. This skill does not install it.

## Usage

### UI (always kimi)

> “Polish the Team invite page” / any layout, form chrome, Tailwind, page UI

Outer agent must:

1. Find exact files and product rules itself.
2. Launch `opencode run --agent build --model opencode-go/kimi-k2.7-code --auto …`
   with a **direct** numbered edit list (edit file X — do Y).
3. **Not** implement UI itself; **not** tell kimi to explore first for known files.
4. Kill runs that only Read for minutes with zero writes; re-prompt shorter.

### Non-UI / general

> “Use opencode with minimax to fix X”

Outer agent picks model from `opencode models`, runs in a worktree when a live dev
server watches the repo, backgrounds the job, reviews the diff.

## Prompt rule of thumb

| Bad | Good |
|-----|------|
| “Implement invite UI; survey the codebase first” | “Edit TeamPage.tsx: remove password fields; CTA Invite” |
| “Make it professional” | “Match LoginPage Card spacing; button label Invite” |

Full template and launch pattern: `skills/opencode-cli/SKILL.md`.
