---
name: quento
description: |
  Interact with Quento via its REST API. Full coverage: invoices, clients, companies,
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

# /quento — Quento Invoicing API

Full REST API coverage: 32 endpoints across invoices, clients, companies (sellers), products, bank accounts, analytics, and KSeF (Polish national e-invoicing system).

## Agent invariants

**Always follow these rules:**

1. **Use `$SUBDOMAIN` from config** — every API call goes to `https://$SUBDOMAIN.quento.app/api/v1/actions/:tool_name`. Never hardcode the subdomain.
2. **GET for reads, POST for mutations** — `list_*`, `get_*`, `lookup_*` use GET with query params. Everything else uses POST with JSON body.
3. **Resolve IDs before acting** — use `list_clients`, `list_companies`, `list_invoices` to find IDs before passing them to mutation tools.
4. **VAT lookups first** — if a user provides a NIP or EU VAT number, ALWAYS call `lookup_company` before creating the client. It auto-fills name and address from the VAT registry.
5. **Minimal params** — Quento infers defaults for payment_method, currency, bank account, and dates from company settings. Only include what the user explicitly stated.
6. **KSeF is Poland-only** — KSeF endpoints only work for companies with `country: "PL"`.
7. **Invoice state flow** — draft → (issue) → issued → (mark_paid) → paid. You cannot edit a non-draft invoice. Cancel works from any state.

## Authentication setup

Set these in your environment or `.quento/config.json`:

```bash
export QUENTO_SUBDOMAIN="yourcompany"   # the subdomain of your Quento account
export QUENTO_API_KEY="qnt_..."         # from Settings → API in Quento
```

Check config:
```bash
cat .quento/config.json 2>/dev/null || echo "No project config found"
```

All examples below use:
```bash
BASE="https://$QUENTO_SUBDOMAIN.quento.app/api/v1/actions"
AUTH="Authorization: Bearer $QUENTO_API_KEY"
```

## Quick reference

| Task | Method | Endpoint |
|------|--------|----------|
| List invoices | GET | `list_invoices` |
| Get invoice | GET | `get_invoice?id=123` |
| Create invoice (draft) | POST | `create_invoice` |
| Update draft invoice | POST | `update_invoice` |
| Issue invoice | POST | `change_invoice_status` |
| Mark invoice paid | POST | `mark_invoice_paid` |
| Send invoice email | POST | `send_invoice_email` |
| Cancel invoice | POST | `change_invoice_status` |
| Create correction | POST | `create_correction_invoice` |
| Get PDF link | GET | `get_invoice_pdf_link?id=123` |
| List clients | GET | `list_clients` |
| Create client | POST | `create_client` |
| Update client | POST | `update_client` |
| Delete client | POST | `delete_client` |
| List companies | GET | `list_companies` |
| Lookup NIP/VAT | GET | `lookup_company?tax_id=...` |
| Get statistics | GET | `get_statistics?period=current_month` |
| List products | GET | `list_products` |
| Submit to KSeF | POST | `submit_invoice_to_ksef` |
| KSeF payables | GET | `list_ksef_payables` |

## Decision trees

### Creating an invoice

```
User wants to create an invoice?
├── Have client name only? → list_clients?search=name → get client_id
│   (If not found → check if user has NIP → lookup_company → create_client)
├── Have NIP/VAT number? → lookup_company?tax_id=NIP → create_client → use client_id
├── Have items? → include in create_invoice body
│   Format: [{description, quantity, unit_price, vat_rate, unit}]
│   Units: pcs, h, kg, m, szt, etc.
│   VAT rates: 0, 5, 8, 23 (Poland); 0, 7, 19 (Germany)
├── Ready to create? → POST create_invoice {client_id, items}
│   → Returns draft invoice with id and invoice_number
└── Issue it? → POST change_invoice_status {id, action: "issue"}
    → Send it? → POST send_invoice_email {id}
```

