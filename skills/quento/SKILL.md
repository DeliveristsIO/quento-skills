---
name: quento
description: |
  Interact with Quento via its MCP server: invoices, clients, companies, products,
  bank accounts, work logs, analytics, and KSeF (Polish e-invoicing). Use for ANY
  invoicing question or action — creating, issuing, sending, correcting or cancelling
  invoices, marking invoices paid, invoice PDFs, adding or looking up clients by
  NIP/VAT, revenue and overdue/unpaid statistics, logging work time and billing it,
  KSeF submissions of sales invoices, and KSeF purchase (cost) invoices / payables
  received from suppliers. Triggers on "quento", "/quento", "invoice", "faktura",
  "e-faktura", "KSeF", "NIP", "revenue", "how much did I earn", "faktury kosztowe",
  "faktury zakupowe", "zobowiązania", "do zapłaty", "purchase invoices", "payables".
compatibility: Requires an MCP client with OAuth support and a browser to authorize https://quento.app/mcp
metadata:
  author: Deliverists.IO
  homepage: https://quento.app
---

# /quento — Quento via MCP

Quento is driven through its **MCP (Model Context Protocol) server**. Once connected, your agent calls tools directly — no CLI, no curl. This skill documents which tools exist, what they do, and how to chain them for common workflows.

## MCP connection

Quento exposes its MCP server at:

```
https://quento.app/mcp
```

**Authentication is standard MCP OAuth.** Add the server with no headers to your MCP client config (`~/.claude.json` for Claude Code — under `mcpServers`, not `~/.claude/mcp.json`), restart, and authenticate in the browser when prompted (Claude Code: `/mcp` → **quento** → **Authenticate**). The client stores and renews the OAuth tokens. Do not configure API keys, bearer-token headers, or tenant-specific MCP URLs.

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

You (the agent reading this) can bootstrap the connection: if this skill triggered but the Quento tools are missing (no `list_invoices_tool` via tool search), the MCP server is not configured or authorized in your client yet. Do this:

1. **Add the server to your own client's MCP config** — you know your own config location and format. Claude Code: `claude mcp add --transport http --scope user quento https://quento.app/mcp`. Codex: `codex mcp add quento --url https://quento.app/mcp`. OpenCode: add `"quento": { "type": "remote", "url": "https://quento.app/mcp" }` under `"mcp"` in `opencode.json`. Other clients: add `https://quento.app/mcp` as a remote/HTTP MCP server in your equivalent config.
2. **Hand the browser step to the user** — authorization is a human-only step. Tell them to restart the session, then authenticate (Claude Code: `/mcp` → **quento** → **Authenticate**; Codex: `codex mcp login quento`; OpenCode: `opencode mcp auth quento`, or it prompts automatically on first use; other clients: their "needs login" prompt), signing in to Quento and clicking **Authorize**. It's once per machine.
3. **Verify after restart** by calling `list_invoices_tool` — real data means you're connected.

If your client only supports stdio MCP servers, use the `mcp-remote` shim (`npx mcp-remote https://quento.app/mcp`) to proxy stdio to HTTP and complete the same browser OAuth flow. If the client cannot complete MCP OAuth, tell the user that it is unsupported; do not fall back to an API key or raw HTTP calls. See [install.md](https://github.com/DeliveristsIO/quento-skills/blob/main/install.md).

**Tool names carry a `_tool` suffix** — e.g. the tool is `list_invoices_tool`, not `list_invoices`. The three exceptions are `create_client`, `get_client`, and `update_client`, which have no suffix. All tool references below use the real, callable names.

**If the tools don't show up** (`ToolSearch`/agent can't find `list_invoices_tool` etc.): restart the client after adding the server, then open its MCP settings and complete OAuth authorization. Claude Code: `/mcp` → **quento** → **Authenticate**. Codex: `codex mcp login quento`, then inspect `/mcp` after restarting if needed. Do not work around a missing OAuth connection with `curl` or manually supplied credentials.

## Agent invariants

**Always follow these rules when using Quento tools:**

