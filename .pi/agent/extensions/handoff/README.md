# Handoff Extension

Transfer conversation context to a new focused session without compaction loss.

## Problem

Compaction is lossy — it summarizes older messages, discarding details. When you want to shift focus (e.g., "now implement this for teams") but keep the full context accessible, you need a fresh session with a well-crafted prompt.

## How It Works

`/handoff <goal for new thread>` does three things:

1. **Extracts context** — Reads the current session branch including compaction summaries and post-compaction entries
2. **Generates a prompt** — Uses your current model to produce a self-contained prompt covering decisions made, files touched, and the next task
3. **Opens a new session** — Creates a fresh session linked to the current one as its parent, with the generated prompt in the editor for review

## Usage

```
/handoff now implement this for teams as well
/handoff execute phase one of the plan
/handoff check other places that need this fix
```

The generated prompt appears in the editor so you can tweak it before submitting. Press Esc to cancel at any point.

## Files

- `index.ts` — Extension source

## Requirements

- Interactive (TUI) mode
- An active model with API key configured
- A non-empty conversation branch

## Source

Based on the [handoff.ts example](https://github.com/earendil-works/pi-mono/tree/main/packages/coding-agent/examples/extensions/handoff.ts) from the pi monorepo.
