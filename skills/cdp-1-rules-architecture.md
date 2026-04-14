# CDP | 1 | Rules + Architecture

## Connection Info 

Host: pgsql-cdpdb-pri.centralretail.com.vn · Database: redshiftdb · PostgreSQL 16

Tool: postgres_cdp:execute_query

Data range: par_month 202506–202603

Partner codes: GOH = GO!, TPS = Tops, MNG = MiniGo

---

## 🚫 Absolute Rules — Never Guess, Never Assume

1. Never guess column names or data types. Check the Datatype Reference first, or run information_schema or LIMIT 5. Especially: trans_type and tender_type are varchar — never compare with integer literals.
1. Never assume enum values. Run SELECT DISTINCT first if unsure.
1. Never assume empty result = "no data". Debug filters before drawing conclusions.
1. Never add JOINs not documented here without checking schema first.
1. Never interpret strange numbers without flagging explicitly.
1. Never assume table schema location. refined.* for analytics, report.* for output/views. 
> When in doubt → query to verify, never guess.

---

## ⚠️ Execution Rules

- 🔴 MUST read SQL Snippets entry before writing any query. Match pattern first. Do NOT derive from Event Reference Table.
- Always prefix schema: refined. or report.
- Ignore tables with suffix _bk, _bkup, _bk_a
- Timeout ~5 min → always filter on indexed columns, avoid full scans
---

## 📐 Architecture Overview

```javascript
Wallet (d_wallet)
└── Consumer (d_consumer)           — 1:1 with wallet
│   ├── d_consumer_contact          — phone, email
│   ├── d_consumer_address          — address
│   ├── d_consumer_meta             — KV extras
│   ├── d_consumer_dimension        — label/value pairs
│   └── d_consumer_segmentation     — segment data
├── Identity (d_identity)           — phone/card/email identifiers
│   └── d_identity_meta             — KV extras
├── Account (d_account)             — POINTS or ECOUPON accounts
│   ├── d_account_meta              — KV: debitpointamount, accounttransactionid
│   ├── d_token                     — voucher barcode, 1:1 with ECOUPON account
│   └── d_campaign                  — campaign linked to ECOUPON account
└── Wallet Transactions (f_wallet_transaction)  — wx level
    ├── f_wallet_transaction_account            — bridge wx → ax
    ├── f_wallet_account_transaction            — ax level (raw point movements)
    │   ├── f_account_transaction_detail        — bill receipt, store, amount
    │   └── f_ref_voucher                       — voucher_no ↔ account_transaction_id
    ├── f_wallet_transaction_adj_result         — adjustment result
    └── d_wallet_transaction_meta               — KV metadata for wx
```

Three-level hierarchy:

- wx level (f_wallet_transaction) — one row per POS transaction. Has summary columns.
- bridge (f_wallet_transaction_account) — maps wx → ax via id ↔ account_acccounttransactionid (triple-c typo — real column name).
- ax level (f_wallet_account_transaction) — one row per account movement. Use ax.value for point amounts by default.
---

## ⚠️ Datatype Reference — Check Before Writing WHERE / JOIN

> Columns not listed here → run information_schema first.

> 🔴 Most common errors:

- trans_type, tender_type, reason_code, sales_channel → varchar, always use string literals

> - personal_birthdate → varchar, do NOT use date functions

> - last_updated (f_wallet_transaction) → timestamptz, others are timestamp

> - siteid → numeric, branch_code is varchar → cast: branch_code::numeric

> - gross_sales DOES NOT EXIST in sales_* → use net_price_tot

> - d_branch_site DOES NOT EXIST → use report.d_site_region

> - refined.d_customer_info DOES NOT EXIST → use report.d_customer_info

> - balanaces_usable in d_account has triple-a typo — spell exactly as shown

---

## 🔗 Join Patterns

