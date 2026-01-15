# 💰 Incentive Compensation Tool

A self-service dashboard for CSMs to track their book of business, quarterly revenue performance, and YoY pacing — powered by **live data** from BigQuery.

## 🚀 Quick Start

**Live Dashboard:** [incentive-compensation-tool.quick.shopify.io](https://incentive-compensation-tool.quick.shopify.io)

1. Open the dashboard
2. Grant BigQuery access when prompted (one-time)
3. Enter your name (auto-populated from your Shopify identity)
4. Click **⚡ Load Live Data**
5. View your real-time compensation data!

---

## 📡 How It Works

This Quick site uses the **Quick.js BigQuery API** (`quick.dw`) to query Shopify's data warehouse directly in your browser:

```javascript
// Example: Query accounts for a CSM
const results = await quick.dw.querySync(`
    SELECT account_name, gmv_usd, revenue_usd
    FROM \`shopify-dw.sales.sales_accounts\`
    WHERE merchant_success_manager LIKE '%Maya Marks%'
`);
```

### Data Sources

| Metric | BigQuery Table |
|--------|----------------|
| Account List | `shopify-dw.sales.sales_accounts` |
| Shop Mapping | `shopify-dw.sales.sales_unified_entity_mapping` |
| Revenue (Quarterly) | `shopify-dw.finance.shop_netsuite_account_daily_profit_summary` |
| GMV | `shopify-dw.finance.gmv` |

---

## 🔐 Authentication

The dashboard uses Google OAuth to access BigQuery:

1. First visit prompts for BigQuery permission
2. Your Shopify Google account grants access
3. Queries run with your personal permissions
4. No data is stored — everything is queried live

---

## 📊 Dashboard Features

### Summary Metrics
- **Total Accounts** — Number of accounts in your book
- **2025 Total Revenue** — Sum of Q1-Q4 2025 revenue
- **L12M GMV** — Last 12 months Gross Merchandise Volume
- **Average Take Rate** — (L12M Revenue / L12M GMV) × 100

### YoY Pacing
- **Q1 2025 Revenue** — Baseline quarter
- **Q1 2026 YTD** — Current quarter progress
- **Pacing Rate** — How Q1 2026 is tracking vs Q1 2025

### Account Details Table
- Account name with Salesforce link
- L12M GMV and Revenue
- Take Rate
- Quarterly revenue breakdown (Q1-Q4 2025, Q1 2026)
- YoY comparison

---

## 🔄 Refreshing Data

Since data is pulled live from BigQuery:
- Click **⚡ Load Live Data** anytime to refresh
- Data reflects latest available in the data warehouse
- No caching — always up-to-date

---

## 📁 Repository Structure

```
IncentiveCompensationTool/
├── index.html              # Main dashboard (deployed to Quick)
├── data/                   # Legacy static data (optional)
└── README.md
```

---

## 🛠 Technical Details

### Quick.js APIs Used

```javascript
// BigQuery data warehouse
quick.dw.querySync(sql)

// Authentication
quick.auth.requestScopes([...])

// User identity
quick.id.fullName
quick.id.email
```

### Key Queries

**1. Get CSM's Accounts:**
```sql
SELECT account_id, name, account_url, country_code, shop_id
FROM `shopify-dw.sales.sales_accounts` sa
JOIN `shopify-dw.sales.sales_unified_entity_mapping` m
  ON sa.account_id = m.salesforce_account_id
WHERE LOWER(sa.merchant_success_manager) LIKE '%name%'
```

**2. Get Quarterly Revenue:**
```sql
SELECT shop_id,
  SUM(CASE WHEN date BETWEEN '2025-01-01' AND '2025-03-31' 
      THEN estimated_profit_usd ELSE 0 END) AS q1_2025
FROM `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`
WHERE shop_id IN (...)
GROUP BY shop_id
```

**3. Get GMV:**
```sql
SELECT shop_id, SUM(gmv_usd) AS l12m_gmv
FROM `shopify-dw.finance.gmv`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
GROUP BY shop_id
```

---

## 👩‍💻 Maintainer

**Maya Marks** - [@MareikeMarks](https://github.com/MareikeMarks)

Questions? Slack: #csm-tools
