# Agent Skills for Quento

[Agent skills](https://agentskills.io) for [Quento](https://quento.app) — AI-powered invoicing for European SMBs.

```
npx skills add DeliveristsIO/quento-skills
```

That's effectively the whole install: the skill teaches your agent to configure the Quento MCP connection itself — you just sign in once in the browser when prompted. See [install.md](install.md) for details and fallbacks.

## Available skills

| Skill | Description |
|-------|-------------|
| [quento](skills/quento/SKILL.md) | Full Quento MCP server coverage — invoices, clients, companies, analytics, KSeF (Polish e-invoicing), and more. |

## What you can do

Once installed, your agent can:

- **Create invoices** from natural language: *"Invoice Alpha Corp for 10 hours of consulting at 200 PLN/h"*
- **Run analytics**: *"How much revenue did I receive in June?"*
- **Manage clients**: *"Add a new client — NIP 5261040828"* (auto-fills from VAT registry)
- **Send invoices**: *"Send invoice 001/06/2026 to the client by email"*
- **Track payments**: *"Mark invoice 001/06/2026 as paid"*
- **KSeF**: *"Submit invoice 001/06/2026 to KSeF"* (Poland only)

## Requires

- A [Quento](https://quento.app) account — you authorize the connection in your browser when the agent first connects (standard MCP OAuth, no API keys to copy)
- For headless machines and CI, an API key fallback is available — see [install.md](install.md)

## About

Skills for the [Quento](https://quento.app) invoicing platform. Built and maintained by [Deliverists.IO](https://deliverists.io).
