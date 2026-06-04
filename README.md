# InterpretAI Agent Skills

Installable [agent skills](https://skills.sh) from InterpretAI, usable with
Claude Code, Cursor, Codex, and other agents via the `npx skills` CLI.

## Skills

| Skill | Description |
| ----- | ----------- |
| [`postagent-ship-mail-and-packages`](./skills/postagent-ship-mail-and-packages) | Print and physically mail a document to a US address via the PostAgent API, paid per-send in USDC over x402. |

## Install

```bash
# List everything in this repo
npx skills add interpretai-tech/agent-skills --list

# Install a specific skill (optionally to a specific agent, globally)
npx skills add interpretai-tech/agent-skills --skill postagent-ship-mail-and-packages -g -a claude-code
```
