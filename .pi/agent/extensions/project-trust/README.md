# Project Trust Extension

Intercepts the `project_trust` event to give you control over when pi trusts a project directory.

## How It Works

When pi starts in a directory with project-level configs (`.pi`, `AGENTS.md`, `CLAUDE.md`, or `.agents/skills`), it fires a `project_trust` event. This extension intercepts it and presents a selection dialog with these options:

| Option | Effect |
|--------|--------|
| **Trust and remember** | Trust this project permanently (`trusted: "yes", remember: true`) |
| **Trust with note and remember** | Trust permanently with an optional annotation |
| **Trust this session** | Trust only for the current session (`trusted: "yes"`) |
| **Do not trust this session** | Skip trust for this session (`trusted: "no"`) |
| **Let built-in prompt decide** | Pass through to pi's default trust flow |

The first handler that returns `yes` or `no` wins and suppresses the built-in trust prompt. Returning `undecided` defers to the next handler or pi's default flow.

## Files

- `index.ts` — Extension source

## Usage

This extension is auto-discovered from `~/.pi/agent/extensions/project-trust/`. No additional setup needed — it activates on next `/reload` or pi restart.

## Source

Based on the [project-trust.ts example](https://github.com/earendil-works/pi-mono/tree/main/packages/coding-agent/examples/extensions/project-trust.ts) from the pi monorepo.
