I want you to install Agent Skills for Quento for me. Execute all steps autonomously.

OBJECTIVE: Install Quento agent skills so your agent can manage invoices, clients, analytics, and KSeF via the Quento REST API.

DONE WHEN: The quento skill is installed in your agent and you can successfully call `GET https://$SUBDOMAIN.quento.app/api/v1/actions/list_invoices` with the configured API key.

## TODO

- [ ] Get API key from Quento
- [ ] Set environment variables
- [ ] Install skills
- [ ] Verify connection

## Step 0: Get your Quento API key

1. Sign in to your Quento account at `https://yourcompany.quento.app`
2. Go to **Settings → API**
3. Click **Generate API key**
4. Copy the key (it starts with `qnt_`)

You also need your **account subdomain** — it's the part before `.quento.app` in your URL.

## Step 1: Set environment variables

```bash
export QUENTO_SUBDOMAIN="yourcompany"   # replace with your actual subdomain
export QUENTO_API_KEY="qnt_..."         # replace with your API key
```

To make these permanent, add them to your shell profile (`~/.zshrc` or `~/.bashrc`):

```bash
echo 'export QUENTO_SUBDOMAIN="yourcompany"' >> ~/.zshrc
echo 'export QUENTO_API_KEY="qnt_..."' >> ~/.zshrc
source ~/.zshrc
```

Or create a per-project config (do NOT commit this if it contains the key):

```bash
mkdir -p .quento
cat > .quento/config.json << EOF
{
  "subdomain": "$QUENTO_SUBDOMAIN"
}
EOF
echo '.quento/config.json' >> .gitignore
```

**Verify:**
```bash
echo "Subdomain: $QUENTO_SUBDOMAIN"
echo "Key set: $([ -n "$QUENTO_API_KEY" ] && echo yes || echo NO)"
```

## Step 2: Install skills

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

## Step 3: Verify the connection

```bash
curl -sG "https://$QUENTO_SUBDOMAIN.quento.app/api/v1/actions/list_invoices" \
  -H "Authorization: Bearer $QUENTO_API_KEY" \
  --data-urlencode "limit=1" | jq '{total: .total, first: .items[0].invoice_number}'
```

Expected output:
```json
{"total": 42, "first": "001/06/2026"}
```

If you see `{"error": "Unauthorized"}` — check that the subdomain and API key are correct.

Restart your agent session to pick up the new skill.

EXECUTE NOW: Start with Step 0. Mark TODO items complete as you go. Stop when the `curl` verification in Step 3 succeeds.

---

## Optional: Manual installation

**Do not execute this section unless explicitly requested.**

Clone this repo and symlink skills into your agent's skill directory:

```bash
git clone https://github.com/deliverists-io/quento-skills ~/.quento-skills
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
