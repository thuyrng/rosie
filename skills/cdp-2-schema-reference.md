# CDP | 2 | Schema Reference

# Table of Contents

## Schema: refined  

Dimension tables
1. d_account
2. d_account_meta
3. d_campaign
4. d_consumer
5. d_consumer_address
6. d_consumer_contact
7. d_consumer_dimension
8. d_consumer_meta
9. d_consumer_segmentation
10. d_identity
11. d_identity_deleted
12. d_identity_meta
13. d_mer_structure
14. d_sales_channel
15. d_store_master
16. d_tender
17. d_token
18. d_wallet
19. d_wallet_meta
20. d_wallet_transaction_meta

Fact tables
21. f_account_create_campaign
22. f_account_transaction_detail
23. f_ref_voucher
24. f_staff_point
25. f_wallet_account_transaction
26. f_wallet_transaction
27. f_wallet_transaction_account
28. f_wallet_transaction_adj_result

Sales tables
29. sales_receipt_all
30. sales_sku_all
31. sales_tender_all

## Schema: report

1. d_customer_info
---

### 1. refined.d_account

Loyalty accounts (POINTS or ECOUPON type) linked to a wallet. One wallet can have multiple accounts.

Indexes:
- 📌 wallet_id — btree
- 📌 account_id — btree
- 📌 campaign_id — btree
- 📌 type — btree
- 📌 state — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 2. refined.d_account_meta

Key-value metadata attached to loyalty accounts.

Indexes:
- 📌 account_id — btree
- 📌 key — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 3. refined.d_campaign

Campaign master data including rules, rewards, and distribution configuration.

Indexes:
- 📌 campaign_id — btree
- 📌 ees_master_event_id — btree

### 4. refined.d_consumer

Consumer (member) profile data.

Indexes:
- 🔑 consumer_id — btree unique
- 🔑 consumer_id — btree unique
- 📌 wallet_id — btree
- 📌 state — btree
- 📌 type — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 5. refined.d_consumer_address

Consumer address records.

Indexes: No dedicated index beyond partition.
Indexes:
- - — — —

### 6. refined.d_consumer_contact

Consumer contact information (phone, email) stored as key-value pairs.

Indexes:
- 📌 consumer_id — btree
- 📌 type — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 7. refined.d_consumer_dimension

Consumer dimension / segment label-value pairs.

Indexes:
- 📌 consumer_id — btree
- 📌 label — btree
- 🔗 label, value — btree

### 8. refined.d_consumer_meta

Key-value metadata for consumers.

Indexes:
- 📌 consumer_id — btree
- 📌 key — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 9. refined.d_consumer_segmentation

Consumer segmentation assignments.

Indexes:
- - — — —

### 10. refined.d_identity

Consumer identity records (phone, card, etc.) linked to wallets.

Indexes:
- 🔑 identity_id — btree unique
- 📌 identity_id — btree
- 📌 wallet_id — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 11. refined.d_identity_deleted

Deleted identity audit records.

Indexes: No dedicated index beyond partition fields.

### 12. refined.d_identity_meta

Key-value metadata for identities.

Indexes:
- 📌 identity_id — btree
- 📌 key — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 13. refined.d_mer_structure

Merchandise category hierarchy (up to 8 levels). JOIN required: cate_id = id AND partner_code = branch_code.

Indexes:
- 🔑 id, branch_code — btree unique (composite PK)
- 📌 id — btree

### 14. refined.d_sales_channel

Sales channel dimension per banner.

Indexes:
- 🔑 id, banner_code — btree unique (composite PK)
- 📌 id — btree
- 📌 banner_code — btree

### 15. refined.d_store_master

Store master dimension — store details per banner.

Indexes:
- 🔑 store_code — btree unique
- 📌 store_code — btree
- 📌 banner_code — btree
- 📌 bu — btree

### 16. refined.d_tender

Tender type dimension per banner.

Indexes:
- 🔑 id, banner_code — btree unique (composite PK)
- 📌 id — btree
- 📌 banner_code — btree

### 17. refined.d_token

Voucher/coupon token records associated with ECOUPON accounts.

Indexes:
- 🔑 token_id — btree unique
- 📌 token_id — btree
- 📌 account_id — btree
- 📌 status — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 18. refined.d_wallet

Wallet master records. One wallet per THE1 member.

Indexes:
- 🔑 wallet_id — btree unique
- 📌 wallet_id — btree
- 📌 status — btree
- 📌 type — btree
- 📌 date_created — btree
- 🔗 par_month, par_day — btree

### 19. refined.d_wallet_meta

Key-value metadata for wallets.

Indexes:
- 📌 wallet_id — btree
- 📌 key — btree
- 📌 date_created — btree
- 🔗 wallet_id, key — btree
- 🔗 par_month, par_day — btree

### 20. refined.d_wallet_transaction_meta

Key-value metadata for wallet transactions.

Indexes:
- 📌 wallet_transaction_id — btree
- 📌 key — btree
- 📌 event_stored_at — btree
- 🔗 par_month, par_day — btree

## Fact Tables

### 21. refined.f_account_create_campaign

Records of ECOUPON accounts created from campaign exchange (point debit → voucher issuance).

Indexes:
- - — — —

### 22. refined.f_account_transaction_detail

Supplementary detail for account transactions (bill receipt, amount, store).

Indexes:
- 📌 account_transaction_id — btree
- 📌 event_id — btree
- 📌 event_stored_at — btree

### 23. refined.f_ref_voucher

Voucher reference records used in transactions.

Indexes:
- 📌 account_transaction_id — btree
- 📌 event_id — btree
- 📌 event_stored_at — btree
- 📌 voucher_no — btree
- 🔗 par_month, par_day — btree