### Checking revenue / analytics

```
Revenue question?
├── "This month" → GET get_statistics?period=current_month
├── "Last month" → GET get_statistics?period=last_month
├── "This year" → GET get_statistics?period=current_year
├── Custom range → GET get_statistics?from_date=YYYY-MM-DD&to_date=YYYY-MM-DD
└── Specific currency → add &currency=PLN (otherwise all currencies reported)

Stats include:
  - revenue: money received (by paid_at date)
  - outstanding: all unpaid invoices (any period)
  - issued/paid/overdue counts
  - top clients by revenue
```

### Finding an invoice

```
Know invoice number? → GET get_invoice?invoice_number=001%2F06%2F2026
Know ID? → GET get_invoice?id=123
Search? → GET list_invoices?search=client+name
By status? → GET list_invoices?status=overdue
By period? → GET list_invoices?from_date=2026-06-01&to_date=2026-06-30
By payment date? → GET list_invoices?paid_from=2026-06-01&paid_to=2026-06-30
```

## Common workflows

### Create and issue an invoice

```bash
# 1. Find or create client (skip if you have client_id)
curl -sG "$BASE/list_clients" -H "$AUTH" --data-urlencode "search=Alpha Corp" | jq '.items[0].id'
# → 42

# 2. Create draft
curl -s -X POST "$BASE/create_invoice" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "client_id": 42,
  "items": [
    {"description": "Web development", "quantity": 10, "unit_price": 200, "vat_rate": 23, "unit": "h"},
    {"description": "Hosting", "quantity": 1, "unit_price": 50, "vat_rate": 23, "unit": "month"}
  ]
}' | jq '{id, invoice_number, total}'
# → {"id": 101, "invoice_number": "001/06/2026", "total": "2511.00 PLN"}

# 3. Issue it (assigns invoice number, finalizes)
curl -s -X POST "$BASE/change_invoice_status" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "id": 101,
  "action": "issue"
}' | jq '.status'

# 4. Send to client
curl -s -X POST "$BASE/send_invoice_email" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "id": 101
}' | jq '.sent'
```

### Mark invoice as paid

```bash
curl -s -X POST "$BASE/mark_invoice_paid" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "invoice_number": "001/06/2026",
  "payment_date": "2026-06-20"
}' | jq '.status'
```

### Get monthly revenue

```bash
curl -sG "$BASE/get_statistics" -H "$AUTH" \
  --data-urlencode "period=current_month" | jq '{revenue, outstanding, period}'
```

### Look up a company by NIP and create as client

```bash
# 1. Look up NIP in Polish VAT registry
curl -sG "$BASE/lookup_company" -H "$AUTH" --data-urlencode "tax_id=5261040828" | jq .

# 2. Create client with auto-filled data
curl -s -X POST "$BASE/create_client" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "name": "PEKAO S.A.",
  "nip": "5261040828",
  "country": "PL"
}' | jq '{id, name}'
```

### Create invoice in EUR for foreign client

```bash
curl -s -X POST "$BASE/create_invoice" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "client_id": 15,
  "currency": "EUR",
  "items": [
    {"description": "Consulting", "quantity": 5, "unit_price": 150, "vat_rate": 0, "unit": "h"}
  ],
  "pdf_secondary_language": "en"
}' | jq '{id, total}'
```

### Submit to KSeF (Poland only)

```bash
# Invoice must already be issued
curl -s -X POST "$BASE/submit_invoice_to_ksef" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "invoice_number": "001/06/2026"
}' | jq .

# Check status
curl -sG "$BASE/get_ksef_status" -H "$AUTH" --data-urlencode "invoice_number=001/06/2026" | jq .
```

---

## Resource reference

### Invoices

**List invoices**
```bash
GET /list_invoices
Params: status, invoice_number, client_id, currency, search,
        from_date, to_date, paid_from, paid_to, limit (default 10, max 100)

Status values: draft | issued | sent | paid | overdue | cancelled
Note: "issued", "sent", "overdue" are all unpaid invoices.
```

