---
name: quento
description: |
  Interact with Quento via its MCP server. Full coverage: invoices, clients, companies,
  products, bank accounts, analytics, and KSeF (Polish e-invoicing).
  Use for ANY invoicing question or action — creating invoices, checking revenue,
  managing clients, sending invoices, querying statistics, and KSeF submissions.
triggers:
  # Direct invocations
  - quento
  - /quento
  # Invoice actions
  - create invoice
  - new invoice
  - draft invoice
  - issue invoice
  - send invoice
  - mark paid
  - mark invoice paid
  - cancel invoice
  - correction invoice
  - invoice PDF
  - get invoice
  - list invoices
  - find invoice
  - show invoice
  - invoice status
  # Client actions
  - add client
  - new client
  - create client
  - update client
  - list clients
  - find client
  - show client
  # Company/seller actions
  - list companies
  - add company
  - company details
  - lookup NIP
  - lookup VAT
  - look up company
  # Analytics
  - revenue
  - earnings
  - statistics
  - stats
  - how much did I earn
  - overdue invoices
  - unpaid invoices
  - outstanding receivables
  # KSeF (Polish e-invoicing)
  - KSeF
  - ksef
  - submit to ksef
  - e-invoice
  - e-faktura
  - purchase invoice
  - payables
  # Products
  - product catalog
  - add product
  - list products
  # Bank accounts
  - bank account
  # Common questions
  - invoice quento
  - quento invoice
  - quento client
  - quento stats
invocable: true
argument-hint: "[action] [args...]"
---

# /quento — Quento via MCP

Quento is driven through its **MCP (Model Context Protocol) server**. Once connected, your agent calls tools directly — no CLI, no curl. This skill documents which tools exist, what they do, and how to chain them for common workflows.

## MCP connection

Quento exposes its MCP server at:

```
https://quento.app/mcp
```

**Authentication is standard MCP OAuth** — no API keys to copy. Add the server with no headers to your MCP client config (`~/.claude.json` for Claude Code — under `mcpServers`, not `~/.claude/mcp.json`), restart, and authenticate in the browser when prompted (Claude Code: `/mcp` → **quento** → **Authenticate**). Tokens are stored by the client and renew automatically.

```json
{
  "mcpServers": {
    "quento": {
      "type": "http",
      "url": "https://quento.app/mcp"
    }
  }
}
```

### If the Quento MCP tools are not available — set up the connection yourself

You (the agent reading this) can bootstrap the connection: if this skill triggered but the Quento tools are missing (no `list_invoices_tool` via tool search), the MCP server just isn't configured in your client yet. **Prefer this setup over the curl/API-key fallback even if a `QUENTO_API_KEY` happens to be in the environment** — an interactive user is better served by the native, browser-authorized connection (the env key may also point at a different environment than the user expects). Reserve curl for genuinely headless sessions or when the user explicitly asks. Do this:

1. **Add the server to your own client's MCP config** — you know your own config location and format. Claude Code: `claude mcp add --transport http --scope user quento https://quento.app/mcp`. Codex: `codex mcp add quento --url https://quento.app/mcp`. OpenCode: add `"quento": { "type": "remote", "url": "https://quento.app/mcp" }` under `"mcp"` in `opencode.json`. Other clients: add `https://quento.app/mcp` as a remote/HTTP MCP server in your equivalent config.
2. **Hand the browser step to the user** — authorization is a human-only step. Tell them to restart the session, then authenticate (Claude Code: `/mcp` → **quento** → **Authenticate**; Codex: `codex mcp login quento`; OpenCode: `opencode mcp auth quento`, or it prompts automatically on first use; other clients: their "needs login" prompt), signing in to Quento and clicking **Authorize**. It's once per machine.
3. **Verify after restart** by calling `list_invoices_tool` — real data means you're connected.

If you can't modify your own config, or your client doesn't support HTTP MCP with OAuth, use the `mcp-remote` stdio shim (`npx mcp-remote https://quento.app/mcp`) or the API key fallback below — full walkthrough in [install.md](../../install.md).

### Fallback: API key (headless machines, CI, curl scripting)

