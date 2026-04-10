# CDP | 6 | Index & Partition Reference

## Partition Strategy Overview

Two partitioning mechanisms are used across the CDP database:   

> Important: par_month + par_day indexes are composite. Filtering par_day alone will not use the index. Always include both.

---

## Schema: refined

---

### refined.d_account

Partition columns: par_month, par_day

---

### refined.d_account_meta

Partition columns: par_month, par_day

---

### refined.d_campaign

Partition columns: par_month, par_day (no dedicated partition index — use date_created or campaign_id)

---

### refined.d_consumer

Partition columns: par_month, par_day

---

### refined.d_consumer_contact

Partition columns: par_month, par_day

---

### refined.d_consumer_dimension

Partition columns: par_month, par_day

---

### refined.d_consumer_meta

Partition columns: par_month, par_day

---

### refined.d_identity

Partition columns: par_month, par_day

---

### refined.d_identity_meta

Partition columns: par_month, par_day

---

### refined.d_mer_structure

Partition columns: par_month, par_day

JOIN rule: Always join on cate_id = id AND partner_code = branch_code

---

### refined.d_sales_channel

Partition columns: par_month, par_day

---

### refined.d_store_master

Partition columns: par_month, par_day

---

### refined.d_tender

Partition columns: par_month, par_day

---

### refined.d_token

Partition columns: par_month, par_day

---

### refined.d_wallet

Partition columns: par_month, par_day

---

### refined.d_wallet_meta

Partition columns: par_month, par_day

---

### refined.d_wallet_transaction_meta

Partition columns: par_month, par_day

---

### refined.f_account_transaction_detail

Partition columns: par_month, par_day

---

### refined.f_account_create_campaign

(No standalone indexes beyond partition columns)

Partition columns: par_month, par_day

---

### refined.f_ref_voucher

Partition columns: par_month, par_day

---

### refined.f_staff_point

Partition columns: par_month, par_day

---

### refined.f_wallet_account_transaction

Partition type: TimescaleDB hypertable — auto-chunked by event_stored_at (~30-day intervals)

Partition columns: event_stored_at (TimescaleDB) + par_month, par_day (custom)

Query strategy: Filter account_id or account_transaction_id for single-wallet lookups; use event_stored_at range for time-based aggregations.

---

### refined.f_wallet_transaction

Partition type: TimescaleDB hypertable — auto-chunked by event_stored_at (~30-day intervals)

