# CDP | 4 | SQL Snippets

> 🔴 MUST read this entry before writing any CDP query. Match business question to a snippet below and use exact pattern. Do NOT derive from Event Reference Table.

🧭 QUERY ROUTER — Đọc bảng này để chọn đúng snippet

Business question → Which snippet to use:

• Find a customer (phone / email / card / wallet_id) → Snippet 1: Customer Lookup
• Revenue, sales, net_price_tot by date or store → Snippet 2: Revenue / Sales
• Trace transaction history of a specific wallet, debug a transaction → Snippet 3: Params CTE or Snippet 5: Master Wallet History
• Total points EARNED in a day / month → Snippet 4 → branch e_point (event = EARN)
• Points EXCHANGED for a Cash Coupon / Voucher → Snippet 4 → branch e_exchange (DEBIT)
• Points USED TO PAY at POS (Pay with Point) → Snippet 4 → branch e_pwp (tender_type = '90')
• Voucher USED at checkout (Used as Voucher) → Snippet 4 → branch e_voucher (tender_type = '67')
• Full event timeline for one wallet (all event types) → Snippet 5: Master Wallet History (3-branch CTE)

⛔ If you only need to understand the meaning of event_name / ax_event → read CDP | 3. Do NOT derive queries from CDP | 3.

---

## 1. Customer Lookup

```sql
SELECT consumer_id, wallet_id, first_name, last_name, phone_number, email
FROM report.d_customer_info
WHERE phone_number = ':phone';
-- or: WHERE email = ':email'
-- or: WHERE wallet_id = ':wallet_id'
-- or: WHERE customer_id = ':card_no'  (THE1 card number)
```

---

## 2. Revenue / Sales by Store

```sql
SELECT business_date, partner_code, SUM(net_price_tot) AS revenue
FROM refined.sales_receipt_all
WHERE partner_code = 'TPS'                                  -- index: partner_code
  AND business_date BETWEEN '2026-01-01' AND '2026-01-31'  -- index: business_date
GROUP BY business_date, partner_code;
```

---

