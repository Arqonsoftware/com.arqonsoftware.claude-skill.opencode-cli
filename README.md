# opencode-cli

A Claude Code skill/plugin that teaches Claude how to delegate coding tasks to the
[opencode](https://opencode.ai) CLI — including how to pick a model, how to get it to use its
own read-only subagents (e.g. `explore`) for research before implementing, and how to run it
in the background so a long task doesn't block the conversation.

## Install

As a plugin, via a marketplace pointing at this repo, or by copying `skills/opencode-cli/`
into your `~/.claude/skills/` (user-level) or `<repo>/.claude/skills/` (project-level)
directory.

## Requirements

The `opencode` CLI must already be installed and on `PATH`. This skill does not install it.

## Usage

Just ask Claude Code to use opencode for a task, e.g.:

> "Use opencode with the deepseek v4 pro model to add X, using its explore subagent to
> research the codebase first."

Claude will discover available models/agents on your install (`opencode models`,
`opencode agent list`) rather than assuming fixed names, then run it in the background and
report back.