**Get invoice**
```bash
GET /get_invoice?id=123
GET /get_invoice?invoice_number=001%2F06%2F2026
```

**Create invoice** (creates as draft)
```bash
POST /create_invoice
Required: client_id, items[]
Optional: company_id, payment_terms, sale_date, issue_date, payment_method,
          bank_account_id, currency, notes, internal_notes, issue_location,
          purchase_order_number, split_payment, pdf_secondary_language,
          discount_percentage, stripe_invoice_id, based_on

Items: {description*, quantity*, unit_price*, vat_rate*, unit, product_id}
  * required per item
  unit_price: in invoice currency (not cents)
  vat_rate: percentage, e.g. 23 for 23%
  based_on: "stripe" | "paypal" | "klarna" | "revolut" | "wise"
```

**Update invoice** (draft only)
```bash
POST /update_invoice
Required: id OR invoice_number
Optional: client_id, sale_date, issue_date, payment_terms, currency,
          payment_method, bank_account_id, notes, internal_notes,
          discount_percentage, items[], replace_items (bool)

replace_items: true = replace all items; false (default) = merge by description
```

**Change invoice status**
```bash
POST /change_invoice_status
Required: (id OR invoice_number), action
action: "issue" (draft→issued) | "cancel" (any→cancelled)
```

**Mark invoice paid**
```bash
POST /mark_invoice_paid
Required: id OR invoice_number
Optional: payment_date (default: today)
```

**Send invoice email**
```bash
POST /send_invoice_email
Required: id OR invoice_number
Optional: recipient_email, recipient_emails[], message, auto_issue (default: true)
```

**Get PDF link**
```bash
GET /get_invoice_pdf_link?id=123
```

**Create correction invoice**
```bash
POST /create_correction_invoice
Required: original_invoice_id OR original_invoice_number
Optional: items[], correction_reason, other invoice fields
```

---

### Clients

**List clients**
```bash
GET /list_clients
Params: search, limit (default 10)
```

**Create client**
```bash
POST /create_client
Required: name
Optional: nip, tax_id, country, email, phone, address, city, postal_code,
          payment_terms, notes, locale, currency
```

**Update client**
```bash
POST /update_client
Required: id
Optional: same fields as create
```

**Delete client**
```bash
POST /delete_client
Required: id
Note: fails if client has invoices — deactivate instead
```

---

### Companies (sellers)

**List companies**
```bash
GET /list_companies
```

**Get company**
```bash
GET /get_company?id=1
GET /get_company?name=MyCompany
GET /get_company?nip=5261040828
```

**Lookup company by NIP/VAT** (external VAT registry lookup)
```bash
GET /lookup_company?tax_id=5261040828
GET /lookup_company?tax_id=DE123456789
Returns: name, address, city, postal_code, country, nip/vat_id
```

**Create company**
```bash
POST /create_company
Required: name
Optional: nip, regon, tax_id, country, email, phone, address, city,
          postal_code, website, default_currency, locale, bank_accounts[]
```

**Update company**
```bash
POST /update_company
Required: id
Optional: same fields as create
```

---

### Products

**List products**
```bash
GET /list_products
Params: search, limit
```

**Create product**
```bash
POST /create_product
Required: name, unit_price, vat_rate
Optional: description, unit, currency
```

**Update product**
```bash
POST /update_product
Required: id
Optional: name, unit_price, vat_rate, description, unit, currency
```

**Delete product**
```bash
POST /delete_product
Required: id
```

---

### Bank accounts

**List bank accounts**
```bash
GET /list_bank_accounts
Params: company_id, currency, active (bool)
```

**Get bank account**
```bash
GET /get_bank_account?id=5
```