1. **Resolve IDs first** — use `list_clients_tool`, `list_companies_tool`, or `list_invoices_tool` to find IDs before calling mutation tools.
2. **VAT lookups first** — if the user provides a NIP or EU VAT number, ALWAYS call `lookup_company_tool` before `create_client`. It auto-fills name and address from the tax authority registry.
3. **Minimal params** — Quento infers payment_method, currency, bank account, and dates from company settings. Only pass what the user explicitly stated.
4. **Detect currency mismatches** — after resolving the seller company, compare its default currency with any currency the user attached to item prices or totals. If they differ, fetch and present the current exchange rate before creating or updating the invoice. Follow the "Foreign-currency amount for a PLN company" workflow below; never silently convert or relabel amounts.
5. **Invoice state machine** — `draft → issue → issued → mark_paid → paid`. You cannot edit a non-draft invoice. `cancel` works from any state. `unmark_paid` (via change_invoice_status_tool) reverts a wrongly-paid invoice back to issued.
6. **KSeF is Poland-only** — KSeF tools only work for companies with `country: "PL"` and a configured KSeF token.
7. **Never estimate financial figures** — all amounts must come from tool results or an explicitly cited exchange-rate source. If a tool or rate source returns no data, say so explicitly.
8. **Bill work-log entries with `create_invoice_draft_from_work_logs_tool`** — never `create_invoice_tool`. Only the dedicated tool links the invoice to the entries and marks them billed.

## Tools

### Invoices

| Tool | What it does |
|------|-------------|
| `list_invoices_tool` | List with filters: status, date range (by issue date), paid date range, client, search, currency |
| `get_invoice_tool` | Get one invoice by ID or invoice_number |
| `create_invoice_tool` | Create draft. Requires: items[]. client_id is optional — a draft can be created without a client and one added later via update_invoice_tool, but a client is required before issuing. Returns id and invoice_number. |
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

Feature-flagged per account (`work_logs`). If these tools answer "The work journal feature is not enabled for this account", tell the user to enable it in Quento; do not fall back to `create_invoice_tool` for journal entries.

| Tool | What it does |
|------|-------------|
| `create_work_log_tool` | Log time spent for a client: client_id, summary, optional work_date (default today), duration_minutes, project_name, tasks[]. Omit duration when the user didn't state it — never guess. |
| `list_work_logs_tool` | List entries; filter by client_id, billing_status (`unbilled` \| `billed`), date_from/date_to, limit (default 20, max 100). Returns total hours in filter. |
| `update_work_log_tool` | Update an entry by id: summary, date, duration, project, client, tasks (replaces the list), billing_status |
| `summarize_unbilled_work_tool` | Unbilled time per client: entry count, total hours, amount when the client has an hourly rate, and the entry IDs |
| `create_invoice_draft_from_work_logs_tool` | Create a DRAFT invoice from explicit `work_log_ids` of one client (all unbilled). Task items become one line each; plain entries aggregate into a single hours line and must have a duration. Links entries to the invoice and marks them billed. Never issues or sends. |

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
| `list_ksef_payables_tool` | List purchase (cost) invoices received from suppliers via KSeF — faktury kosztowe/zakupowe. Read-only. Params: `company_id`, `status` (to_pay, needs_review, overdue, paid), `search` (supplier, NIP, invoice/KSeF number, payment title), `payment_details` (`ready` \| `missing`), `limit` (default 10, max 50), `sort` (`issue_date_desc` default, `issue_date_asc`, `due_date_asc`, `due_date_desc`, `amount_desc`, `amount_asc`). Returns supplier, invoice number, gross amount, due date, status, payment readiness, plus a gross total per currency across all matches. |

Purchase invoices (faktury kosztowe/zakupowe) are NOT in `list_invoices_tool` or `get_statistics_tool`, which cover sales invoices only. For any question about cost invoices, what to pay, or supplier totals, call `list_ksef_payables_tool` and use only the amounts it returns. The tool cannot start bank payments; payment preparation and QR are in the Quento web app.

Note: if `get_ksef_upo_tool` is missing from the live `tools/list`, the official receipt (UPO) isn't retrievable via MCP; check `get_ksef_status_tool` or the Quento web app instead.

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

### Foreign-currency amount for a PLN company

When the resolved seller company's default currency is PLN but the user supplies an amount in USD (or another foreign currency):

