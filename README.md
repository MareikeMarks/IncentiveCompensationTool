# 💰 Incentive Compensation Tool

A self-service dashboard for CSMs to track their book of business, quarterly revenue performance, and YoY pacing.

## 🚀 Quick Start

**Live Dashboard:** [incentive-compensation-tool.quick.shopify.io](https://incentive-compensation-tool.quick.shopify.io)

1. Enter your name in the dashboard
2. If your data exists in `/data/`, it loads automatically
3. If not, follow the instructions below to add your data

---

## 📁 Repository Structure

```
IncentiveCompensationTool/
├── index.html              # Main dashboard (deployed to Quick)
├── data/
│   ├── maya-marks.json     # Example CSM data file
│   └── [your-name].json    # Add your data here!
└── README.md
```

---

## 📊 Adding Your Data

### Step 1: Generate Your Data

Open **Cursor** and run this prompt:

```
Generate my book of business JSON file for [Your Name] with:
- All my accounts with Salesforce links
- L12M GMV and Revenue
- Take Rate (L12M Revenue / L12M GMV × 100)
- Quarterly revenue for 2025 (Q1-Q4)
- Q1 2026 YTD revenue
- Format as JSON matching data/maya-marks.json schema
```

### Step 2: Create Your Data File

Save your JSON file as `data/[your-name].json`

Example: `data/john-smith.json`

### Step 3: Commit to This Repo

```bash
git add data/[your-name].json
git commit -m "Add data for [Your Name]"
git push
```

### Step 4: Load Your Dashboard

Go to the dashboard and enter your name - your data will load automatically!

---

## 📋 Data Schema

```json
{
  "csm_name": "Your Name",
  "generated_at": "2026-01-15T20:30:00Z",
  "totals": {
    "account_count": 37,
    "total_l12m_gmv": 3284567890,
    "total_l12m_revenue": 45678901,
    "total_2025_revenue": 42156789,
    "avg_take_rate": 1.39
  },
  "q1_comparison": {
    "q1_2025": 9876543,
    "q1_2026_ytd": 2567890,
    "pacing_rate": 26.0
  },
  "accounts": [
    {
      "name": "Account Name",
      "sfdc_url": "https://shopify.lightning.force.com/...",
      "l12m_gmv": 456789012,
      "l12m_revenue": 6789012,
      "take_rate": 1.49,
      "q1_2025": 1567890,
      "q2_2025": 1678901,
      "q3_2025": 1789012,
      "q4_2025": 1890123,
      "q1_2026": 412345,
      "country": "DE"
    }
  ]
}
```

---

## 🔄 Keeping Data Fresh

Run the Cursor prompt monthly (or whenever you need updated data) and push a new version of your JSON file.

---

## 👩‍💻 Maintainer

**Maya Marks** - [@MareikeMarks](https://github.com/MareikeMarks)

Questions? Slack: #csm-tools

