I want you to install Agent Skills for Quento for me. Execute all steps autonomously.

OBJECTIVE: Install Quento agent skills so your agent can manage invoices, clients, analytics, and KSeF via the Quento MCP server.

DONE WHEN: The quento skill is installed in your agent, the Quento MCP server is configured, and a `tools/list` call against it returns tools like `list_invoices_tool`.

## TODO

- [ ] Get API key + subdomain from Quento
- [ ] Set the QUENTO_API_KEY environment variable
- [ ] Configure the Quento MCP server connection
- [ ] Install skill docs
- [ ] Verify connection

## Step 0: Get your Quento API key

1. Register, or sign in to your existing Quento account, at `https://www.quento.app`

2. Click your **account email** in the top-right of the nav bar → **Integrations**

![Account dropdown → Integrations](skills/quento/images/integrations-menu.png)

3. Scroll down and expand **Advanced Integrations** → **Your Credentials**, then copy the **API Key**

![Advanced Integrations → Your Credentials](skills/quento/images/api-key-credentials.png)

You also need your **Account Subdomain**, shown right below the API Key on the same screen — it's the part before `.quento.app` in your URL.

Anyone with the API key has full access to the account's data — never commit it or a screenshot of it. If it ever leaks, click **Regenerate Key** on that same screen.

## Step 1: Set the QUENTO_API_KEY environment variable

```bash
export QUENTO_API_KEY="..."   # copied from Advanced Integrations -> Your Credentials -> API Key in Step 0
```

To make this permanent, add it to your shell profile (`~/.zshrc` or `~/.bashrc`):

```bash
echo 'export QUENTO_API_KEY="..."' >> ~/.zshrc
source ~/.zshrc
```

**Verify:**
```bash
echo "Key set: $([ -n "$QUENTO_API_KEY" ] && echo yes || echo NO)"
```

## Step 2: Configure the Quento MCP server connection

Add Quento's MCP server to your agent's MCP client config. For Claude Code, that's `~/.claude.json` under `mcpServers` (project-level `.mcp.json` works too):

```json
{
  "mcpServers": {
    "quento": {
      "type": "http",
      "url": "https://yourcompany.quento.app/mcp",
      "headers": {
        "Authorization": "Bearer ${QUENTO_API_KEY}"
      }
    }
  }
}
```

Replace `yourcompany` with the **Account Subdomain** you copied in Step 0. `${QUENTO_API_KEY}` is expanded from the environment variable you just set — don't hardcode the key in this file.

## Step 3: Install skill docs

```bash
npx skills add deliverists-io/quento-skills
```

This installs the Quento skill into your agent using the [Agent Skills](https://agentskills.io) standard. The installer auto-detects your agent (Claude Code, Cursor, Codex, VS Code, Goose, Amp, and others).

To install for a specific agent:

```bash
npx skills add deliverists-io/quento-skills -a claude-code
```

To install globally (available across all projects):

```bash
npx skills add deliverists-io/quento-skills -g
```

**Verify:**
```bash
npx skills list
# Expected: quento skill listed
```

## Step 4: Verify the connection

MCP servers connect when your agent starts, so restarting is the normal way to pick up the new server — but you can verify the server and key work right now, over raw HTTP, without waiting for a restart:

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

Expected output: `list_invoices_tool`

If the key or subdomain are wrong, you'll get an auth error instead of that tool name — double check both against Step 0.

Restart your agent session to pick up the new MCP server.

EXECUTE NOW: Start with Step 0. Mark TODO items complete as you go. Stop when the `tools/list` check in Step 4 returns `list_invoices_tool`.


## Optional: Manual installation

**Do not execute this section unless explicitly requested.**

Clone this repo and symlink skills into your agent's skill directory:

```
git clone https://github.com/deliverists-io/quento-skills ~/.quento-skills  
mkdir -p ~/.claude/skills  
ln -sfn ~/.quento-skills/skills/quento ~/.claude/skills/quento
```

For per-project installation:

```
mkdir -p .claude/skills  
ln -sfn ~/.quento-skills/skills/quento .claude/skills/quento
```

Update with:

```
cd ~/.quento-skills && git pull
```

Symlinks pick up changes immediately without reinstalling.

