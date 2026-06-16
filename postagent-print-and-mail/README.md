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

```bash
npx skills add interpretai-tech/agent-tools --skill postagent-print-and-mail -y
```

Or install for a specific agent (e.g. Claude Code, globally):

```bash
npx skills add interpretai-tech/agent-tools --skill postagent-print-and-mail -g -a claude-code -y
```

> `-y` skips the interactive scope/confirmation prompt so the install completes
> in one shot. Drop it if you prefer to confirm the scope and review before
> install.

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
