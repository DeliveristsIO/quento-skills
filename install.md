I want you to install Agent Skills for Quento for me. Execute all steps autonomously.

OBJECTIVE: Install Quento agent skills so your agent can manage invoices, clients, analytics, and KSeF via the Quento MCP server.

DONE WHEN: The quento skill is installed, the Quento MCP server is configured in the agent's config file, and a restart confirms the agent can call Quento tools (e.g. it lists real invoices when asked).

## TODO

- [ ] Get API key + subdomain from Quento
- [ ] Configure the Quento MCP server connection
- [ ] Install skill docs
- [ ] Restart and verify

## Step 0: Get your Quento API key

1. Register, or sign in to your existing Quento account, at `https://www.quento.app`

2. Click your **account email** in the top-right of the nav bar → **Integrations**

![Account dropdown → Integrations](skills/quento/images/integrations-menu.png)

3. Scroll down and expand **Advanced Integrations** → **Your Credentials**, then copy the **API Key** and the **Account Subdomain**

![Advanced Integrations → Your Credentials](skills/quento/images/api-key-credentials.png)

Anyone with the API key has full access to the account's data — never commit it or a screenshot of it. If it ever leaks, click **Regenerate Key** on that same screen.

## Step 1: Configure the Quento MCP server connection

Add this to your agent's MCP client config. For Claude Code, that's `~/.claude.json` under `mcpServers` (a project-level `.mcp.json` works too):

```json
{
  "mcpServers": {
    "quento": {
      "type": "http",
      "url": "https://yourcompany.quento.app/mcp",
      "headers": {
        "Authorization": "Bearer your-api-key-here"
      }
    }
  }
}
```

Replace `yourcompany` with your **Account Subdomain** and `your-api-key-here` with your **API Key**, both from Step 0.

This is a plain JSON file edit — no terminal, shell profile, or environment variable needed, so it works the same way on macOS, Linux, and Windows. This config file lives on your machine only; it's not part of any project repo, so pasting the key directly here is fine.

> **Prefer not to store the key in a plaintext config file?** Set it as an environment variable instead — `QUENTO_API_KEY` on macOS/Linux (`export QUENTO_API_KEY="..."` in `~/.zshrc` or `~/.bashrc`, then `source` it) or Windows (`setx QUENTO_API_KEY "..."` in PowerShell, then open a new terminal) — and reference it as `"Authorization": "Bearer ${QUENTO_API_KEY}"` instead of the literal key. `${VAR}` expansion support depends on your MCP client; Claude Code supports it. Skip this if you're not comfortable with shell configuration — pasting the key directly above is just as functional.

## Step 2: Install skill docs

```bash
npx skills add DeliveristsIO/quento-skills
```

This installs the Quento skill into your agent using the [Agent Skills](https://agentskills.io) standard. The installer auto-detects your agent (Claude Code, Cursor, Codex, VS Code, Goose, Amp, and others).

To install for a specific agent:

```bash
npx skills add DeliveristsIO/quento-skills -a claude-code
```

To install globally (available across all projects):

```bash
npx skills add DeliveristsIO/quento-skills -g
```

**Verify:**
```bash
npx skills list
# Expected: quento skill listed
```

## Step 3: Restart and verify

MCP servers connect when your agent starts, so restart your agent session now. Then ask it something like *"list my Quento invoices"* — if it responds with real data instead of an error, the connection works.

<details>
<summary>Advanced: verify the MCP server directly, without restarting</summary>

This drives the server over raw HTTP (JSON-RPC), useful for troubleshooting or for an autonomous agent that wants to confirm the key/subdomain are correct before a restart:

```bash
URL="https://yourcompany.quento.app/mcp"   # your subdomain from Step 0
SID=$(curl -sS -D - -o /dev/null -X POST "$URL" \
  -H "Authorization: Bearer $QUENTO_API_KEY" \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"install-check","version":"1.0"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')
curl -sS -o /dev/null -X POST "$URL" \
  -H "Authorization: Bearer $QUENTO_API_KEY" -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
curl -sS -X POST "$URL" \
  -H "Authorization: Bearer $QUENTO_API_KEY" -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | grep -o 'list_invoices_tool'
```

Expected output: `list_invoices_tool`. If the key or subdomain are wrong, you'll get an auth error instead. This requires `QUENTO_API_KEY` to be set as an environment variable (see the callout in Step 1) — or just substitute your literal key into the `-H "Authorization: Bearer ..."` lines above.

</details>

EXECUTE NOW: Start with Step 0. Mark TODO items complete as you go. Stop once Step 3's verification succeeds.

---

## Optional: Manual installation

**Do not execute this section unless explicitly requested.**

Clone this repo and symlink skills into your agent's skill directory:

```bash
git clone https://github.com/DeliveristsIO/quento-skills ~/.quento-skills
mkdir -p ~/.claude/skills
ln -sfn ~/.quento-skills/skills/quento ~/.claude/skills/quento
```

For per-project installation:

```bash
mkdir -p .claude/skills
ln -sfn ~/.quento-skills/skills/quento .claude/skills/quento
```

Update with:
```bash
cd ~/.quento-skills && git pull
```

Symlinks pick up changes immediately without reinstalling.
