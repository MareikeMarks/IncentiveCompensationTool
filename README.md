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

### BigQuery Queries

The dashboard runs **10 queries** in parallel to build your complete book of business view.

---

#### Query 1: Account Information
Get all accounts assigned to the MSM with their basic info and shop mappings.

**Table:** `shopify-dw.sales.sales_accounts`

```sql
SELECT DISTINCT
    sa.account_id,
    sa.name AS account_name,
    sa.account_url,
    sa.merchant_success_manager,
    sa.account_country_code,
    sa.account_type,
    COALESCE(sa.service_model, 'SMB') AS service_model,
    m.shop_id
FROM `shopify-dw.sales.sales_accounts` sa
LEFT JOIN `shopify-dw.sales.sales_unified_entity_mapping` m
    ON sa.account_id = m.salesforce_account_id
WHERE LOWER(sa.merchant_success_manager) LIKE '%maya%marks%'
    AND sa.account_type = 'Customer'
    AND sa.service_model IN ('Mid-Market', 'Large', 'Enterprise')
```

**Returns:** Account ID, name, SFDC URL, country, service model (segment), shop IDs

---

#### Query 2: Revenue by Quarter
Get historical and current revenue by quarter for NRR calculations.

**Table:** `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN estimated_profit_usd ELSE 0 END) AS l12m_revenue,
    SUM(CASE WHEN date BETWEEN '2025-01-01' AND '2025-03-31' THEN estimated_profit_usd ELSE 0 END) AS q1_2025,
    SUM(CASE WHEN date BETWEEN '2025-04-01' AND '2025-06-30' THEN estimated_profit_usd ELSE 0 END) AS q2_2025,
    SUM(CASE WHEN date BETWEEN '2025-07-01' AND '2025-09-30' THEN estimated_profit_usd ELSE 0 END) AS q3_2025,
    SUM(CASE WHEN date BETWEEN '2025-10-01' AND '2025-12-31' THEN estimated_profit_usd ELSE 0 END) AS q4_2025,
    SUM(CASE WHEN date BETWEEN '2026-01-01' AND '2026-03-31' THEN estimated_profit_usd ELSE 0 END) AS q1_2026
FROM `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`
WHERE shop_id IN (${shopIds})
    AND account_type = 'Income'
GROUP BY shop_id
```

**Returns:** L12M revenue + revenue by quarter (Q1-Q4 for 2025 and 2026)

**Note:** `estimated_profit_usd` includes ALL revenue sources (SP fees, subscriptions, transaction fees, apps, shipping, tax, capital, POS, etc.)

---

#### Query 3: GMV (Gross Merchandise Volume)
Get GMV for take rate calculations.

**Table:** `shopify-dw.finance.shop_gmv_daily_summary_v1`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN gmv_usd ELSE 0 END) AS l12m_gmv,
    SUM(CASE WHEN date BETWEEN '2025-01-01' AND '2025-03-31' THEN gmv_usd ELSE 0 END) AS q1_gmv_2025,
    SUM(CASE WHEN date BETWEEN '2026-01-01' AND '2026-03-31' THEN gmv_usd ELSE 0 END) AS q1_gmv_2026
FROM `shopify-dw.finance.shop_gmv_daily_summary_v1`
WHERE shop_id IN (${shopIds})
GROUP BY shop_id
```

**Returns:** L12M GMV, Q1 2025 GMV (for YoY comparison), Q1 2026 GMV

**Note:** GMV = Total order value across ALL payment methods (SP, Adyen, PayPal, Klarna, etc.)

---

#### Query 4: Shopify Payments Adoption
Check if SP is enabled on each shop.

**Table:** `shopify-dw.money_products.shopify_payments_adoption_current`

```sql
SELECT shop_id, is_enabled AS sp_enabled
FROM `shopify-dw.money_products.shopify_payments_adoption_current`
WHERE shop_id IN (${shopIds})
```

---

#### Query 5: GPV (Gross Payments Volume)
Get volume processed ONLY through Shopify Payments.

**Table:** `shopify-dw.finance.shop_gpv_daily_summary_v1`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN gpv_usd ELSE 0 END) AS l12m_gpv
FROM `shopify-dw.finance.shop_gpv_daily_summary_v1`
WHERE shop_id IN (${shopIds})
GROUP BY shop_id
```

**Returns:** L12M GPV (Shopify Payments volume only)

**Note:** GPV ≠ GMV. GPV only includes orders processed through Shopify Payments, not third-party gateways.

---

#### Query 6: SP Revenue (Shopify Payments Specific)
Get revenue ONLY from Shopify Payments processing fees.

**Table:** `shopify-dw.finance.payments_revenue_and_costs_v1`

