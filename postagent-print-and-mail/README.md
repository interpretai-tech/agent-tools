# PostAgent — Print & Mail (Agent Skill)

An installable [agent skill](https://skills.sh) that lets your AI coding agent
**print and physically mail** a document (PDF, HTML, Markdown, text, DOCX, or
image) to a **US postal address** via the [PostAgent](https://postagent-api.interpretai.tech)
API. Each send is paid per-call in **USDC on Base** using the
[x402](https://www.x402.org/) payment protocol, then printed and delivered via
[Lob](https://www.lob.com/).

> Sending mail spends real money and is **irreversible** once submitted. The
> skill always quotes first and requires explicit user confirmation before
> paying.

## Install

**Claude Code** (global) — always pass `-a claude-code` so it lands in
`~/.claude/skills/`, the only place Claude Code reads skills from:

```bash
npx skills add interpretai-tech/agent-tools --skill postagent-print-and-mail -g -a claude-code -y
```

Then **restart Claude Code** — the skill list is read once at session start, so a
newly installed skill won't appear until you relaunch.

Other agents (auto-detected — installs to each agent's own skills dir, e.g.
`.agents/skills/` for Codex/Cursor/Gemini):

```bash
npx skills add interpretai-tech/agent-tools --skill postagent-print-and-mail -y
```

> Without `-a claude-code`, the CLI may install only to the generic
> `.agents/skills/` path, which **Claude Code does not read** — the skill will
> appear "installed" but never load.
>
> `-y` skips the interactive scope/confirmation prompt. Drop it if you prefer to
> confirm the scope and review before install.

## What it does

1. **Upload** a document (free) — finished letter or `{{field}}` mail-merge template.
2. **Quote** (free) — verifies US sender/recipient addresses and locks a USDC price for 15 minutes.
3. **Pay** — the agent's x402 wallet pays the quote's `paymentUrl`; PostAgent prints and mails the letter.
4. **Track** (free) — poll job status and USPS tracking.

Works in any agent via the **REST API** (plain HTTP, no setup) — and optionally
via the **PostAgent MCP server** (`https://postagent-api.interpretai.tech/mcp`)
when it's configured. See [`SKILL.md`](./SKILL.md) for the full instructions the
agent follows.

## Requirements

- A US sender and US recipient address (2-letter state code, 5-digit/ZIP+4).
- An x402-capable wallet with USDC on Base mainnet to pay per send.

## License

[MIT](./LICENSE)
