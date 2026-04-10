---
icon: glasses
layout:
  width: wide
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# CDP | LOYALTY



<pre class="language-skill"><code class="lang-skill"><strong>---
</strong>name: db-query-assistant
description: "Use this skill whenever the user asks anything about data in the CDP database — even casual questions like 'how many customers?', 'revenue this month?', 'find member X', 'write a query for...', 'what is the schema of table Y?', 'why is the query slow?', 'trace a wallet', 'point history', 'how much earn/redeem', 'why are points wrong'. Also trigger when the user wants to explore tables, debug SQL, translate any business question into a query, or understand the THE1 wallet event flow. Always consult this skill BEFORE writing SQL — never guess table or column names. Trigger on keywords: query, SQL, table, schema, customer, member, transaction, sales, loyalty, point, wallet, report, revenue, earn, redeem, refund, voucher, ecoupon, settle, spend, campaign, the1, v_loyalty_transaction."
---

# DB Query Assistant — CDP (postgres_cdp)

## Connection Info

**Host:** `pgsql-cdpdb-pri.centralretail.com.vn` · **Database:** `redshiftdb` · **PostgreSQL 16**  
**Tool:** `postgres_cdp:execute_query`  
**Partner codes:** `GOH` = GO!, `TPS` = Tops, `MNG` = MiniGo

---

## 🚫 Absolute Rules — Never Guess, Never Assume

1. **Never guess column names or data types.** Check Datatype Reference first. `trans_type` and `tender_type` are **varchar** — never compare with integers.
2. **Never assume enum values.** Run `SELECT DISTINCT` first if unsure.
3. **Never assume empty result = "no data".** Debug filters before drawing conclusions.
4. **Never add JOINs not documented here** without checking schema first.
5. **Never assume table schema location.** `refined.*` for analytics, `report.*` for output/views. `refined.d_customer_info` does NOT exist — use `report.d_customer_info`.

> **General rule:** When in doubt → query to verify, never guess.

---

## ⚠️ Execution Rules

- 🔴 **MUST read SQL Snippets before writing any query.** Match business question to a pattern first. Do NOT derive from the Event Reference Table — that is for lookup only.
- 🔴 **Do NOT execute queries without user confirmation.** Draft SQL → show to user → wait for "ok" / "run it" → then call `postgres_cdp:execute_query`.
- 🔴 **When user asks for a query: return SQL code only — no lengthy explanation.** Always read the skill carefully and verify before returning. Never guess column or table names.
- Always prefix schema: `refined.` or `report.`
- Ignore tables with suffix `_bk`, `_bkup`, `_bk_a`
- Timeout ~5 min → always filter on indexed columns, avoid full scans

---

## 💡 wx Level vs ax Level — Which to Use

| | wx level (`f_wallet_transaction`) | ax level (`f_wallet_account_transaction`) |
|---|---|---|
| Rows | 1 per POS receipt | 1 per point movement |
| Point amount | `summary_result_points_*` (aggregated) | `value` (raw, per event) ✅ **default** |
| When to use | Only when user says "wx level" / "summary" | **Always by default** |

> ⚠️ When using ax level, **always** filter `da.type = 'POINTS'` + correct `ax.event` to avoid mixing ECOUPON rows.

---

## ✅ Valid Event Pairs for POINTS

> ⚠️ `POSCONNECT.WALLET.SETTLE` also generates `ax_event='REDEEM'` for ECOUPON accounts — without `da.type='POINTS'` filter, voucher rows will mix into point totals.

| event_name | ax_event | Meaning |
|---|---|---|
| `POSCONNECT.WALLET.SETTLE` | `EARN` | Credit base earn points from purchase |
| `POSCONNECT.WALLET.SETTLE` | `CREDIT` | Credit redeemed voucher points back |
| `POSCONNECT.WALLET.REFUND` | `REFUND_DEBIT` | Claw back points (×2: original earn + redeem) |
| `POSCONNECT.WALLET.REFUND` | `EARN` | Recalculate earn after refund |
| `POSCONNECT.WALLET.REFUND` | `CREDIT` | Recalculate redeemed points after refund |
| `POSCONNECT.WALLET.SPEND` | `SPEND` | Use points at POS → deduct POINTS |
| `POSCONNECT.WALLET.SPEND.VOID` | `VOID` | Cancel spend → restore POINTS |
| `SERVICE.WALLET.TRANSACTION.DEBIT` | `DEBIT` | Deduct points for Cash Coupon exchange |
| `SERVICE.WALLET.TRANSACTION.CREDIT` | `CREDIT` | Restore points due to system error |
| `SERVICE.WALLET.BACKENDPOINTS` | `CREDIT` | Manual point adjustment (CS/ops) |
| `SERVICE.WALLET.BACKENDPOINTS` | `DEBIT` | Manual point adjustment (CS/ops) |
| `SERVICE.WALLET.POINTS.EXPIRE` | `EXPIRY` | Points expired |

