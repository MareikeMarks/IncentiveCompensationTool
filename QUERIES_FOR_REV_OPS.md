# BigQuery Queries Used in Incentive Compensation Dashboard

## Overview
This dashboard pulls data from Shopify's data warehouse to calculate NRR targets, take rates, and revenue metrics for assigned accounts.

---

## Query 1: Account Information
**Purpose:** Get all accounts assigned to the MSM with their basic info and shop mappings

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

**Returns:**
- Account ID (Salesforce)
- Account name
- Account URL
- MSM name
- Country code
- Service model (segment)
- Shop IDs (multiple shops per account possible)

---

## Query 2: Revenue by Quarter
**Purpose:** Get historical and current revenue by quarter for NRR calculations

**Table:** `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN estimated_profit_usd ELSE 0 END) AS l12m_revenue,
    SUM(CASE WHEN date BETWEEN '2025-01-01' AND '2025-03-31' THEN estimated_profit_usd ELSE 0 END) AS q1_2025,
    SUM(CASE WHEN date BETWEEN '2025-04-01' AND '2025-06-30' THEN estimated_profit_usd ELSE 0 END) AS q2_2025,
    SUM(CASE WHEN date BETWEEN '2025-07-01' AND '2025-09-30' THEN estimated_profit_usd ELSE 0 END) AS q3_2025,
    SUM(CASE WHEN date BETWEEN '2025-10-01' AND '2025-12-31' THEN estimated_profit_usd ELSE 0 END) AS q4_2025,
    SUM(CASE WHEN date BETWEEN '2026-01-01' AND '2026-03-31' THEN estimated_profit_usd ELSE 0 END) AS q1_2026,
    SUM(CASE WHEN date BETWEEN '2026-04-01' AND '2026-06-30' THEN estimated_profit_usd ELSE 0 END) AS q2_2026,
    SUM(CASE WHEN date BETWEEN '2026-07-01' AND '2026-09-30' THEN estimated_profit_usd ELSE 0 END) AS q3_2026,
    SUM(CASE WHEN date BETWEEN '2026-10-01' AND '2026-12-31' THEN estimated_profit_usd ELSE 0 END) AS q4_2026
FROM `shopify-dw.finance.shop_netsuite_account_daily_profit_summary`
WHERE shop_id IN (${shopIds})
    AND account_type = 'Income'
    AND date >= '2024-01-15'
GROUP BY shop_id
```

**Returns:**
- Last 12 months revenue
- Revenue by quarter (Q1-Q4 for 2025 and 2026)

**Notes:**
- Uses `estimated_profit_usd` which includes ALL revenue sources:
  - Shopify Payments processing fees
  - Subscription fees
  - Transaction fees (for non-SP payments)
  - App/theme revenue
  - Shopify Shipping
  - Shopify Tax
  - Shop Pay revenue share
  - Capital interest
  - Markets fees
  - POS Pro subscriptions
  - All other Shopify revenue

---

## Query 3: GMV (Gross Merchandise Volume)
**Purpose:** Get GMV for take rate calculations

**Table:** `shopify-dw.finance.shop_gmv_daily_summary_v1`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN gmv_usd ELSE 0 END) AS l12m_gmv,
    SUM(CASE WHEN date BETWEEN '2025-01-01' AND '2025-03-31' THEN gmv_usd ELSE 0 END) AS q1_gmv_2025,
    SUM(CASE WHEN date BETWEEN '2026-01-01' AND '2026-03-31' THEN gmv_usd ELSE 0 END) AS q1_gmv_2026
FROM `shopify-dw.finance.shop_gmv_daily_summary_v1`
WHERE shop_id IN (${shopIds})
    AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
GROUP BY shop_id
```

**Returns:**
- Last 12 months GMV
- Q1 2025 GMV (for YoY comparison)
- Q1 2026 GMV (current quarter)

**Notes:**
- GMV = Total order value across ALL payment methods (SP, Adyen, PayPal, Klarna, etc.)
- Used to calculate **GMV Take Rate** = Total Revenue / GMV

---

## Query 4: Shopify Payments Adoption
**Purpose:** Check if SP is enabled on each shop

**Table:** `shopify-dw.money_products.shopify_payments_adoption_current`

```sql
SELECT shop_id, is_enabled AS sp_enabled
FROM `shopify-dw.money_products.shopify_payments_adoption_current`
WHERE shop_id IN (${shopIds})
```

**Returns:**
- Whether Shopify Payments is enabled (boolean)

---

## Query 5: GPV (Gross Payments Volume)
**Purpose:** Get volume processed ONLY through Shopify Payments

**Table:** `shopify-dw.finance.shop_gpv_daily_summary_v1`

```sql
SELECT 
    shop_id,
    SUM(CASE WHEN date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY) THEN gpv_usd ELSE 0 END) AS l12m_gpv
FROM `shopify-dw.finance.shop_gpv_daily_summary_v1`
WHERE shop_id IN (${shopIds})
    AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
GROUP BY shop_id
```

**Returns:**
- Last 12 months GPV (Shopify Payments volume only)

**Notes:**
- GPV = Order value processed through Shopify Payments ONLY
- Does NOT include third-party gateways (Adyen, PayPal, etc.)
- **SP Penetration** = GPV / GMV (what % of orders go through SP)

---

## Query 6: SP Revenue (Shopify Payments Specific)
**Purpose:** Get revenue ONLY from Shopify Payments processing fees

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

**Returns:**
- SP processing revenue (card processing fees)
- FX fee revenue (currency conversion fees)
- Chargeback revenue (chargeback fees)

**Notes:**
- This is ONLY Shopify Payments revenue
- Does NOT include subscriptions, transaction fees, or other revenue
- **Pure SP Take Rate** = SP Revenue / GPV (SP processing fees as % of SP volume)

---

## Query 7: Daily Revenue (for Run Rate Chart)
**Purpose:** Get daily revenue for Q1 2026 to track pacing against target

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
ORDER BY date ASC
```

**Returns:**
- Daily revenue by shop for current quarter
- Used to calculate 7-day moving average and project quarter-end total

---

## Key Metrics Calculated

### 1. GMV Take Rate
```
GMV Take Rate = Total Revenue / GMV * 100
```
- Measures total Shopify revenue as % of total merchant GMV
- Includes ALL revenue sources (SP, subscriptions, transaction fees, etc.)
- Lower if merchant uses third-party gateways (Adyen, PayPal, Klarna)

### 2. Pure SP Take Rate
```
Pure SP Take Rate = SP Revenue / GPV * 100
```
- Measures ONLY Shopify Payments revenue as % of SP volume
- Isolates SP processing fees from other revenue
- Not affected by third-party gateways
- Typically 1.5-3% depending on merchant's SP rate

### 3. SP Penetration
```
SP Penetration = GPV / GMV * 100
```
- What % of orders go through Shopify Payments vs third-party gateways
- Higher penetration = more revenue from processing fees

### 4. NRR (Net Revenue Retention)
```
NRR = Q1 2026 Revenue / Q1 2025 Revenue * 100
```
- Measures year-over-year growth
- Target: Match or exceed prior year's quarter revenue

---

## Data Freshness
- **Daily tables** (`*_daily_summary*`): Updated daily, typically D-1 (yesterday's data)
- **Adoption tables** (`*_adoption_current`): Near real-time
- `CURRENT_DATE()` returns today's date but data typically reflects through yesterday

---

## Questions?
Contact: Maya Marks (creator of this dashboard)