## 3. Params CTE Pattern — Wallet Transactions by Date Range

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
    JOIN params p ON w.wallet_id = p.p_wallet_id            -- index: wallet_id
    WHERE w.event_stored_at BETWEEN p.p_date_from AND p.p_date_to  -- index: event_stored_at
),
ecoupon AS (
    SELECT da.wallet_id
    FROM refined.d_account da
    JOIN params p ON da.wallet_id = p.p_wallet_id           -- index: wallet_id
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

---

## 4. Point Ledger — Daily/Store Point Movement Summary

> Use for summarizing point movements across many wallets. Do NOT use for single wallet debugging — use Snippet 6 (Master Wallet History) instead.

Branch reference:

> ⚠️ EARN uses w.transaction_time. STAFF_EARN uses w.event_date. Different columns.

> ⚠️ reason_code in f_staff_point is varchar → IN ('201','202',...) NOT IN (201,202,...).

```sql
WITH exchange_camp AS (
    SELECT dc.*,
        json_extract_path_text(dc.offer::json, 'Reward', 'Standard', 'Value', 'DiscountAmount')::real AS discount_amt
    FROM refined.d_campaign dc
    WHERE json_extract_path_text(dc.offer::json, 'Reward', 'Standard', 'Value', 'DiscountAmount') IS NOT NULL
),
e_point AS (
    SELECT DATE(w.transaction_time) AS event_date, w.wallet_id, ax.account_id,
        ax.event AS ax_event, w.event_name, w.location_storeid AS store_code,
        ax.value AS point_value, 'EARN' AS type
    FROM refined.f_wallet_transaction w
    JOIN refined.f_wallet_transaction_account wxa ON w.id::text = wxa.id::text
    JOIN refined.f_wallet_account_transaction ax ON wxa.account_acccounttransactionid::text = ax.account_transaction_id::text
    JOIN refined.d_account d ON ax.account_id::text = d.account_id::text
    WHERE w.event_name IN ('POSCONNECT.WALLET.SETTLE', 'POSCONNECT.WALLET.REFUND')
      AND d.type = 'POINTS'
      AND ax.event IN ('EARN', 'CREDIT', 'REFUND_DEBIT')
      AND w.transaction_time >= ':date_from'   -- ⚙️ replace
      AND w.transaction_time <  ':date_to'
),
e_staff AS (
    SELECT DATE(w.event_date) AS event_date, w.wallet_id, NULL AS account_id,
        NULL AS ax_event, w.event_name, 'THE1' AS store_code,
        w.summary_result_points_earn AS point_value, 'STAFF_EARN' AS type
    FROM refined.f_wallet_transaction w
    WHERE w.event_name = 'SERVICE.WALLET.BACKENDPOINTS'
      AND SPLIT_PART(w.reference, '-', 1) IN ('20','21','22','23')
      AND DATE(w.event_date) >= ':date_from'
      AND DATE(w.event_date) <  ':date_to'
),
e_exchange AS (
    SELECT d.date_created::date AS event_date, d.wallet_id, d.account_id,
        NULL AS ax_event, NULL AS event_name, NULL AS store_code,
        CASE WHEN d.status = 'CANCELLED' THEN 0
             ELSE COALESCE(dam.value::bigint, 0) END AS point_value,
        'EXCHANGE' AS type
    FROM refined.d_account d
    JOIN exchange_camp ec ON d.campaign_id::text = ec.campaign_id::text
    LEFT JOIN refined.d_account_meta dam ON d.account_id::text = dam.account_id::text AND dam.key = 'debitpointamount'
    WHERE d.date_created::date >= ':date_from'
      AND d.date_created::date <  ':date_to'
      AND d.campaign_id != '90064586'
),
e_pwp AS (
    SELECT st.business_date AS event_date, ci.wallet_id, NULL AS account_id,
        NULL AS ax_event, NULL AS event_name, st.branch_code AS store_code,
        st.net_price_tot AS point_value, 'PAY_WITH_POINT' AS type
    FROM refined.sales_tender_all st
    JOIN report.d_customer_info ci ON st.card_no = ci.customer_id
    WHERE st.tender_type = '90'
      AND st.partner_code = ':partner_code'          -- ⚙️ replace
      AND st.business_date BETWEEN ':date_from' AND ':date_to'
),
e_voucher AS (
    SELECT st.business_date AS event_date, ci.wallet_id, d.account_id,
        NULL AS ax_event, NULL AS event_name, st.branch_code AS store_code,
        CASE WHEN st.trans_type IN ('2','3') THEN -COALESCE(dam.value::bigint, 0)
             ELSE COALESCE(dam.value::bigint, 0) END AS point_value,
        'USED_AS_VOUCHER' AS type
    FROM refined.sales_tender_all st
    JOIN report.d_customer_info ci ON st.card_no = ci.customer_id
    JOIN refined.d_account d ON ci.wallet_id = d.wallet_id AND d.type = 'ECOUPON'
    LEFT JOIN refined.d_account_meta dam ON d.account_id::text = dam.account_id::text AND dam.key = 'debitpointamount'
    WHERE st.tender_type = '67'
      AND st.partner_code = ':partner_code'
      AND st.business_date BETWEEN ':date_from' AND ':date_to'
      AND st.trans_type = '1'
)
SELECT * FROM e_point
UNION ALL SELECT * FROM e_staff
UNION ALL SELECT * FROM e_exchange
UNION ALL SELECT * FROM e_pwp
UNION ALL SELECT * FROM e_voucher
ORDER BY event_date, wallet_id;
```

---

---

## 5. Master Wallet History (Single Wallet Trace)

> Full event timeline for one wallet. 3-branch UNION CTE covering all event types.

> ⚠️ v_loyalty_transaction does NOT exist — use this CTE.

Branches:

1. Branch 1 — Regular wx+ax (SETTLE, SPEND, REFUND, DEBIT, CREDIT, BACKENDPOINTS, EXPIRE...): joined via f_wallet_transaction_account bridge.
1. Branch 2 — ECOUPON creation from campaign (WALLET.ACCOUNT.CREATE.CAMPAIGN): from d_account + d_token.
1. Branch 3 — Voucher cancellation (POSCONNECT.ACCOUNT.UNREDEEM): from f_wallet_account_transaction + d_account.
All branches push wallet_id = ':wallet_id' into each CTE for index efficiency.

Key output columns: wallet_transaction_id, account_transaction_id, event_name, wx_type, ax_event, ax_value, balance_before_current, balance_change, balance_after_current, orig_bill_receipt, orig_bill_amount, store_id, campaign_id, event_stored_at.

Branch 1 excludes: POSCONNECT.WALLET.SPEND.VOID rows where ax.event='SPEND' · POSCONNECT.WALLET.REFUND rows where w.type='SETTLE' · rows where LENGTH(w.account::text) < 3.

```sql
WITH f_event AS (

    -- Branch 1: wx + ax (SETTLE, SPEND, REFUND, DEBIT, CREDIT, BACKENDPOINTS, EXPIRE...)
    SELECT
        w.wallet_id,
        w.wallet_transaction_id,
        w.parent_wallet_transaction_id                        AS parent_wx_id,
        w.event_name,
        w.type                                                AS wx_type,
        w.status                                              AS wx_status,
        ax.account_transaction_id,
        ax.parent_account_transaction_id                      AS parent_ax_id,
        ax.account_id,
        ax.event                                              AS ax_event,
        ax.value                                              AS ax_value,
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
        axd.billreceipt                                       AS orig_bill_receipt,
        axd.amount                                            AS orig_bill_amount,
        axd.merchant_store_id                                 AS store_id,
        w.summary_total_qualifying_amount_base_earn           AS amount_for_base_earn,
        axd.total_used_points                                 AS orig_total_used_points,
        w.reference                                           AS wx_reference,
        NULL                                                  AS campaign_id,
        COALESCE(ax.event_id, w.event_id)                    AS event_id,
        COALESCE(ax.event_stored_at, w.event_stored_at)      AS event_stored_at
    FROM refined.f_wallet_transaction w
    LEFT JOIN refined.f_wallet_transaction_account wx
        ON w.id::text = wx.id::text
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON wx.account_acccounttransactionid::text = ax.account_transaction_id::text
       AND w.event_name = ax.event_name
    LEFT JOIN refined.f_account_transaction_detail axd
        ON ax.account_transaction_id::text = axd.account_transaction_id::text
    WHERE w.wallet_id = ':wallet_id'
      AND CASE
            WHEN w.event_name = 'POSCONNECT.WALLET.SPEND.VOID' AND ax.event = 'SPEND' THEN 0
            WHEN w.event_name = 'POSCONNECT.WALLET.REFUND' AND w.type = 'SETTLE' THEN 0
            WHEN LENGTH(w.account::text) < 3 THEN 0
            ELSE 1
          END = 1

    UNION ALL

    -- Branch 2: ECOUPON creation from campaign
    SELECT
        da.wallet_id,
        NULL AS wallet_transaction_id,
        NULL AS parent_wx_id,
        ax.event_name,
        da.client_type                                                        AS wx_type,
        da.status                                                             AS wx_status,
        ax.account_transaction_id,
        dam.value                                                             AS parent_ax_id,
        da.account_id,
        da.state                                                              AS ax_event,
        (da."overrides" -> 'Offer' -> 'Reward' ->> 'DiscountAmount')::bigint AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS orig_total_used_points,
        dt.token                                                              AS wx_reference,
        da.campaign_id,
        ax.event_id,
        da.date_created                                                       AS event_stored_at
    FROM refined.d_account da
    JOIN refined.d_campaign dc
        ON da.campaign_id::text = dc.campaign_id::text
       AND da.type = 'ECOUPON'
    LEFT JOIN refined.d_token dt
        ON da.account_id::text = dt.account_id::text
    LEFT JOIN refined.d_account_meta dam
        ON da.account_id::text = dam.account_id::text
       AND dam.key = 'accounttransactionid'
    LEFT JOIN refined.f_wallet_account_transaction ax
        ON da.account_id::text = ax.account_id::text
    WHERE da.wallet_id = ':wallet_id'
      AND ax.event_name = 'WALLET.ACCOUNT.CREATE.CAMPAIGN'
      AND ax.event = 'CREATE'

    UNION ALL

    -- Branch 3: Voucher cancellation (POSCONNECT.ACCOUNT.UNREDEEM)
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
        ax.event                         AS ax_event,
        ax.value                         AS ax_value,
        NULL AS balance_before_current,
        NULL AS balance_change,
        NULL AS balance_after_current,
        NULL AS orig_bill_receipt,
        NULL AS orig_bill_amount,
        NULL AS store_id,
        NULL AS amount_for_base_earn,
        NULL AS orig_total_used_points,
        dt.token                         AS wx_reference,
        NULL                             AS campaign_id,
        ax.event_id,
        ax.event_stored_at
    FROM refined.f_wallet_account_transaction ax
    LEFT JOIN refined.d_account da
        ON ax.account_id::text = da.account_id::text
    LEFT JOIN refined.d_token dt
        ON da.account_id::text = dt.account_id::text
    WHERE da.wallet_id = ':wallet_id'
      AND ax.event_name = 'POSCONNECT.ACCOUNT.UNREDEEM'
)

SELECT
    wallet_transaction_id, parent_wx_id,
    account_transaction_id, parent_ax_id,
    e.wallet_id, e.account_id,
    e.event_stored_at, e.event_name,
    wx_type, ax_event, wx_status, ax_value,
    balance_before_current, balance_change, balance_after_current,
    orig_bill_receipt, orig_bill_amount,
    amount_for_base_earn, orig_total_used_points,
    wx_reference, dc.campaign_id, dc.class, dc.details,
    event_id, store_id
FROM f_event e
LEFT JOIN refined.d_account da  ON e.account_id = da.account_id
LEFT JOIN refined.d_campaign dc ON da.campaign_id = dc.campaign_id
ORDER BY e.event_stored_at;
```

