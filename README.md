# Agent Skills for Quento

[Agent skills](https://agentskills.io) for [Quento](https://quento.app) — AI-powered invoicing for European SMBs.

```
npx skills add DeliveristsIO/quento-skills
```

That's effectively the whole install: the installer auto-detects supported agents, and the skill teaches them to configure the Quento MCP connection. You only need to complete browser authorization when prompted. The optional `-a` flag restricts installation to one agent; it is not required. See [install.md](install.md) for details.

## See it in action

[![Watch Quento Agent Skills working in Codex](https://img.youtube.com/vi/KNJRGLmh1oc/maxresdefault.jpg)](https://youtu.be/KNJRGLmh1oc)

In this short demo, Codex uses the Quento skill to list invoices and start a new invoice directly from the terminal.

### Claude Code plugin

In Claude Code you can install this as a plugin instead — it bundles the skill *and* the Quento MCP server config in one step:

```
/plugin marketplace add DeliveristsIO/quento-skills
/plugin install quento@deliverists
```

Then restart Claude Code, run `/mcp`, select **quento**, and authenticate in the browser. No API keys — standard MCP OAuth.

## Available skills

| Skill | Description |
|-------|-------------|
| [quento](skills/quento/SKILL.md) | Quento MCP workflows for invoices, clients, companies, analytics, KSeF (Polish e-invoicing), and more. |

## What you can do

Once installed, your agent can:

- **Create invoices** from natural language: *"Invoice Alpha Corp for 10 hours of consulting at 200 PLN/h"*
- **Run analytics**: *"How much revenue did I receive in June?"*
- **Manage clients**: *"Add a new client — NIP 5261040828"* (auto-fills from VAT registry)
- **Send invoices**: *"Send invoice 001/06/2026 to the client by email"*
- **Track payments**: *"Mark invoice 001/06/2026 as paid"*
- **KSeF**: *"Submit invoice 001/06/2026 to KSeF"* (Poland only)

## Requires

- A [Quento](https://quento.app) account
- An MCP client that supports OAuth, with a browser available to complete authorization

## About

Skills for the [Quento](https://quento.app) invoicing platform. Built and maintained by [Deliverists.IO](https://deliverists.io).