```sql
SELECT 
    shop_id,
    SUM(processing_revenue_usd) AS l12m_sp_revenue,
    SUM(fx_fee_revenue_usd) AS l12m_fx_revenue,
    SUM(chargeback_revenue_usd) AS l12m_chargeback_revenue
FROM `shopify-dw.finance.payments_revenue_and_costs_v1`
WHERE shop_id IN (${shopIds})
    AND DATE(date) >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
GROUP BY shop_id
```

**Returns:** SP processing revenue, FX fees, chargeback fees

**Note:** This is ONLY Shopify Payments revenue (not subscriptions, transaction fees, or other revenue)

---

#### Query 7: Daily Revenue (for Run Rate Chart)
Get daily revenue for Q1 2026 to track pacing against target.

**Table:** `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`

```sql
SELECT 
    shop_id,
    date,
    SUM(estimated_profit_usd) AS daily_revenue
FROM `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`
WHERE shop_id IN (${shopIds})
    AND account_type = 'Income'
    AND date BETWEEN '2026-01-01' AND CURRENT_DATE()
GROUP BY shop_id, date
```

**Returns:** Daily revenue by shop for current quarter (used for 7-day moving average and projections)

---

#### Query 8: GMV Growth Rate (QoQ Comparison)
Track quarter-over-quarter GMV growth to identify scaling merchants.

**Table:** `shopify-dw.finance.shop_gmv_daily_summary_v1`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date BETWEEN '2025-10-01' AND '2025-12-31' THEN gmv_usd ELSE 0 END) AS q4_2025_gmv,
    SUM(CASE WHEN date BETWEEN '2026-01-01' AND CURRENT_DATE() THEN gmv_usd ELSE 0 END) AS q1_2026_gmv_ytd,
    CASE 
        WHEN DATE_DIFF(CURRENT_DATE(), DATE('2026-01-01'), DAY) > 0 THEN
            SUM(CASE WHEN date BETWEEN '2026-01-01' AND CURRENT_DATE() THEN gmv_usd ELSE 0 END) 
            / DATE_DIFF(CURRENT_DATE(), DATE('2026-01-01'), DAY) * 90
        ELSE 0
    END AS q1_2026_gmv_projected
FROM `shopify-dw.finance.shop_gmv_daily_summary_v1`
WHERE shop_id IN (${shopIds})
GROUP BY shop_id
```

**Returns:** Q4 2025 GMV, Q1 2026 GMV YTD, Q1 2026 GMV projected

**Use Case:** GMV growth = leading indicator for future revenue opportunity. High GMV growth + missing products = upsell target.

---

#### Query 9: SP Penetration Trends
Track if merchants are routing MORE or LESS volume through Shopify Payments over time. Compares last 3 months vs previous 3 months (6 months total lookback).

**Tables:** 
- `shopify-dw.finance.shop_gpv_daily_summary_v1`
- `shopify-dw.finance.shop_gmv_daily_summary_v1`

```sql
WITH gpv_trends AS (
    SELECT 
        shop_id,
        SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) THEN gpv_usd ELSE 0 END) AS gpv_l3m,
        SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY) 
                 AND date < DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) THEN gpv_usd ELSE 0 END) AS gpv_prev3m
    FROM `shopify-dw.finance.shop_gpv_daily_summary_v1`
    WHERE shop_id IN (${shopIds})
    AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    GROUP BY shop_id
),
gmv_trends AS (
    SELECT 
        shop_id,
        SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) THEN gmv_usd ELSE 0 END) AS gmv_l3m,
        SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY) 
                 AND date < DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) THEN gmv_usd ELSE 0 END) AS gmv_prev3m
    FROM `shopify-dw.finance.shop_gmv_daily_summary_v1`
    WHERE shop_id IN (${shopIds})
    AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    GROUP BY shop_id
)
SELECT 
    gpv.shop_id,
    gpv.gpv_l3m,
    gpv.gpv_prev3m,
    gmv.gmv_l3m,
    gmv.gmv_prev3m,
    CASE WHEN gmv.gmv_l3m > 0 THEN (gpv.gpv_l3m / gmv.gmv_l3m) * 100 ELSE 0 END AS sp_pen_l3m,
    CASE WHEN gmv.gmv_prev3m > 0 THEN (gpv.gpv_prev3m / gmv.gmv_prev3m) * 100 ELSE 0 END AS sp_pen_prev3m