**Mandatory filter when querying points:**
```sql
JOIN refined.d_account da ON ax.account_id::text = da.account_id::text
WHERE da.type = 'POINTS'
  AND ax.event_name IN (
      'POSCONNECT.WALLET.SETTLE','POSCONNECT.WALLET.REFUND',
      'POSCONNECT.WALLET.SPEND','POSCONNECT.WALLET.SPEND.VOID',
      'SERVICE.WALLET.TRANSACTION.DEBIT','SERVICE.WALLET.TRANSACTION.CREDIT',
      'SERVICE.WALLET.BACKENDPOINTS','SERVICE.WALLET.POINTS.EXPIRE'
  )
  AND ax.event IN ('EARN','CREDIT','DEBIT','SPEND','VOID','REFUND_DEBIT','EXPIRY')
</code></pre>

***

### 🔑 Index Strategy

> **Rule:** If unsure which index to use — **ask the user first**, do not self-select.

| Situation                                  | Priority filter               |
| ------------------------------------------ | ----------------------------- |
| **Single wallet query** (have `wallet_id`) | `wallet_id` — most selective  |
| **Aggregate by time** (no `wallet_id`)     | `event_stored_at BETWEEN ...` |
| **Filter by date on sales tables**         | `business_date`               |
| **Filter by date on dimension tables**     | `date_created`                |

> 🔴 **NEVER use `par_day` or `par_month` as filters in any query.**\
> Reason: ETL loads data for day N into partition day N+1 → filtering by par will return **data shifted by 1 day**.\
> Additionally, dimension tables (`d_account`, `d_wallet`, `d_consumer`, `d_identity`) only have a single par\_day (today's snapshot) → completely useless for date filtering.\
> **Always use:** `event_stored_at` · `business_date` · `date_created` · `transaction_time` for date/time filtering.

**Per-table indexed columns:**

| Table                          | Indexed columns                                                                                           |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `f_wallet_transaction`         | `wallet_id` · `wallet_transaction_id` · `id` · `event_stored_at`                                          |
| `f_wallet_account_transaction` | `account_id` · `account_transaction_id` · `event_stored_at` DESC                                          |
| `f_wallet_transaction_account` | `wallet_transaction_id` · `account_acccountid` · `account_acccounttransactionid` · `event_stored_at` DESC |
| `f_account_transaction_detail` | `account_transaction_id` · `event_id` · `event_stored_at`                                                 |
| `f_ref_voucher`                | `account_transaction_id` · `voucher_no` · `event_stored_at`                                               |
| `d_wallet`                     | `wallet_id` · `status` · `date_created`                                                                   |
| `d_account`                    | `wallet_id` · `account_id` · `campaign_id`                                                                |
| `d_consumer`                   | `consumer_id` · `wallet_id`                                                                               |
| `d_identity`                   | `identity_id` · `wallet_id`                                                                               |
| `d_token`                      | `token_id` · `account_id`                                                                                 |
| `d_campaign`                   | `campaign_id` · `ees_master_event_id` ⚠️ no partition index → filter `date_created` or `campaign_id`      |
| `sales_receipt_all`            | `receipt_no` · `partner_code` · `branch_code` · `business_date` · `card_no`                               |
| `sales_sku_all`                | `receipt_no` · `partner_code` · `business_date` · `(card_no, business_date)`                              |
| `sales_tender_all`             | `receipt_no` · `partner_code` · `branch_code` · `business_date` · `card_no`                               |

> ⚡ `d_account` has no composite `(wallet_id, type)` index — filter `wallet_id` first, then `AND type = 'ECOUPON'`.

***

### ⚠️ Datatype Reference

> Columns not listed → run `information_schema` first.

| Column                                           | Table(s)                                                                                                  | Type           | Gotcha                                                   |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------- | -------------- | -------------------------------------------------------- |
| `wallet_id`                                      | all                                                                                                       | varchar        |                                                          |
| `account_id`                                     | all                                                                                                       | varchar        |                                                          |
| `account_transaction_id`                         | f\_wallet\_account\_transaction, f\_wallet\_transaction\_account, f\_account\_transaction\_detail         | varchar        |                                                          |
| `wallet_transaction_id`                          | f\_wallet\_transaction, f\_wallet\_transaction\_account                                                   | varchar        |                                                          |
| `id`                                             | f\_wallet\_transaction, f\_wallet\_transaction\_account, f\_account\_transaction\_detail, f\_ref\_voucher | varchar        |                                                          |
| `event_name`                                     | all f\_wallet\_\*                                                                                         | varchar        |                                                          |
| `event` (ax\_event)                              | f\_wallet\_account\_transaction                                                                           | varchar        |                                                          |
| `value`                                          | f\_wallet\_account\_transaction                                                                           | bigint         |                                                          |
| `balance_before_*` / `balance_after_*`           | f\_wallet\_account\_transaction                                                                           | bigint         |                                                          |
| `summary_result_points_*`                        | f\_wallet\_transaction                                                                                    | bigint         |                                                          |
| `basket_contents` / `basket_payment` / `account` | f\_wallet\_transaction                                                                                    | jsonb          |                                                          |
| `overrides`                                      | d\_account                                                                                                | jsonb          |                                                          |
| `details` / `settings` / `offer` / `rules`       | d\_campaign                                                                                               | jsonb          |                                                          |
| `data`                                           | d\_consumer                                                                                               | jsonb          |                                                          |
| `type`                                           | d\_account, d\_consumer, d\_identity, f\_wallet\_transaction                                              | varchar        |                                                          |
| `status`                                         | d\_account, d\_wallet, f\_wallet\_transaction, d\_token                                                   | varchar        |                                                          |
| `campaign_id`                                    | d\_account, d\_campaign                                                                                   | varchar        |                                                          |
| `personal_birthdate`                             | d\_consumer                                                                                               | varchar ⚠️     | NOT a date type — no date functions                      |
| `partner_code`                                   | sales\_\*                                                                                                 | varchar        |                                                          |
| `branch_code`                                    | sales\_\*, d\_mer\_structure                                                                              | varchar        |                                                          |
| `trans_type`                                     | sales\_receipt\_all, sales\_sku\_all, sales\_tender\_all                                                  | varchar ⚠️     | `'1'`=normal `'2'`=void `'3'`=return                     |
| `tender_type`                                    | sales\_tender\_all                                                                                        | varchar ⚠️     | `'90'`=Pay with Point `'67'`=Cash Voucher                |
| `net_price_tot`                                  | sales\_\*                                                                                                 | numeric        |                                                          |
| `sales_channel_main`                             | sales\_sku\_all                                                                                           | varchar ⚠️     | `'1'`=Online `'2'`=Offline                               |
| `customer_type`                                  | sales\_sku\_all                                                                                           | integer ⚠️     | no quotes needed                                         |
| `trans_date` / `business_date`                   | sales\_\*                                                                                                 | date           |                                                          |
| `event_stored_at` / `date_created`               | f\_wallet\__, d\__                                                                                        | timestamp      |                                                          |
| `last_updated`                                   | f\_wallet\_transaction                                                                                    | timestamptz ⚠️ | careful in UNION                                         |
| `reason_code`                                    | f\_staff\_point, f\_account\_transaction\_detail                                                          | varchar ⚠️     | NOT integer                                              |
| `amount`                                         | f\_account\_transaction\_detail, f\_ref\_voucher                                                          | bigint         |                                                          |
| `siteid`                                         | report.d\_site\_region                                                                                    | numeric ⚠️     | join: `branch_code::numeric = siteid`                    |
| `balanaces_usable`                               | d\_account                                                                                                | bigint ⚠️      | triple-a typo — spell exactly                            |
| `consent_agree`                                  | report.d\_customer\_info                                                                                  | boolean ⚠️     | `= true` not `= 'true'`                                  |
| `birthdate`                                      | report.d\_customer\_info                                                                                  | date           | DIFFERENT from `d_consumer.personal_birthdate` (varchar) |

> 🔴 **Most common errors:** `trans_type`, `tender_type`, `reason_code`, `sales_channel_main` are **varchar** · `customer_type` is **integer** · `gross_sales` DOES NOT EXIST → use `net_price_tot` · `d_branch_site` DOES NOT EXIST → use `report.d_site_region` · `refined.d_customer_info` DOES NOT EXIST → use `report.d_customer_info`

***

### 🗂️ Schema

#### refined — Member & Wallet

| Table                | PK                         | Key Columns                                                                                                                                                           |
| -------------------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `d_wallet`           | `wallet_id`                | `status`: ACTIVE/DELETED/MERGED · `state`: EARNONLY/EARNBURN/DEFAULT                                                                                                  |
| `d_consumer`         | `consumer_id`, `wallet_id` | 1 wallet = 1 consumer · `personal_firstname/lastname/birthdate/gender` · `data` (JSONB)                                                                               |
| `d_consumer_contact` | `consumer_id`              | `type` (phone/email), `value`                                                                                                                                         |
| `d_identity`         | `identity_id`              | types: CUSTOMER\_ID/EMAIL/MOBILE/STAFF\_ID · `value` · `status`: ACTIVE/DELETED/SUSPENDED                                                                             |
| `d_account`          | `account_id`               | `wallet_id` NOT UNIQUE · `type='POINTS'` or `'ECOUPON'` · `balances_available`, `balances_current` · `state`: EARNONLY/EARNBURN/LOADED/UNLOADED · `overrides` (JSONB) |
| `d_campaign`         | `campaign_id`              | `status`: ACTIVE/INACTIVE/SUSPENDED/EXPIRED/DRAFT/DELETED · `class`: CONTINUITY/STAMP\_CARD/COUPON · `details`/`offer`/`settings` (JSONB)                             |
| `d_token`            | `token_id`, `account_id`   | `token` (barcode), 1:1 with `account_id` · `valid_from/to`                                                                                                            |
| `d_account_meta`     | `account_id`, `key`        | `key='accounttransactionid'` → links to ax · `key='debitpointamount'` → points for coupon exchange                                                                    |

#### refined — Store & Sales

| Table             | PK                 | Description                                                                     |
| ----------------- | ------------------ | ------------------------------------------------------------------------------- |
| `d_store_master`  | `store_code`       | `bu`, `banner`, `banner_code`, `store_name`, `companyname` (NOT company\_name)  |
| `d_tender`        | `id`               | `tender_name`, `banner_code` · id=`'90'`=Pay with Point, id=`'67'`=Cash Voucher |
| `d_mer_structure` | `id` (= mer\_code) | `cate_lv1`→`lv5` hierarchy · `branch_code` = BU code                            |

> ⚠️ `d_branch_site` DOES NOT EXIST → use `report.d_site_region`: `siteid` (numeric), `sitename`, `ops_region`, `region`, `bu`.

#### refined — Fact: Loyalty

| Table                          | Key Columns                                                                                                                                                                                   |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `f_wallet_transaction`         | **wx level** · `wallet_id`, `wallet_transaction_id`, `id` · `transaction_time` = POS time · `type` = wx\_type · `reference` · `summary_result_points_*` · `basket_contents`, `basket_payment` |
| `f_wallet_account_transaction` | **ax level** · `account_transaction_id`, `account_id` · `event` = ax\_event · `value` = raw point amount · `balance_before_*/after_*`                                                         |
| `f_wallet_transaction_account` | **Bridge** · maps wx via `id` · maps ax via `account_acccounttransactionid` _(triple-c typo)_                                                                                                 |
| `f_account_transaction_detail` | `billreceipt`, `amount`, `merchant_store_id`, `total_used_points`, `channel`, `reason_code`                                                                                                   |
| `f_ref_voucher`                | Maps `voucher_no` ↔ `account_transaction_id`                                                                                                                                                  |
| `f_staff_point`                | `staff_id`, `phone_number`, `bu`, `storeid` (NOT store\_id), `point_value`, `reason_code` (varchar), `date` (date type), `id` (uuid PK)                                                       |
| `f_account_create_campaign`    | `account_id`, `campaign_id`, `exchange_value`, `voucher_value`                                                                                                                                |

#### refined — Fact: Sales

| Table               | Key Columns                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sales_receipt_all` | `partner_code`, `branch_code`, `member_number`, `card_no`, `receipt_no`, `business_date`, `net_price_tot` · `trans_type` varchar: `'1'`=normal `'2'`=void `'3'`=return |
| `sales_sku_all`     | `sku_id/name`, `qty` (numeric), `net_price_tot`, `cate_id`, `sales_channel_main`                                                                                       |
| `sales_tender_all`  | `tender_type` varchar → `d_tender.id`, `net_price_tot`, `tender_ref_no`                                                                                                |