A per-account API key also works: account email dropdown → **Integrations** → **Advanced Integrations** → **Your Credentials** (screenshot walkthrough in [install.md](../../install.md)). API keys are only accepted on the tenant URL `https://{subdomain}.quento.app/mcp` (the tenant-neutral `quento.app/mcp` is OAuth-only) and go in an `"Authorization": "Bearer ..."` header — either the literal key, or `Bearer ${QUENTO_API_KEY}` expanded from your shell environment (Claude Code supports `${VAR}` expansion; export it before launching). Keep the key secure — anyone with it has full account access; use **Regenerate Key** immediately if it ever leaks.

**Tool names carry a `_tool` suffix** — e.g. the tool is `list_invoices_tool`, not `list_invoices`. The three exceptions are `create_client`, `get_client`, and `update_client`, which have no suffix. All tool references below use the real, callable names.

**If the tools don't show up** (`ToolSearch`/agent can't find `list_invoices_tool` etc.): MCP servers connect at Claude Code startup. On the OAuth path this usually means the server isn't authorized yet — run `/mcp` → **quento** → **Authenticate**. On the API-key fallback it almost always means `QUENTO_API_KEY` wasn't exported in the shell that launched the session — restart Claude Code from a shell where it's set. As a same-session workaround, you can drive the server directly over HTTP with `curl` — POST JSON-RPC to the `url` above with `Content-Type: application/json` and `Accept: application/json, text/event-stream` headers, calling `tools/call` directly.

**Session persistence:** the Quento MCP server runs in **stateless mode** (SEP-2567) — each POST is self-contained, no `Mcp-Session-Id` is issued or required, and `initialize` is an optional no-op. This is required for multi-worker deployments (puma `WEB_CONCURRENCY>1`) where a stateful session map held in a per-process class variable cannot be shared across workers. You can POST `tools/call` directly with just the `Authorization` header — no handshake, no session id. (Older server builds ran stateful mode; there the `Mcp-Session-Id` from `initialize` expired within seconds, so the handshake-per-call function below was needed. The function is forward-compatible and works against both modes — in stateless mode the `initialize` and `notifications/initialized` calls are harmless no-ops.)

A reusable shell function (one handshake per call — works against stateful and stateless servers alike):

```bash
QUENTO_URL="https://yourcompany.quento.app/mcp"
QUENTO_API_KEY="..." # or export it

# Simplest form for stateless servers (current): POST tools/call directly.
quento_call() {              # quento_call <tool_name> '<json arguments>'
  local tool="$1" args="${2:-{}}"
  curl -sS -X POST "$QUENTO_URL" \
    -H "Authorization: Bearer $QUENTO_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json, text/event-stream" \
    -H "MCP-Protocol-Version: 2024-11-05" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/call\",\"params\":{\"name\":\"$tool\",\"arguments\":$args}}"
}

# Robust form: full initialize → notifications/initialized → tools/call handshake
# per call. Required only for older stateful server builds; a no-op overhead on
# stateless ones. Use if quento_call ever returns "Session not found".
quento_mcp() {                # quento_mcp <tool_name> '<json arguments>'
  local tool="$1" args="${2:-{}}"
  local h sid
  h=$(curl -sS -i -X POST "$QUENTO_URL" \
    -H "Authorization: Bearer $QUENTO_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json, text/event-stream" \
    -H "MCP-Protocol-Version: 2024-11-05" \
    -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}')
  sid=$(printf '%s' "$h" | grep -i '^mcp-session-id' | awk '{print $2}' | tr -d '\r')
  curl -sS -X POST "$QUENTO_URL" \
    -H "Authorization: Bearer $QUENTO_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json, text/event-stream" \
    -H "MCP-Protocol-Version: 2024-11-05" \
    -H "Mcp-Session-Id: $sid" \
    -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' -o /dev/null
  curl -sS -X POST "$QUENTO_URL" \
    -H "Authorization: Bearer $QUENTO_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json, text/event-stream" \
    -H "MCP-Protocol-Version: 2024-11-05" \
    -H "Mcp-Session-Id: $sid" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"$tool\",\"arguments\":$args}}"
}

# Example:
# quento_call get_statistics_tool '{"period":"current_month"}'
# quento_call list_invoices_tool '{"from_date":"2026-06-01","to_date":"2026-06-30"}'
```

## Agent invariants

**Always follow these rules when using Quento tools:**