**Create bank account**
```bash
POST /create_bank_account
Required: company_id, bank_name, account_number, currency
Optional: iban, swift_code, default (bool), active (bool)
```

**Update bank account**
```bash
POST /update_bank_account
Required: id
Optional: bank_name, account_number, currency, iban, swift_code, default, active
```

**Delete bank account**
```bash
POST /delete_bank_account
Required: id
```

---

### Analytics

**Get statistics**
```bash
GET /get_statistics
Params: period, from_date, to_date, currency

period: current_month (default) | last_month | current_quarter | last_quarter
        | current_year | last_year | all_time
from_date/to_date: override period (YYYY-MM-DD)
currency: ISO 4217 code (e.g. PLN, EUR) — omit for all currencies

Returns:
  revenue: money received in period (by payment date / paid_at)
  outstanding: all-time unpaid (issued + overdue)
  counts: draft, issued, paid, overdue
  top_clients: ranked by revenue
```

---

### KSeF (Poland only)

**Submit to KSeF** — requires issued invoice + company with KSeF token
```bash
POST /submit_invoice_to_ksef
Required: id OR invoice_number
```

**Get KSeF status**
```bash
GET /get_ksef_status?invoice_id=123
GET /get_ksef_status?invoice_number=001%2F06%2F2026
Optional: submission_id
Returns: status (pending | submitted | accepted | rejected | error), reference_number, UPO
```

**List KSeF submissions**
```bash
GET /list_ksef_submissions
Params: status (pending|submitted|accepted|rejected|error), limit, company_id
```

**List KSeF payables** — purchase invoices received via KSeF
```bash
GET /list_ksef_payables
Params: company_id, status, search, payment_details (ready|missing), limit (max 50)
```

**Get KSeF UPO** — Urzędowe Poświadczenie Odbioru (official receipt)
```bash
GET /get_ksef_upo?id=123
```

---

## Error handling

**HTTP status codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request — check params in `error` field |
| 401 | Unauthorized — verify `QUENTO_API_KEY` and subdomain |
| 402 | Plan limit reached — upgrade required |
| 403 | Forbidden — feature not available on your plan |
| 404 | Not found — check ID/invoice_number |
| 422 | Validation error — see `errors` field |
| 429 | Rate limit — wait and retry |
| 500 | Server error — try again |

**Error response shape:**
```json
{"error": "human-readable message", "errors": ["field: message"]}
```

**Common errors:**

- `"Unauthorized"` → wrong or missing API key; verify `Authorization: Bearer $QUENTO_API_KEY`
- `"Invoice cannot be modified"` → invoice is issued/paid; only drafts can be updated
- `"Client not found"` → wrong `client_id`; use `list_clients` to find it
- `"Plan limit reached"` → account on Free/Starter plan; show upgrade message
- `"KSeF not configured"` → company lacks KSeF credentials in Settings

---

## Configuration

Per-project config at `.quento/config.json`:
```json
{
  "subdomain": "yourcompany",
  "api_key": "qnt_..."
}
```

Initialize:
```bash
mkdir -p .quento
echo '{"subdomain": "yourcompany"}' > .quento/config.json
```

Add to `.gitignore` if config contains the API key:
```
.quento/config.json
```

---

## Notes

- **Bilingual PDFs**: set `pdf_secondary_language` on `create_invoice` — e.g. Polish company issuing to a German client: `"pdf_secondary_language": "de"`. Adds translations in parentheses.
- **Split payment (MPP)**: Polish VAT requirement for invoices > 15,000 PLN. Set `split_payment: true`.
- **Invoice number URL encoding**: `/` must be `%2F` in query strings: `001%2F06%2F2026`.
- **Multi-currency**: when no `currency` filter is given, `get_statistics` returns each currency separately — amounts are never summed across currencies.
- **Stripe invoices**: set `based_on: "stripe"` and `stripe_invoice_id` on `create_invoice` to track Stripe-originated invoices.
