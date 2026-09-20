# Manual Payments Ledger — Case Study

> A password-protected Cloudflare Worker that replaced a chaotic manual process: upload bank statements, reconcile against daily cashbook entries, export clean records — all without a backend server, database subscription, or IT ticket.

<p>
  <img src="https://img.shields.io/badge/role-Sole%20Builder-orange" alt="Sole Builder">
  <img src="https://img.shields.io/badge/runtime-Cloudflare%20Workers-F38020?style=flat&logo=cloudflare&logoColor=white" alt="Cloudflare Workers">
  <img src="https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black" alt="JavaScript">
</p>

> 🔒 **Why this is a case study, not full source.** This system processes real bank statements and store-level cash data for a live business. Production code and credentials are kept private. This page documents the architecture, reconciliation logic, and engineering decisions.

---

## The problem

A 25+-outlet entertainment chain processes payments across Card, Cash, UPI, and third-party gateways every day. The finance team had no central record:

- Bank credits arrived in PDF/Excel statements, one per account
- Daily cashbook entries were tracked in a separate Google Sheet
- Reconciliation happened (or didn't) via manual copy-paste
- Payment mode mis-categorisation (BharatPe showing up as "Resilient Innovations", Zomato as "Eternal Limited") made matching unreliable
- Month-end reconciliation: hours of work, still full of gaps

---

## What I built

A single Cloudflare Worker (no origin server, no database) serving three tools behind Basic Auth:

### `/upload` — Bank Statement Ingestion
Upload raw bank statements (CSV/Excel) from any ICICI account. The worker parses entity names against a known PSP registry — `Resilient Innovations` → BharatPe, `Eternal Limited` → Zomato, etc. — so payment modes are classified automatically. Stores data in Cloudflare KV.

### `/export` — Clean Ledger Export
Pull any date range, any store, any payment mode. Downloads as a clean Excel-compatible file. Finance team gets the same view every time, no manual reformatting.

### `/reconcile` — Bank Reconciliation Dashboard
The core feature. Loads the BankCredits sheet alongside cashbook daily entries and surfaces:

| Check | What it catches |
|---|---|
| Credits with no matching cashbook entry | Banking received, cashbook not closed |
| Cashbook entries with no matching credit | Closed in sheet, payment never arrived |
| Amount mismatches (same day, same mode) | Partial payment, gateway hold, entry error |
| Unknown entity names | New PSP not yet in the registry |

Traffic-light status per store per day — Green (reconciled), Amber (partial), Red (missing).

---

## Beyond reconciliation: the full "360 system"

What started as a reconciliation tool grew into a 6-module finance system, all on the same Worker:

| Module | What it does |
|---|---|
| Bills | Vendor bill capture and tracking |
| Party Master | Central registry of vendors/parties, feeds the matching engine |
| Bill ↔ Payment Matching | Links incoming payments to the bills they settle |
| Manage P&L | Rolls store-level income/expense into a management P&L view |
| **GST Ledger** | Tracks GST collected/paid per transaction, structured for return filing |
| **TDS Ledger** | Tracks TDS deducted per vendor payment against the applicable section/rate |

The GST and TDS ledgers were the two modules that mattered most to get right, since both feed statutory filings rather than just internal reporting — a wrong classification here isn't a dashboard glitch, it's a compliance problem. Real source formats forced specific handling: GSTR-2B arrives as 12 state-wise sheets with a two-row merged header, and TDS certificates arrive as one Form 16A **PDF per vendor per quarter** — so ingestion had to handle merged-header spreadsheets and PDF extraction, not just clean CSVs.

One module remains open — a "reconciliation improvement" pass flagged by the finance owner, pending a firmer spec on exactly what additional matching behaviour is needed.

---

## Architecture

```
Browser → Cloudflare Worker (Basic Auth)
              ├── /upload  → parse → Cloudflare KV store
              ├── /export  → KV read → Excel blob response
              └── /reconcile → KV + BankCredits sheet → diff table
```

**Why Cloudflare Workers, not a traditional server?**

Zero cold-start, global edge, no server to maintain. The entire thing deploys with `wrangler deploy` in under 30 seconds. For a tool used by a small finance team a few times per week, spinning up and paying for a VPS is unnecessary.

**Why Basic Auth, not a full login system?**

This tool is internal-only, behind a single shared password. A full auth system (sessions, password resets, user management) would add weeks of work and attack surface for something with 3–4 users. HTTP Basic Auth over HTTPS is secure enough for the threat model.

**PSP entity registry**

Indian payment processors rarely appear under their brand name on bank statements — they use their registered company names. The worker includes a lookup table mapping legal entity names to payment brands. This table is maintained as a flat list and can be extended without a code deploy.

---

## Key engineering decisions

**KV over a database**

Cloudflare KV is eventually consistent and read-optimised — perfect for a ledger where writes happen once (statement upload) and reads happen many times (reconciliation, export). No schema migrations, no connection pooling, no monthly database bill.

**Reconciliation logic**

Matching bank credits to cashbook entries is fuzzy: amounts may differ by rounding, dates may differ by one day (late-night deposits), and a single credit may cover multiple cashbook lines (batch settlement). The reconciliation engine:
1. Exact-matches first (date + store + mode + amount within ±₹1)
2. Fuzzy-matches on date ±1 day for modes known to settle overnight
3. Flags remaining unmatched items with the closest candidate and the delta

**No frontend framework**

The UI is plain HTML + vanilla JS. No build step, no `node_modules`, no bundler. The entire tool is one Worker script and three HTML files. This keeps the deploy surface minimal and the audit trail obvious.

---

## Impact

- Finance team can reconcile a month of statements in **minutes instead of hours**
- Unknown PSP names flagged automatically — no more "what is Resilient Innovations?"
- Missing credits surfaced before month-end, not during audit
- Deployed and maintained by one person, zero infrastructure overhead

---

## Tech stack

`Cloudflare Workers` · `Cloudflare KV` · `JavaScript` · `Wrangler CLI` · `HTML/CSS` · `Bank Statement Parsing` · `REST API design`

---

### Author

**Souvik Kundu** — Business Intelligence & Automation Engineer.

📫 [LinkedIn](https://linkedin.com/in/souvik-kundu-bi) · [GitHub](https://github.com/Souvikkundu369)