#### report — Views & Tables

| Table/View        | Columns                                                                                                                                                                                                                             |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `d_customer_info` | `consumer_id`, `wallet_id`, `wallet_status`, `customer_id` (card\_no), `phone_number`, `email`, `first_name`, `last_name`, `gender`, `birthdate` (date ✅), `consent_agree` (boolean), `state_point`, `date_created`, `last_updated` |
| `d_site_region`   | `siteid` (numeric ⚠️), `sitename`, `ops_region`, `region`, `bu`                                                                                                                                                                     |

***

### 🔗 Join Patterns

| Goal                  | How to join                                                                                                                                          |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Store name from sales | `sales_*.branch_code = d_store_master.store_code`                                                                                                    |
| Site/region info      | `report.d_site_region WHERE siteid = branch_code::numeric`                                                                                           |
| Tender name           | `sales_tender_all.tender_type = d_tender.id AND partner_code = d_tender.banner_code`                                                                 |
| Category hierarchy    | `sales_sku_all.cate_id = d_mer_structure.id`                                                                                                         |
| Phone/email of wallet | `d_identity WHERE type='MOBILE'` or `d_wallet → d_consumer → d_consumer_contact`                                                                     |
| Customer info quickly | `report.d_customer_info WHERE phone_number = '...'`                                                                                                  |
| wx → ax               | `f_wallet_transaction.id = f_wallet_transaction_account.id` → `.account_acccounttransactionid = f_wallet_account_transaction.account_transaction_id` |
| ax → bill detail      | `f_wallet_account_transaction.account_transaction_id = f_account_transaction_detail.account_transaction_id`                                          |
| wx → meta             | `f_wallet_transaction.wallet_transaction_id = d_wallet_transaction_meta.wallet_transaction_id AND key = 'transactiondescription'`                    |
| Points only filter    | `JOIN refined.d_account da ON ax.account_id::text = da.account_id::text WHERE da.type = 'POINTS'`                                                    |

