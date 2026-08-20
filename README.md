# own-your-funnel

An agent skill that turns your AI coding assistant into a product analyst. It maps your funnel, tracks missing events, and generates reports from your database—without installing any new analytics SDKs.

## How it works

1. **No new vendors**: It uses your existing Google Analytics/Meta Pixel and your own database.
2. **Local execution**: Runs as a skill inside your AI agent. No SaaS accounts, no data leaves your machine (except to your AI provider).
3. **Read-only**: It queries aggregates only. It will never write to your DB or pull raw user rows into context.

## Requirements

- An AI agent that supports skill folders (Claude Code, Grok CLI, Cursor, Antigravity, etc.).
- An existing analytics setup and/or database in your codebase.

## Installation

Copy this folder into your agent's skills directory:

```bash
.claude/skills/own-your-funnel/
.grok/skills/own-your-funnel/
.cursor/skills/own-your-funnel/
```

## Usage

### 1. Setup

Ask your agent:
> "Read own-your-funnel/SKILL.md and run setup. I don't know what I should be tracking."

The agent will infer your architecture, locate your tracking wrapper, map your DB schema for revenue, and propose a funnel. **It will ask for your confirmation** before writing `funnel.yaml`.

### 2. Daily Operations

Once `funnel.yaml` is confirmed, use these commands:

- `/ask`: Ask a single question (e.g., "Why are users dropping off at checkout?"). The agent runs the query, shows the code, and answers.
- `/report`: Generates a periodic Markdown report comparing the last 7 days to the previous 7, detailing funnel drop-off, revenue, anomalies, and concrete actions.

## What it will NOT do

- Add new analytics SDKs.
- Write or modify your database.
- Send PII (emails/phones) to analytics vendors.
- Guess or invent metrics without querying the mapped database columns.

## License

MIT © 2026 Shaurya.