1. **Resolve IDs first** — use `list_clients_tool`, `list_companies_tool`, or `list_invoices_tool` to find IDs before calling mutation tools.
2. **VAT lookups first** — if the user provides a NIP or EU VAT number, ALWAYS call `lookup_company_tool` before `create_client`. It auto-fills name and address from the tax authority registry.
3. **Minimal params** — Quento infers payment_method, currency, bank account, and dates from company settings. Only pass what the user explicitly stated.
4. **Invoice state machine** — `draft → issue → issued → mark_paid → paid`. You cannot edit a non-draft invoice. `cancel` works from any state. `unmark_paid` (via change_invoice_status_tool) reverts a wrongly-paid invoice back to issued.
5. **KSeF is Poland-only** — KSeF tools only work for companies with `country: "PL"` and a configured KSeF token.
6. **Never estimate financial figures** — all amounts must come from tool results. If a tool returns no data, say so explicitly.

## Tools

### Invoices

| Tool | What it does |
|------|-------------|
| `list_invoices_tool` | List with filters: status, date range (by issue date), paid date range, client, search, currency |
| `get_invoice_tool` | Get one invoice by ID or invoice_number |
| `create_invoice_tool` | Create draft. Requires: client_id, items[]. Returns id and invoice_number. |
| `update_invoice_tool` | Update a draft. Use `replace_items: true` when correcting items to avoid duplicates. Supports `currency` (relabels amounts, re-picks bank account) — never cancel+recreate to change currency; that burns an invoice number. |
| `change_invoice_status_tool` | `action: "issue"` (draft→issued), `action: "cancel"`, or `action: "unmark_paid"` (paid→issued, undo a mistaken payment) |
| `mark_invoice_paid_tool` | Mark as paid. Optional: payment_date (default today). |
| `send_invoice_email_tool` | Email to client. Auto-issues drafts by default. |
| `get_invoice_pdf_link_tool` | Returns the invoice's PDF URL (requires the caller to be logged in to view; not a public/shareable link) |
| `create_correction_invoice_tool` | Create a correction of an issued invoice |

Note: `list_invoices_tool`'s `from_date`/`to_date` filter by **issue date**; use `paid_from`/`paid_to` to filter by payment date instead.

**Invoice items format:**
```
{description, quantity, unit_price, vat_rate, unit, product_id?}
unit_price: in invoice currency (not cents)
vat_rate: percentage — 23, 8, 5, 0 (PL); 19, 7, 0 (DE)
unit: h, pcs, szt, kg, m, month, etc.
```

**Status values:** `draft` | `issued` | `sent` | `overdue` | `paid` | `cancelled`
Note: issued, sent, and overdue are all unpaid.

---

### Clients

| Tool | What it does |
|------|-------------|
| `list_clients_tool` | Search by name, email, or NIP. Returns id, name, nip, email. |
| `get_client` | Get one client by ID |
| `create_client` | Create new client. If user provides NIP, call `lookup_company_tool` first. |
| `update_client` | Update client details by ID |

Note: there is no `delete_client` tool on the live server — clients cannot be deleted via MCP.

---

### Companies (sellers)

| Tool | What it does |
|------|-------------|
| `list_companies_tool` | List seller companies in the account |
| `get_company_tool` | Get one by ID, name, or NIP |
| `create_company_tool` | Create a new seller company |
| `update_company_tool` | Update company details |
| `lookup_company_tool` | **Look up NIP/EU VAT in registry** — returns name and address. Call before create_client when user provides a tax ID. |

---

### Products

| Tool | What it does |
|------|-------------|
| `list_products_tool` | List product catalog |
| `get_product_tool` | Get one product by ID |
| `create_product_tool` | Add product: name, unit_price, vat_rate, unit |
| `update_product_tool` | Update product |

Note: there is no `delete_product` tool on the live server.

---

### Bank accounts

| Tool | What it does |
|------|-------------|
| `list_bank_accounts_tool` | List by company, currency, active status |
| `create_bank_account_tool` | Add bank account to a company |
| `update_bank_account_tool` | Update account details |

Note: there is no `get_bank_account` or `delete_bank_account` tool on the live server — use `list_bank_accounts_tool` to look one up.

---

### Work journal (time tracking)

Feature-flagged per account (`work_logs`) — if these tools respond "not enabled", the account has no access yet.