***

### 🧠 Event Reference Table

> For understanding meaning only — do NOT use to derive new queries. Use SQL Snippets instead.

| event\_name                           | wx\_type       | ax\_event             | account       | Meaning                                 |
| ------------------------------------- | -------------- | --------------------- | ------------- | --------------------------------------- |
| `POSCONNECT.WALLET.SETTLE`            | SETTLE         | REDEEM                | ECOUPON       | Condition voucher applied               |
| `POSCONNECT.WALLET.SETTLE`            | SETTLE         | **EARN**              | **POINTS** 🟡 | Credit base earn points                 |
| `POSCONNECT.WALLET.SETTLE`            | SETTLE         | **CREDIT**            | **POINTS** 🟡 | Credit voucher points back              |
| `POSCONNECT.WALLET.SETTLE`            | REDEEM\_SETTLE | REDEEM                | ECOUPON       | Apply Cash Voucher discount             |
| `POSCONNECT.WALLET.REFUND`            | REFUND         | **REFUND\_DEBIT ×2**  | **POINTS** 🟡 | Claw back original earn + redeem        |
| `POSCONNECT.WALLET.REFUND`            | REFUND         | **EARN**              | **POINTS** 🟡 | Recalculate earn after refund           |
| `POSCONNECT.WALLET.REFUND`            | REFUND         | UNREDEEM              | ECOUPON       | Claw back ECOUPON                       |
| `POSCONNECT.WALLET.REFUND`            | REFUND         | REDEEM                | ECOUPON       | Recalculate condition voucher           |
| `POSCONNECT.WALLET.REFUND`            | REFUND         | **CREDIT**            | **POINTS** 🟡 | Recalculate redeemed points             |
| `POSCONNECT.WALLET.SPEND`             | SPEND          | **SPEND**             | **POINTS** 🟡 | Use points at POS                       |
| `POSCONNECT.WALLET.SPEND.VOID`        | SPEND          | **VOID**              | **POINTS** 🟡 | Cancel spend → restore points           |
| `POSCONNECT.ACCOUNT.UNREDEEM`         | —              | UNREDEEM              | ECOUPON       | Cancel Cash Voucher (before SETTLE)     |
| `WALLET.ACCOUNT.CREATE.CAMPAIGN`      | CASH\_COUPON   | CREATE                | ECOUPON       | Exchange points for Cash Coupon         |
| `WALLET.ACCOUNT.CREATE.CAMPAIGN`      | OFFER          | CREATE                | ECOUPON       | Receive voucher from campaign           |
| `WALLET.ACCOUNT.CREATE.CAMPAIGN`      | QUEST          | CREATE                | ECOUPON       | Tracking objectives                     |
| `SERVICE.WALLET.TRANSACTION.DEBIT`    | DEBIT          | **DEBIT**             | **POINTS** 🟡 | Deduct points for Cash Coupon exchange  |
| `SERVICE.WALLET.TRANSACTION.CREDIT`   | CREDIT         | **CREDIT**            | **POINTS** 🟡 | Restore points (system error)           |
| `SERVICE.WALLET.TRANSACTION.UNREDEEM` | UNREDEEM       | UNREDEEM              | ECOUPON       | Cancel voucher via backend              |
| `SERVICE.WALLET.BACKENDPOINTS`        | ADJUSTMENT     | **CREDIT/DEBIT**      | **POINTS** 🟡 | Manual point adjustment                 |
| `SERVICE.WALLET.POINTS.EXPIRE`        | EXPIRY         | **EXPIRY**            | **POINTS** 🟡 | Points expired                          |
| `SERVICE.WALLET.MERGE`                | various        | (same as normal flow) | —             | Replay transaction when merging wallets |

> 🟡 = POINTS event — **must filter `da.type = 'POINTS'`**

**Transaction flow:**

* **Normal purchase:** SETTLE → REDEEM×N (ECOUPON) + EARN + CREDIT (POINTS)
* **Purchase + Cash Voucher:** CREATE(CASH\_COUPON) → DEBIT → REDEEM\_SETTLE → SETTLE+EARN
* **Refund:** REFUND → UNREDEEM + REFUND\_DEBIT×2 + EARN + REDEEM + CREDIT

***

### 📝 SQL Snippets

> 🔴 **Always pull the matching snippet before writing any query.**

#### 1. Customer Lookup

```sql
SELECT consumer_id, wallet_id, first_name, last_name, phone_number
FROM report.d_customer_info
WHERE phone_number = ':phone';
```

#### 2. Revenue / Sales

```sql
SELECT business_date, partner_code, SUM(net_price_tot) AS net_revenue
FROM refined.sales_receipt_all
WHERE partner_code = 'TPS'
  AND business_date BETWEEN '2026-01-01' AND '2026-01-31'
GROUP BY business_date, partner_code;
```

#### 3. Params CTE — Loyalty Transactions by Wallet + Date Range

