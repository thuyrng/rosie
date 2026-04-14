# CDP | 3 | Event Reference + Points Query Rules

> Pull this when the question involves points, earn, spend, refund, expire, vouchers, or transaction event flow.  

---

## 💡 Two Levels — Which to Use

Every loyalty transaction generates data at two levels simultaneously:

wx level (f_wallet_transaction) — one row per POS receipt. Has pre-aggregated summary columns (summary_result_points_earn etc.) that are fast but hide breakdown. Cannot tell EARN vs CREDIT from wx alone, no balance snapshots.

ax level (f_wallet_account_transaction) — multiple rows per receipt, one per individual point movement. Each row has a specific event type, the exact value moved, and balance_before_* / balance_after_* snapshots. This is the raw source of truth.

Example — same receipt, two views:

```javascript
wx level (1 row):
  summary_result_points_earn = 3000   ← total, no breakdown

ax level (3 rows):
  event = 'EARN'   value = 2500       ← base earn from purchase
  event = 'CREDIT' value = 500        ← points from redeemed vouchers credited back
  event = 'REDEEM' value = -1000      ← ECOUPON redeemed (NOT a point event)
```

> ⚠️ POSCONNECT.WALLET.SETTLE also generates ax_event='REDEEM' for ECOUPON accounts — without da.type='POINTS' filter, voucher rows will mix into point totals.

---

## ✅ Valid Event Pairs for POINTS

Mandatory filter — always include when querying points:

```sql
JOIN refined.d_account da ON ax.account_id::text = da.account_id::text
WHERE da.type = 'POINTS'
  AND ax.event_name IN (
      'POSCONNECT.WALLET.SETTLE',
      'POSCONNECT.WALLET.REFUND',
      'POSCONNECT.WALLET.SPEND',
      'POSCONNECT.WALLET.SPEND.VOID',
      'SERVICE.WALLET.TRANSACTION.DEBIT',
      'SERVICE.WALLET.TRANSACTION.CREDIT',
      'SERVICE.WALLET.BACKENDPOINTS',
      'SERVICE.WALLET.POINTS.EXPIRE'
  )
  AND ax.event IN ('EARN','CREDIT','DEBIT','SPEND','VOID','REFUND_DEBIT','EXPIRY')
```

---

## 🧠 Event Reference Table

> For understanding meaning only — do NOT use to derive queries. Use SQL Snippets entry instead.

> 🟡 = POINTS event — must filter da.type = 'POINTS'

Transaction flow summary:

- Normal purchase: SETTLE → REDEEM×N (ECOUPON) + EARN + CREDIT (POINTS)
- Purchase + Cash Voucher: CREATE(CASH_COUPON) → DEBIT → REDEEM_SETTLE → SETTLE+EARN
- Refund: REFUND → UNREDEEM + REFUND_DEBIT×2 + EARN + REDEEM + CREDIT
