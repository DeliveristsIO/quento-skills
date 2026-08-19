quento-skills

# Quento Skills Repo

Distribution repo for Quento agent skills.

## Key constraints

- Skills in `skills/` are the source of truth. Edit them directly.

- `README.md`, `install.md`, and `AGENTS.md` are safe to edit directly.

- Keep `skills/quento/SKILL.md` in sync with the tools registered for the MCP channel in the Quento app's `lib/quento_ai/capability_registry.rb`. Tool implementations and schemas live in `lib/mcp/tools/`, but files in that directory are not necessarily exposed over MCP.

## Repo structure

```
skills/  
  quento/  
    SKILL.md        ← MCP skill for using Quento  
.claude-plugin/
  plugin.json       ← Claude Code plugin manifest + MCP connection
  marketplace.json  ← Claude Code marketplace catalog
README.md           ← Short intro + npx install command  
install.md          ← Autonomous install agent prompt  
AGENTS.md           ← This file
```

## Adding a new tool

1. Confirm the tool is registered with the `mcp` channel in the Quento app's `lib/quento_ai/capability_registry.rb`

2. Read its schema and behavior in the corresponding `lib/mcp/tools/` implementation

3. Add it to the appropriate tool table and workflow in `skills/quento/SKILL.md`

4. Add relevant triggers if it covers a new user intent, and update `README.md` or `install.md` when the public feature or setup flow changes

## Testing

Verify through an OAuth-capable MCP client: connect to `https://quento.app/mcp`, complete browser authorization, then call a read-only tool such as `list_invoices_tool`. Do not document or test API-key, bearer-token, tenant-subdomain, or raw `curl` authentication flows. See [install.md](install.md) and the troubleshooting notes in [skills/quento/SKILL.md](skills/quento/SKILL.md) under "MCP connection".