```sql
WITH params AS (
    SELECT
        '89850772'::text        AS p_wallet_id,
        '2025-06-01'::timestamp AS p_date_from,
        '2025-06-30'::timestamp AS p_date_to
),
wx_ax AS (
    SELECT w.wallet_id, w.wallet_transaction_id, w.event_name, w.type AS wx_type
    FROM refined.f_wallet_transaction w
    JOIN params p ON w.wallet_id = p.p_wallet_id
    WHERE w.event_stored_at BETWEEN p.p_date_from AND p.p_date_to
),
ecoupon AS (
    SELECT da.wallet_id
    FROM refined.d_account da
    JOIN params p ON da.wallet_id = p.p_wallet_id
    WHERE da.type = 'ECOUPON'
),
unredeem AS (
    SELECT da.wallet_id
    FROM refined.f_wallet_account_transaction ax
    JOIN refined.d_account da ON ax.account_id::text = da.account_id::text
    JOIN params p ON da.wallet_id = p.p_wallet_id
    WHERE ax.event_name = 'POSCONNECT.ACCOUNT.UNREDEEM'
      AND ax.event_stored_at BETWEEN p.p_date_from AND p.p_date_to
)
SELECT * FROM wx_ax
UNION ALL SELECT * FROM ecoupon
UNION ALL SELECT * FROM unredeem
ORDER BY event_stored_at;
```

#### 4. Point Ledger — Daily Point Movement Aggregate

> **Use when:** aggregating point movements across many wallets, reporting earn/spend/adjust by date or store.\
> **Do NOT use when:** debugging a single wallet → use Master Query (section 7).

**Branch index strategy:**

| Branch          | Main table                          | Priority filter                                                                        |
| --------------- | ----------------------------------- | -------------------------------------------------------------------------------------- |
| EARN            | `f_wallet_transaction` + ax         | `w.transaction_time >= 'YYYY-MM-DD'` · `d.type = 'POINTS'`                             |
| STAFF\_EARN     | `f_wallet_transaction`              | `w.event_name = 'SERVICE.WALLET.BACKENDPOINTS'` · reference prefix 20/21/22/23         |
| ADJUSTMENT      | `f_wallet_account_transaction` + wx | `event_name = 'SERVICE.WALLET.BACKENDPOINTS'` · reference code 101/105/107/108/109/110 |
| EXCHANGE        | `d_account`                         | `d.date_created::date >= 'YYYY-MM-DD'` · `campaign_id NOT IN ('90064586')`             |
| PAY WITH POINT  | `sales_tender_all`                  | `tender_type = '90'`                                                                   |
| USED AS VOUCHER | `sales_tender_all`                  | `tender_type = '67'`                                                                   |

> ⚠️ `exchange_camp` CTE filters campaigns with `offer.Reward.Standard.Value.DiscountAmount`. Exclude `campaign_id = '90064586'`.\
> ⚠️ EARN branch uses `w.transaction_time`. STAFF\_EARN uses `w.event_date`. Different columns — do not mix.\
> ⚠️ `f_staff_point.reason_code` is **varchar** — filter: `reason_code IN ('201', '202', ...)`.

