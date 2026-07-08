quento-skills

# Quento Skills Repo

Distribution repo for Quento agent skills.

## Key constraints

- Skills in `skills/` are the source of truth. Edit them directly.

- `README.md`, `install.md`, and `AGENTS.md` are safe to edit directly.

- Keep `skills/quento/SKILL.md` in sync with the MCP tools in the Quento app (`lib/mcp/tools/`). When a new tool is added to Quento, add it here too.

## Repo structure

```
skills/  
  quento/  
    SKILL.md        ← MCP skill for using Quento  
README.md           ← Short intro + npx install command  
install.md          ← Autonomous install agent prompt  
AGENTS.md           ← This file
```

## Adding a new tool

1. Check `lib/mcp/tools/` in the Quento repo for the tool's schema

2. Add it to the appropriate section in `skills/quento/SKILL.md`

3. Add it to the Quick Reference table

4. Add relevant triggers if it covers a new user intent

## Testing

Verify a skill works by driving the Quento MCP server directly over HTTP (JSON-RPC: `initialize` → `notifications/initialized` → `tools/call`). See the verification script in [install.md](install.md) (Step 4) or the troubleshooting notes in [skills/quento/SKILL.md](skills/quento/SKILL.md) under "MCP connection".