FROM gpv_trends gpv
JOIN gmv_trends gmv ON gpv.shop_id = gmv.shop_id
```

**Returns:** Current SP Penetration (Last 3M), Previous SP Penetration (Prev 3M), Change

**Use Case:** 
- **Increasing Penetration (+):** Winning back volume from other PSPs → Revenue opportunity realized
- **Decreasing Penetration (-):** Losing volume to other PSPs → Risk/churn indicator, needs attention
- **Stable Penetration (~0%):** Consistent merchant behavior

---

#### Query 10: Product Adoption Status
Identify which revenue-driving products each merchant has enabled.

**Tables:** Multiple adoption tables

```sql
WITH adoption_data AS (
    SELECT 
        s.shop_id,
        sp.is_enabled AS has_shopify_payments,
        CASE WHEN pos.shop_id IS NOT NULL THEN TRUE ELSE FALSE END AS has_pos,
        CASE WHEN markets.shop_id IS NOT NULL THEN TRUE ELSE FALSE END AS has_markets,
        CASE WHEN b2b.shop_id IS NOT NULL THEN TRUE ELSE FALSE END AS has_b2b,
        capital.has_active_loan AS has_capital
    FROM (SELECT DISTINCT shop_id FROM UNNEST([${shopIds}]) AS shop_id) s
    LEFT JOIN `shopify-dw.money_products.shopify_payments_adoption_current` sp ON s.shop_id = sp.shop_id
    LEFT JOIN (
        SELECT DISTINCT shop_id FROM `shopify-dw.pos.pos_locations_and_devices_snapshot` 
        WHERE snapshot_date = (SELECT MAX(snapshot_date) FROM `shopify-dw.pos.pos_locations_and_devices_snapshot`)
    ) pos ON s.shop_id = pos.shop_id
    LEFT JOIN (
        SELECT DISTINCT shop_id FROM `shopify-dw.international.markets_shops` WHERE is_active = TRUE
    ) markets ON s.shop_id = markets.shop_id
    LEFT JOIN (
        SELECT DISTINCT shop_id FROM `shopify-dw.b2b.b2b_company_locations`
    ) b2b ON s.shop_id = b2b.shop_id
    LEFT JOIN (
        SELECT shop_id, TRUE AS has_active_loan FROM `shopify-dw.capital.capital_advances`
        WHERE status IN ('Active', 'Partially_Repaid') GROUP BY shop_id
    ) capital ON s.shop_id = capital.shop_id
)
SELECT * FROM adoption_data
```

**Returns:** Boolean flags for: SP, POS, Markets, B2B, Capital

**Use Case:** Missing products = upsell opportunities. Prioritize high-GMV accounts with multiple gaps.

---

#### Query 10: Recent Product Enablements
Track recent product launches to measure impact and share success stories.

**Tables:** Multiple adoption history tables

```sql
WITH sp_enablements AS (
    SELECT shop_id, 'Shopify Payments' AS product_name, first_enabled_at AS enabled_date
    FROM `shopify-dw.money_products.shopify_payments_adoption_history`
    WHERE shop_id IN (${shopIds}) AND first_enabled_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
),
pos_enablements AS (
    SELECT shop_id, 'POS' AS product_name, MIN(created_at) AS enabled_date
    FROM `shopify-dw.pos.pos_locations_and_devices_snapshot`
    WHERE shop_id IN (${shopIds}) AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
    GROUP BY shop_id
),
markets_enablements AS (
    SELECT shop_id, 'Markets' AS product_name, created_at AS enabled_date
    FROM `shopify-dw.international.markets_shops`
    WHERE shop_id IN (${shopIds}) AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
)
SELECT * FROM sp_enablements
UNION ALL SELECT * FROM pos_enablements
UNION ALL SELECT * FROM markets_enablements
ORDER BY enabled_date DESC
```

**Returns:** Shop ID, product name, enablement date for recent launches (last 6 months)

**Use Case:** Timeline of your recent wins, measure pre/post revenue impact

---

### Key Metrics Calculated

#### GMV Take Rate
```
GMV Take Rate = Total Revenue / GMV * 100
```
All Shopify revenue as % of total merchant sales. Lower if merchant uses third-party gateways.

#### Pure SP Take Rate
```
Pure SP Take Rate = SP Revenue / GPV * 100
```
SP fees as % of SP volume only. Shows true SP efficiency. Typically 1.5-3% depending on merchant's SP rate.

#### SP Penetration
```
SP Penetration = GPV / GMV * 100
```
What % of orders go through Shopify Payments vs third-party gateways. Higher penetration = more revenue from processing fees.

#### NRR (Net Revenue Retention)
```
NRR = Q1 2026 Revenue / Q1 2025 Revenue * 100
```
Year-over-year growth. Target: Match or exceed prior year's quarter revenue.

#### GMV Growth Rate
```
GMV Growth Rate = (Q1 Projected - Q4 Actual) / Q4 Actual * 100
```
Quarter-over-quarter merchant growth. Leading indicator for future revenue opportunity.

---

## 👩‍💻 Maintainer

**Maya Marks** - [@MareikeMarks](https://github.com/MareikeMarks)

Questions? Slack: #csm-tools