```sql
WITH exchange_camp AS (
    SELECT
        dc.*,
        json_extract_path_text(dc.offer::json, 'Reward', 'Standard', 'Value', 'DiscountAmount')::real AS discount_amt
    FROM refined.d_campaign dc
    WHERE json_extract_path_text(dc.offer::json, 'Reward', 'Standard', 'Value', 'DiscountAmount') IS NOT NULL
),
e_point AS (
    -- Branch EARN: points from POS purchase (SETTLE + REFUND)
    SELECT
        DATE(w.transaction_time) AS event_date,
        w.wallet_id,
        ax.account_id,
        ax.event AS ax_event,
        w.event_name,
        ax.account_transaction_id,
        w.status,
        axd.merchant_store_id AS store_code,
        axd.billreceipt,
        'EARN' AS point_type,
        (CASE
            WHEN ax.event = 'REFUND_DEBIT' THEN -1
            WHEN ax.event IN ('EARN','CREDIT') THEN 1
            WHEN ax.event = 'DEBIT' THEN -1
         END * ax.value) AS ax_value
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx ON w.id = wx.id
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid = ax.account_transaction_id
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id = axd.account_transaction_id
    LEFT JOIN refined.d_account d ON ax.account_id = d.account_id
    WHERE w.transaction_time >= '2025-02-27'
      AND CASE
             WHEN w.event_name = 'POSCONNECT.WALLET.SETTLE' THEN 1
             WHEN w.event_name = 'POSCONNECT.WALLET.REFUND' AND w.type = 'REFUND' THEN 1
             ELSE 0
          END = 1
      AND d.type = 'POINTS'
      AND axd.billreceipt IS NOT NULL

    UNION ALL

    -- Branch STAFF_EARN: CRV staff points (reference prefix 20/21/22/23)
    SELECT
        DATE(w.event_date) AS event_date,
        w.wallet_id,
        ax.account_id,
        ax.event AS ax_event,
        w.event_name,
        ax.account_transaction_id,
        w.status,
        'THE1' AS store_code,
        axd.billreceipt,
        'STAFF_EARN' AS point_type,
        (CASE WHEN ax.event = 'CREDIT' THEN 1 WHEN ax.event = 'DEBIT' THEN -1 END * ax.value) AS ax_value
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx ON w.id = wx.id
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid = ax.account_transaction_id
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id = axd.account_transaction_id
    LEFT JOIN refined.d_wallet_transaction_meta wm
        ON wm.wallet_transaction_id = w.wallet_transaction_id
       AND wm.key = 'transactiondescription' AND wm.operation_type = 'UPDATE'
    WHERE DATE(w.event_date) >= '2025-02-27'
      AND ax.account_id IS NOT NULL
      AND w.event_name = 'SERVICE.WALLET.BACKENDPOINTS'
      AND LEFT((CASE
                    WHEN w.reference LIKE '20250318%' THEN SPLIT_PART(w.reference, '_', 4)
                    ELSE SPLIT_PART(w.reference, '_', 3)
                END), 2) IN ('20', '21', '22', '23')

    UNION ALL

    -- Branch ADJUSTMENT: CS/ops adjustment (reference code 101,105,107,108,109,110)
    SELECT DISTINCT
        DATE(a.event_date) AS event_date,
        d.wallet_id,
        a.account_id,
        a.event AS ax_event,
        a.event_name,
        a.account_transaction_id,
        d.status,
        d.location_storeid AS store_code,
        '' AS billreceipt,
        'ADJUSTMENT' AS point_type,
        (CASE WHEN a.event = 'CREDIT' THEN a.value ELSE -a.value END) AS ax_value
    FROM refined.f_wallet_account_transaction a
    JOIN refined.f_wallet_transaction_account b ON a.account_transaction_id = b.account_acccounttransactionid
    JOIN refined.d_wallet_transaction_meta c ON b.wallet_transaction_id = c.wallet_transaction_id
    JOIN refined.f_wallet_transaction d ON b.wallet_transaction_id = d.wallet_transaction_id
    WHERE a.event_name IN ('SERVICE.WALLET.BACKENDPOINTS')
      AND SPLIT_PART(d.reference, '_', 3) IN ('101', '105', '107', '108', '109', '110')
      AND c.key = 'transactiondescription'
),

d_consolidate AS (
    -- EXCHANGE: exchange points for Cash Coupon
    SELECT
        d.date_created::date AS event_date,
        'THE1' AS store_code,
        'EXCHANGE' AS type,
        d.wallet_id,
        1 AS txn_count,
        SUM(COALESCE(dam.value::numeric, 0) * CASE WHEN d.status = 'CANCELLED' THEN 0 ELSE 1 END) AS point_value
    FROM refined.d_account d
    JOIN exchange_camp ec ON d.campaign_id = ec.campaign_id
    LEFT JOIN refined.d_account_meta dam ON d.account_id = dam.account_id AND dam.key = 'debitpointamount'
    WHERE d.date_created::date >= '2025-02-27'
      AND ec.campaign_id NOT IN ('90064586')
    GROUP BY d.date_created::date, d.wallet_id, d.account_id

    UNION ALL

    -- PAY WITH POINT: tender_type='90'
    SELECT
        sa.business_date,
        sa.branch_code AS store_code,
        'PAY WITH POINT' AS type,
        dci.wallet_id,
        COUNT(DISTINCT sa.display_receipt_no) AS txn_count,
        SUM(sa.net_price_tot) AS point_value
    FROM refined.sales_tender_all sa
    LEFT JOIN report.d_customer_info dci ON sa.card_no = dci.customer_id
    WHERE sa.business_date >= '2025-02-27'
      AND sa.tender_type = '90'
    GROUP BY sa.business_date, sa.branch_code, dci.wallet_id

    UNION ALL

    -- EARN: from e_point CTE
    SELECT
        e.event_date, e.store_code, 'EARN' AS type, e.wallet_id,
        COUNT(DISTINCT e.billreceipt) AS txn_count,
        SUM(e.ax_value) AS point_value
    FROM e_point e
    WHERE e.point_type = 'EARN' AND e.billreceipt IS NOT NULL
    GROUP BY e.event_date, e.store_code, e.wallet_id, e.account_id

    UNION ALL

    -- STAFF_EARN
    SELECT
        e.event_date, 'THE1' AS store_code, 'EARN' AS type, e.wallet_id,
        COUNT(DISTINCT e.account_transaction_id) AS txn_count,
        SUM(e.ax_value) AS point_value
    FROM e_point e
    WHERE e.point_type = 'STAFF_EARN'
    GROUP BY e.event_date, e.store_code, e.wallet_id, e.account_id

    UNION ALL

    -- USED AS VOUCHER: tender_type='67'
    SELECT
        sa.business_date,
        sa.branch_code AS store_code,
        'USED AS VOUCHER' AS type,
        dci.wallet_id,
        COUNT(DISTINCT dt.token) AS txn_count,
        SUM(CASE WHEN sa.trans_type = '1' THEN dam.value::numeric ELSE -dam.value::numeric END) AS point_value
    FROM refined.sales_tender_all sa
    LEFT JOIN report.d_customer_info dci ON sa.card_no = dci.customer_id
    LEFT JOIN refined.d_token dt ON sa.tender_ref_no = dt.token
    JOIN refined.d_account_meta dam ON dt.account_id = dam.account_id
    WHERE sa.business_date >= '2025-02-27'
      AND sa.tender_type = '67'
      AND dam.key = 'debitpointamount'
    GROUP BY sa.business_date, sa.branch_code, dci.wallet_id

    UNION ALL

    -- ADJUSTMENT
    SELECT
        e.event_date, e.store_code, 'ADJUSTMENT' AS type, e.wallet_id,
        COUNT(DISTINCT e.account_transaction_id) AS txn_count,
        SUM(e.ax_value) AS point_value
    FROM e_point e
    WHERE e.point_type = 'ADJUSTMENT'
      AND e.event_date >= '2025-02-27'
    GROUP BY e.event_date, e.store_code, e.wallet_id, e.account_id
)

SELECT event_date, store_code, type, wallet_id, txn_count, point_value
FROM d_consolidate;
```

#### 5. VOID\_RETURN\_MONITOR — Detect Abnormal Void/Return Behavior

> **Output:** `(event_date, store_code, wallet_id, card_no, ax_event, trx_count, ax_value)` per member/date/event.\
> ⚠️ Does not filter `d.type = 'POINTS'` — add `AND d.type = 'POINTS'` if pure points needed.

```sql
WITH trx_summary AS (
    SELECT
        DATE(w.event_date)              AS event_date,
        w.location_storeid              AS store_code,
        w.wallet_id,
        dc.customer_id                  AS card_no,
        ax.event                        AS ax_event,
        COUNT(DISTINCT axd.billreceipt) AS trx_count,
        SUM(ax.value)                   AS ax_value
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx ON w.id = wx.id
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid = ax.account_transaction_id
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id = axd.account_transaction_id
    LEFT JOIN refined.d_account d ON ax.account_id = d.account_id
    JOIN report.d_customer_info dc ON w.wallet_id = dc.wallet_id
    WHERE w.event_stored_at >= '2025-02-27'
      AND CASE
              WHEN ax.event IN ('DEBIT', 'REFUND_DEBIT') THEN 1
              WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' AND w.status = 'VOIDED' THEN 1
              ELSE 0
          END = 1
      AND axd.billreceipt IS NOT NULL
    GROUP BY DATE(w.event_date), w.location_storeid, w.wallet_id, dc.customer_id, ax.event
)
SELECT * FROM trx_summary ORDER BY event_date, card_no;
```

