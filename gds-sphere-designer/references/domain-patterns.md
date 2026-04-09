# Common patterns by domain

### Financial / Banking
- **Entities:** accounts, customers, counterparties
- **Events:** transactions (deposits, withdrawals, transfers)
- **Key dimensions:** tx_count, sum_amount, n_distinct_counterparties, burst_daily
- **Composite:** account × counterparty_bank pairs
- **Temporal:** 90d rolling windows

### Supply chain / ERP
- **Entities:** customers, suppliers, products
- **Events:** orders, line items, shipments
- **Key dimensions:** order_count, total_spend, n_suppliers, price_std, late_days
- **Composite:** supplier × product pairs
- **Temporal:** 90d or quarterly windows

### SaaS / Product analytics
- **Entities:** users, organizations, features
- **Events:** usage events, API calls, sessions
- **Key dimensions:** event_count, session_duration, n_features_used, burst_daily
- **Composite:** user × feature pairs
- **Temporal:** 30d rolling windows

### Accounting / GL
- **Entities:** GL accounts, cost centers, company codes, profit centers
- **Events:** GL entries (journal lines)
- **Key dimensions:** entry_count, sum_amount, n_cost_centers, amount_std
- **Composite:** account × cost_center pairs
- **Temporal:** monthly windows (30d)