| Tool | What it does |
|------|-------------|
| `list_work_logs_tool` | List entries. Filters: client_id, billing_status (`unbilled`/`billed`), date_from, date_to. Returns total hours in filter. |
| `create_work_log_tool` | Log work: client_id, summary, work_date (default today), duration_minutes, project_name, tasks[]. |
| `update_work_log_tool` | Update any field of an entry by ID, incl. billing_status. |
| `summarize_unbilled_work_tool` | Unbilled time per client — entry counts, hours, amount when the client has an hourly_rate, and the entry IDs. |
| `create_invoice_draft_from_work_logs_tool` | Turn explicit `work_log_ids` (one client, all unbilled, all with duration) into a **draft** invoice with a single aggregated hours line. Marks the entries billed. |

Rules:
- **Never guess duration.** If the user didn't say how long the work took, omit `duration_minutes` and ask; entries without duration cannot be invoiced.
- `create_invoice_draft_from_work_logs_tool` requires explicit entry IDs (get them from `summarize_unbilled_work_tool`) and only ever creates a *draft* — issuing/sending stays with the user.
- Deleting the draft invoice releases the entries back to unbilled.
- If the client has an `hourly_rate` set, the draft is priced `hours × rate`; otherwise unit price is 0 and the user edits the draft.

---

### Analytics

| Tool | What it does |
|------|-------------|
| `get_statistics_tool` | Revenue (by payment date), outstanding receivables, invoice counts, top clients |

**Periods:** `current_month` (default) | `last_month` | `current_quarter` | `last_quarter` | `current_year` | `last_year` | `all_time`

Or pass `from_date` / `to_date` for a custom range.

Revenue = money actually received (by `paid_at`), not invoices issued.
Each currency is reported separately — amounts are never summed across currencies.

---

### KSeF — Polish national e-invoicing

| Tool | What it does |
|------|-------------|
| `submit_invoice_to_ksef_tool` | Submit an issued invoice to KSeF. Company must have KSeF credentials configured. |
| `get_ksef_status_tool` | Check acceptance status for a submitted invoice |
| `list_ksef_submissions_tool` | List recent submissions, filter by status |
| `list_ksef_payables_tool` | List incoming purchase invoices received via KSeF |

Note: there is no `get_ksef_upo` tool on the live server — the official receipt (UPO) isn't retrievable via MCP; check `get_ksef_status_tool` or the Quento web app instead.

---

## Common workflows

### Create and issue an invoice

```
1. list_clients_tool(search: "Alpha Corp") → get client_id
   (if not found: lookup_company_tool(tax_id: NIP) → create_client → client_id)
   (if ambiguous — multiple matches — ask the user which client before proceeding)

2. create_invoice_tool(
     client_id: 42,
     items: [
       {description: "Consulting", quantity: 10, unit_price: 200, vat_rate: 23, unit: "h"}
     ]
   )
   → returns {id: 101, invoice_number: "001/06/2026", total: "2460.00 PLN"}

3. change_invoice_status_tool(id: 101, action: "issue")

4. send_invoice_email_tool(id: 101)
```

### Check this month's revenue

```
get_statistics_tool(period: "current_month")
→ {revenue: "18 450 PLN", outstanding: "6 200 PLN", period: "Czerwiec 2026"}
```

### Mark an invoice as paid

```
mark_invoice_paid_tool(invoice_number: "001/06/2026", payment_date: "2026-06-20")
```

**Marking paid is a state change with side effects** — it sets the payment date and records a payment entry. Phrases that mention a payment method are ambiguous: *"this invoice was paid in cash"* might mean "record the payment" (mark_invoice_paid_tool) or only "the payment method should be cash" (update_invoice_tool `payment_method`, drafts only). When the user's phrasing bundles a payment method with "paid" — or when marking paid wasn't the explicit request — confirm which they mean before calling mark_invoice_paid_tool. Changing just the payment method never requires a status change. If an invoice was marked paid by mistake, revert it with `change_invoice_status_tool(action: "unmark_paid")` — clears the payment date and removes the recorded payment entries.

### Fix items on a draft invoice (avoid duplicates)

```
update_invoice_tool(
  id: 101,
  replace_items: true,        ← ALWAYS use this when correcting items
  items: [all items in final form]
)
```

Without `replace_items: true`, the tool matches by description — if you renamed an item it adds a duplicate instead of replacing.