#### 6. v\_loyalty\_transaction — Hardcoded Date/Wallet/Event Filter (Best Performance)

> **Use when:** querying full wallet event history with date/wallet/event filters.\
> **Performance note:** Hardcode values directly in WHERE — do NOT use `CROSS JOIN params` or CTE-based filters as PostgreSQL cannot push them down to index scans.\
> **How to use:** edit the marked lines only — comment out wallet/event filters if not needed.

```sql
WITH f_event AS (

    -- Branch 1: Regular wallet transactions (wx + ax)
    SELECT
        w.wallet_id,
        w.wallet_transaction_id,
        NULLIF(w.parent_wallet_transaction_id::text, '0') AS parent_wx_id,
        w.event_name,
        w."type" AS wx_type,
        w.status AS wx_status,
        ax.account_transaction_id,
        ax.parent_account_transaction_id AS parent_ax_id,
        ax.account_id,
        ax.event AS ax_event,
        ax.value AS ax_value,
        COALESCE(ax.balance_before_current, ax.balance_before_available, ax.balance_before_transactioncount) AS balance_before_current,
        COALESCE(
            CASE WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' THEN ax.value ELSE 0 END
            + ax.balance_after_current - ax.balance_before_current,
            ax.balance_after_available - ax.balance_before_available,
            ax.balance_after_transactioncount - ax.balance_before_transactioncount
        ) AS balance_change,
        COALESCE(
            CASE WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' THEN ax.value ELSE 0 END
            + ax.balance_after_current,
            ax.balance_after_available,
            ax.balance_after_transactioncount
        ) AS balance_after_current,
        axd.billreceipt AS orig_bill_receipt,
        axd.amount AS orig_bill_amount,
        axd.merchant_store_id AS store_id,
        w.summary_total_qualifying_amount_base_earn AS amount_for_base_earn,
        axd.total_used_points AS orig_total_used_points,
        w.reference AS wx_reference,
        NULL AS campaign_id,
        COALESCE(ax.event_id, w.event_id) AS event_id,
        COALESCE(ax.event_stored_at, w.event_stored_at) AS event_stored_at
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx ON w.id::text = wx.id::text
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid::text = ax.account_transaction_id::text
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id::text = axd.account_transaction_id::text
    WHERE w.event_stored_at >= '2026-04-08'          -- ✏️ p_date_from
      AND w.event_stored_at <  '2026-04-09'          -- ✏️ p_date_to
      -- AND w.wallet_id   = '93827415'              -- ✏️ uncomment to filter wallet
      -- AND w.event_name  = 'POSCONNECT.WALLET.SETTLE' -- ✏️ uncomment to filter event
      AND CASE
            WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' AND ax.event = 'SPEND' THEN 0
            WHEN w.event_name = 'POSCONNECT.WALLET.REFUND' AND w.TYPE = 'SETTLE' THEN 0
            WHEN LENGTH(w.account::text) < 3 THEN 0
          ELSE 1
          END = 1

    UNION ALL

    -- Branch 2: ECOUPON
    SELECT
        da.wallet_id,
        NULL AS wallet_transaction_id,
        NULL AS parent_wx_id,
        ax.event_name,
        da.client_type AS wx_type,
        da.status AS wx_status,
        ax.account_transaction_id,
        dam.value AS parent_ax_id,
        da.account_id,
        da.state AS ax_event,
        (da."overrides" -> 'Offer' -> 'Reward' ->> 'DiscountAmount')::bigint AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS orig_total_used_points,
        dt.token AS wx_reference,
        da.campaign_id,
        ax.event_id,
        da.date_created AS event_stored_at
    FROM refined.d_account da
    JOIN refined.d_campaign dc ON da.campaign_id::text = dc.campaign_id::text AND da.TYPE = 'ECOUPON'
    LEFT JOIN refined.d_token dt ON da.account_id::text = dt.account_id::text
    LEFT JOIN refined.d_account_meta dam
        ON da.account_id::text = dam.account_id::text AND dam.key = 'accounttransactionid'
    LEFT JOIN refined.f_wallet_account_transaction ax ON da.account_id::text = ax.account_id::text
    WHERE ax.event_name = 'WALLET.ACCOUNT.CREATE.CAMPAIGN'
      AND ax.event = 'CREATE'
      AND ax.event_stored_at >= '2026-04-08'         -- ✏️ p_date_from
      AND ax.event_stored_at <  '2026-04-09'         -- ✏️ p_date_to
      -- AND da.wallet_id  = '93827415'              -- ✏️ uncomment to filter wallet

    UNION ALL

    -- Branch 3: UNREDEEM
    SELECT
        da.wallet_id,
        NULL AS wallet_transaction_id,
        NULL AS parent_wx_id,
        ax.event_name,
        NULL AS wx_type,
        NULL AS wx_status,
        ax.account_transaction_id,
        ax.parent_account_transaction_id AS parent_ax_id,
        ax.account_id,
        ax.event AS ax_event,
        ax.value AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS total_used_points,
        dt.token AS wx_reference,
        NULL AS campaign_id,
        ax.event_id,
        ax.event_stored_at
    FROM refined.f_wallet_account_transaction ax
    LEFT JOIN refined.d_account da ON ax.account_id::text = da.account_id::text
    LEFT JOIN refined.d_token dt ON da.account_id::text = dt.account_id::text
    WHERE ax.event_name = 'POSCONNECT.ACCOUNT.UNREDEEM'
      AND ax.event_stored_at >= '2026-04-08'         -- ✏️ p_date_from
      AND ax.event_stored_at <  '2026-04-09'         -- ✏️ p_date_to
      -- AND da.wallet_id  = '93827415'              -- ✏️ uncomment to filter wallet
)
SELECT
    wallet_transaction_id, parent_wx_id, account_transaction_id, parent_ax_id,
    e.wallet_id, e.account_id, e.event_stored_at, e.event_name,
    wx_type, ax_event, wx_status, ax_value,
    balance_before_current, balance_change, balance_after_current,
    orig_bill_receipt, orig_bill_amount, amount_for_base_earn, orig_total_used_points,
    wx_reference, dc.campaign_id, dc.class, dc.details, event_id, store_id
FROM f_event e
LEFT JOIN refined.d_account da  ON e.account_id = da.account_id
LEFT JOIN refined.d_campaign dc ON da.campaign_id = dc.campaign_id
LIMIT 5;
```