```
1. Retrieve the latest published exchange rate from a live, authoritative source.
   For PLN pairs, prefer the National Bank of Poland (NBP) average exchange-rate table.

2. Before any invoice mutation, tell the user:
   - the rate and direction (for example, "1 USD = 3.91 PLN")
   - the rate's effective date and source
   - the converted amount, if useful
   - that a current informational rate may differ from the legally required tax/accounting rate

3. If the user has not specified the invoice currency, ask whether to:
   a. create the invoice in USD and keep the entered numerical prices unchanged, or
   b. create it in PLN and convert the prices using the displayed rate.

4. Create or update the draft only after the intended invoice currency is clear.
   Pass `currency: "USD"` when keeping the invoice in USD. When converting to PLN,
   pass the confirmed converted unit prices; changing `currency` alone does not convert values.
```

Always retrieve a fresh rate; do not use model memory or an uncited rate. Never silently convert, and never describe `update_invoice_tool(currency: ...)` as conversion: it only relabels the existing amounts and re-picks the matching bank account. If the user requests a statutory VAT/accounting conversion, do not assume the current rate applies—establish the relevant transaction/tax date and retrieve the rate required for that date.

### Bill logged work

```
1. summarize_unbilled_work_tool(client_id: 42)          → hours and amount pending
2. list_work_logs_tool(client_id: 42, billing_status: "unbilled") → entry ids
3. create_invoice_draft_from_work_logs_tool(work_log_ids: [7, 8, 9])
   → draft invoice; entries are now billed and linked
4. change_invoice_status_tool(id: ..., action: "issue") only when the user asks to issue
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

### Purchase invoices (faktury kosztowe) to pay

```
1. list_companies_tool                               → confirm country is PL
2. list_ksef_payables_tool(status: "overdue")        → late supplier invoices
3. list_ksef_payables_tool(sort: "due_date_asc", status: "to_pay") → what to pay first
4. list_ksef_payables_tool(payment_details: "missing") → invoices needing review
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
- **Tools not appearing at all** — restart the client after adding the server and complete browser authorization in its MCP settings. If OAuth cannot be completed, the client is not supported; do not substitute API keys or raw HTTP calls.
- **`replace_items: true`** — always use this in `update_invoice_tool` when correcting items. Without it, a renamed item (e.g. "Farba 1L" → "Farba 10L") is added as a new duplicate instead of replacing the original.
- **Revenue vs issued** — `get_statistics_tool` revenue is always by `paid_at` (payment date), not issue date. "How much did I earn in June?" means paid in June, not invoiced in June.
- **`list_invoices_tool` date filters** — `from_date`/`to_date` filter by issue date; `paid_from`/`paid_to` filter by payment date. Picking the wrong pair silently returns the wrong invoices instead of erroring.
- **Multi-currency** — statistics never mix currencies. If an account has PLN and EUR invoices, both are returned separately.
- **Currency changes do not convert amounts** — `update_invoice_tool(currency: ...)` only relabels the existing numbers and re-picks a bank account. For a company-currency/user-input mismatch, show a fresh sourced rate and establish the intended invoice currency before mutating the draft.
- **KSeF Poland-only** — KSeF tools silently error or return empty if the company's country isn't PL. Check `list_companies_tool` to confirm country.
- **Draft-only edits** — `update_invoice_tool` fails on issued/paid invoices. For issued invoices, use `create_correction_invoice_tool` instead.
- **`auto_issue` in send_invoice_email_tool** — defaults to `true`, so calling it on a draft automatically issues it first. Pass `auto_issue: false` to disable.
- **`list_ksef_payables_tool` access** — per-user KSeF company access controls apply. If the tool returns empty, the authenticated user may not have been granted KSeF payables access for those companies (granted in Quento settings); full-account users and API keys see all PL companies.
- **PDF links require login** — `get_invoice_pdf_link_tool` returns a URL under the company's own Quento domain, not a signed/temporary public link. The viewer must be logged in to Quento to open it.
- **Work-log billing** — `create_invoice_tool` knows nothing about journal entries and silently leaves them unbilled. Use `create_invoice_draft_from_work_logs_tool` for anything logged via the work journal.
- **No delete tools** — there is no `delete_client`, `delete_product`, or `delete_bank_account` on the live server, and no `get_bank_account`/`get_ksef_upo` either, despite what older docs may say. Don't assume a tool exists — check the live `tools/list` if in doubt.