### Add a client with NIP (Polish VAT)

```
1. lookup_company_tool(tax_id: "5261040828")
   → {name: "PEKAO S.A.", address: "ul. Grzybowska 53/57", city: "Warszawa", ...}

2. create_client(name: "PEKAO S.A.", nip: "5261040828", country: "PL")
```

### Submit to KSeF

```
1. Verify invoice is issued (not draft)
2. submit_invoice_to_ksef_tool(invoice_number: "001/06/2026")
3. get_ksef_status_tool(invoice_number: "001/06/2026")
   → status: "accepted", reference_number: "8992520556-20260622-..."
```

---

### Log work and invoice it

```
1. "Add 2 hours for Kwiaciarnia Aga: Brevo and DNS setup"
   → list_clients_tool(search: "Kwiaciarnia Aga") → client_id
   → create_work_log_tool(client_id, summary: "Brevo and DNS setup", duration_minutes: 120)
   (no duration stated? create without duration_minutes, then ASK the user and
    update_work_log_tool — never guess time)

2. "How much unbilled work for Aga in August?"
   → summarize_unbilled_work_tool(client_id, date_from: "2026-08-01", date_to: "2026-08-31")
   → reports hours, amount (if hourly_rate set), and entry IDs

3. "Draft an invoice from it"
   → create_invoice_draft_from_work_logs_tool(work_log_ids: [12, 15, 18])
   → DRAFT invoice with one "hours × rate" line; entries flip to billed.
   The user reviews and issues it themselves (change_invoice_status_tool or web UI).
```

---

## Gotchas

- **Tool names have a `_tool` suffix** — the real callable name is `list_invoices_tool`, not `list_invoices`. Only `create_client`, `get_client`, `update_client` are unsuffixed. Calling the unsuffixed form for any other tool will fail to resolve.
- **Tools not appearing at all** — if a search for these tools comes up empty even after getting the suffix right, the MCP connection likely never came up (commonly because `QUENTO_API_KEY` wasn't set in the environment Claude Code was launched from). Restart the session from a shell with the key exported. See the workaround in "MCP connection" above for driving the server directly via HTTP in the meantime.
- **Stateless server (no session id needed)** — the Quento MCP server runs in stateless mode: each `POST /mcp` is self-contained, no `Mcp-Session-Id` is issued, and `initialize` is optional. You can `POST` a `tools/call` directly with just the `Authorization` header. If you ever see `{"error":{"code":-32600,"message":"Session not found"}}`, you're talking to an older stateful build — use the `quento_mcp` handshake-per-call function in "MCP connection" above.
- **`replace_items: true`** — always use this in `update_invoice_tool` when correcting items. Without it, a renamed item (e.g. "Farba 1L" → "Farba 10L") is added as a new duplicate instead of replacing the original.
- **Revenue vs issued** — `get_statistics_tool` revenue is always by `paid_at` (payment date), not issue date. "How much did I earn in June?" means paid in June, not invoiced in June.
- **`list_invoices_tool` date filters** — `from_date`/`to_date` filter by issue date; `paid_from`/`paid_to` filter by payment date. Picking the wrong pair silently returns the wrong invoices instead of erroring.
- **Multi-currency** — statistics never mix currencies. If an account has PLN and EUR invoices, both are returned separately.
- **KSeF Poland-only** — KSeF tools silently error or return empty if the company's country isn't PL. Check `list_companies_tool` to confirm country.
- **Draft-only edits** — `update_invoice_tool` fails on issued/paid invoices. For issued invoices, use `create_correction_invoice_tool` instead.
- **`auto_issue` in send_invoice_email_tool** — defaults to `true`, so calling it on a draft automatically issues it first. Pass `auto_issue: false` to disable.
- **`list_ksef_payables_tool` access** — per-user KSeF company access controls apply. If the tool returns empty, the authenticated user may not have `ksef_access` for those companies.
- **PDF links require login** — `get_invoice_pdf_link_tool` returns a URL under the company's own Quento domain, not a signed/temporary public link. The viewer must be logged in to Quento to open it.
- **No delete tools** — there is no `delete_client`, `delete_product`, or `delete_bank_account` on the live server, and no `get_bank_account`/`get_ksef_upo` either, despite what older docs may say. Don't assume a tool exists — check the live `tools/list` if in doubt.