***

### 🚀 Master Query: Full Wallet History (report.v\_loyalty\_transaction pattern)

> ⚠️ `v_loyalty_transaction` **does NOT exist** in DB. Use this CTE pattern instead.\
> ⚠️ Final SELECT **must NOT join `d_account` again** — this causes fan-out because 1 wallet has multiple accounts. `d_campaign` is joined directly via `da.campaign_id` already present in f\_event Branch 2.\
> ⚠️ **When filtering a single wallet:** add `WHERE w.wallet_id = ':wallet_id'` inside Branch 1, and `WHERE da.wallet_id = ':wallet_id'` inside Branch 2+3. **Do NOT filter wallet\_id in Final SELECT** — the `d_account` join there causes fan-out.

```sql
-- report.v_loyalty_transaction
WITH f_event AS (
    SELECT
        w.wallet_id,
        w.wallet_transaction_id,
        NULLIF(w.parent_wallet_transaction_id::text, '0') AS parent_wx_id,
        w.event_name,
        w."type" AS wx_type,
        w.status AS wx_status,
        ax.account_transaction_id,
        ax.parent_account_transaction_id AS parent_ax_id,
        ax.account_id,
        ax.event AS ax_event,
        ax.value AS ax_value,
        COALESCE(ax.balance_before_current, ax.balance_before_available, ax.balance_before_transactioncount) AS balance_before_current,
        COALESCE(
            CASE
                WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' THEN ax.value
                ELSE 0
            END
            + ax.balance_after_current - ax.balance_before_current,
            ax.balance_after_available - ax.balance_before_available,
            ax.balance_after_transactioncount - ax.balance_before_transactioncount
        ) AS balance_change,
        COALESCE(
            CASE
                WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' THEN ax.value
                ELSE 0
            END + ax.balance_after_current,
            ax.balance_after_available,
            ax.balance_after_transactioncount
        ) AS balance_after_current,
        axd.billreceipt AS orig_bill_receipt,
        axd.amount AS orig_bill_amount,
        axd.merchant_store_id AS store_id,
        w.summary_total_qualifying_amount_base_earn AS amount_for_base_earn,
        axd.total_used_points AS orig_total_used_points,
        w.reference AS wx_reference,
        NULL AS campaign_id,
        COALESCE(ax.event_id, w.event_id) AS event_id,
        COALESCE(ax.event_stored_at, w.event_stored_at) AS event_stored_at
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx
        ON w.id::text = wx.id::text
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid::text = ax.account_transaction_id::text
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id::text = axd.account_transaction_id::text
    WHERE 1 = 1
      AND CASE
            WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' AND ax.event = 'SPEND' THEN 0
            WHEN w.event_name = 'POSCONNECT.WALLET.REFUND' AND w.TYPE = 'SETTLE' THEN 0
            WHEN LENGTH(w.account::text) < 3 THEN 0
          ELSE 1
          END = 1

    UNION ALL

    SELECT
        da.wallet_id,
        NULL AS wallet_transaction_id,
        NULL AS parent_wx_id,
        ax.event_name,
        da.client_type AS wx_type,
        da.status AS wx_status,
        ax.account_transaction_id,
        dam.value AS parent_ax_id,
        da.account_id,
        da.state AS ax_event,
        (da."overrides" -> 'Offer' -> 'Reward' ->> 'DiscountAmount')::bigint AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS orig_total_used_points,
        dt.token AS wx_reference,
        da.campaign_id,
        ax.event_id,
        da.date_created AS event_stored_at
    FROM refined.d_account da
    JOIN refined.d_campaign dc
        ON da.campaign_id::text = dc.campaign_id::text
       AND da.TYPE = 'ECOUPON'
    LEFT JOIN refined.d_token dt
        ON da.account_id::text = dt.account_id::text
    LEFT JOIN refined.d_account_meta dam
        ON da.account_id::text = dam.account_id::text
       AND dam.key = 'accounttransactionid'
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON da.account_id::text = ax.account_id::text
    WHERE 1 = 1
      AND ax.event_name = 'WALLET.ACCOUNT.CREATE.CAMPAIGN'
      AND ax.event = 'CREATE'

    UNION ALL

    SELECT
        da.wallet_id,
        NULL AS wallet_transaction_id,
        NULL AS parent_wx_id,
        ax.event_name,
        NULL AS wx_type,
        NULL AS wx_status,
        ax.account_transaction_id,
        ax.parent_account_transaction_id AS parent_ax_id,
        ax.account_id,
        ax.event AS ax_event,
        ax.value AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS total_used_points,
        dt.token AS wx_reference,
        NULL AS campaign_id,
        ax.event_id,
        ax.event_stored_at
    FROM refined.f_wallet_account_transaction ax
    LEFT JOIN refined.d_account da
        ON ax.account_id::text = da.account_id::text
    LEFT JOIN refined.d_token dt
        ON da.account_id::text = dt.account_id::text
    WHERE 1 = 1
      AND ax.event_name = 'POSCONNECT.ACCOUNT.UNREDEEM'
)
SELECT
    wallet_transaction_id,
    parent_wx_id,
    account_transaction_id,
    parent_ax_id,
    e.wallet_id,
    e.account_id,
    e.event_stored_at,
    e.event_name,
    wx_type,
    ax_event,
    wx_status,
    ax_value,
    balance_before_current,
    balance_change,
    balance_after_current,
    orig_bill_receipt,
    orig_bill_amount,
    amount_for_base_earn,
    orig_total_used_points,
    wx_reference,
    dc.campaign_id,
    dc.class,
    dc.details,
    event_id,
    store_id
FROM f_event e
LEFT JOIN refined.d_account da
    ON e.account_id = da.account_id
LEFT JOIN refined.d_campaign dc
    ON da.campaign_id = dc.campaign_id
LIMIT 5;
```

```
```
