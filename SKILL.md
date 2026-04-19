# Stripe Finance Automation

**Fully automated finance tracking system.** Stripe payments and cancellations are logged to Google Sheets in real-time via n8n, with Slack notifications and a monthly P&L report.

Built by [300Growth](https://300growth.com). Free to use.

---

## What It Does

1. **Real-time payment logging** — When a Stripe charge succeeds, a new row is added to the Revenue tab in Google Sheets with client name, amount, date, type (retainer vs one-time), and Stripe ID.
2. **Churn detection** — When a customer cancels their subscription, a churn row is logged automatically.
3. **Slack alerts** — Instant notification in your Slack channel for every payment and every cancellation.
4. **Monthly finance report** — On the 1st of every month, a P&L summary is posted to Slack with revenue, expenses, profit, margins, and YTD totals.
5. **Auto-calculated P&L** — The Google Sheet has formulas that calculate monthly revenue, expenses, net profit, and margins across 12 months.
6. **Executive dashboard** — All-time totals, this-month breakdown, MRR, active client count, and profit margins update automatically.

---

## Architecture

```
Stripe (charge.succeeded / subscription.deleted)
    │
    ▼
n8n Webhook (receives POST)
    │
    ▼
Switch Node (routes by event type)
    │
    ├── Payment branch:
    │   Fetch Customer from Stripe API → Format Data → Append to Google Sheets → Slack Alert
    │
    └── Churn branch:
        Fetch Customer from Stripe API → Format Data → Append to Google Sheets → Slack Alert

Separate workflow:
Schedule Trigger (1st of month, 9 AM UTC)
    → Read P&L from Google Sheets
    → Build report message
    → Post to Slack
```

---

## Prerequisites

You need accounts on these platforms:

| Service | What For | Where to Sign Up |
|---------|----------|------------------|
| **Stripe** | Payment processing | [stripe.com](https://stripe.com) |
| **n8n** | Workflow automation | [n8n.io](https://n8n.io) (cloud or self-hosted) |
| **Google Sheets** | Finance spreadsheet | [sheets.google.com](https://sheets.google.com) |
| **Slack** | Notifications | [slack.com](https://slack.com) |

---

## Step 1: Get Your API Keys

### Stripe API Key (Restricted Key)

1. Go to [Stripe Dashboard](https://dashboard.stripe.com) → **Developers** → **API Keys**
2. Click **Create restricted key**
3. Give it a name (e.g., `n8n-finance`)
4. Set these permissions:
   - **Charges**: Read
   - **Customers**: Read
   - **Subscriptions**: Read
   - **Webhook Endpoints**: Write
5. Copy the key (starts with `rk_live_` or `rk_test_`)

### n8n API Key

1. Go to your n8n instance → click your **profile icon** (bottom-left) → **Settings**
2. Go to **API** section
3. Click **Create API Key**
4. Copy the key

Your n8n base URL is:
- **Cloud**: `https://your-workspace.app.n8n.cloud`
- **Self-hosted**: whatever your instance URL is (e.g., `https://n8n.yourdomain.com`)

### Slack Bot Token

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Name it (e.g., `Finance Bot`), select your workspace
3. Go to **OAuth & Permissions** → scroll to **Scopes** → add:
   - `chat:write`
   - `chat:write.public`
4. Click **Install to Workspace** at the top → **Allow**
5. Copy the **Bot User OAuth Token** (starts with `xoxb-`)
6. Get your channel ID:
   - In Slack, right-click your target channel → **View channel details** → scroll to bottom → copy the **Channel ID** (starts with `C`)

### Google Sheets OAuth2 (in n8n)

1. In n8n, go to **Credentials** → **Add Credential** → search **Google Sheets OAuth2 API**
2. Follow n8n's built-in OAuth flow to connect your Google account
3. Note the **Credential ID** shown in the URL when editing it (e.g., `AbCdEfGh12345678`)

### Stripe Credential (in n8n)

1. In n8n, go to **Credentials** → **Add Credential** → search **Stripe API**
2. Paste your Stripe secret key (or the restricted key from above)
3. Note the **Credential ID**

---

## Step 2: Create the Google Sheet

Create a new Google Sheets spreadsheet with these tabs:

### Tab 1: Revenue

| Row | Content |
|-----|---------|
| A1  | `REVENUE` (title) |
| A3:H3 | Headers: `Date` · `Client Name` · `Amount (USD)` · `Revenue Type` · `Payment Status` · `Subscription Status` · `Stripe ID` · `Notes` |
| A4+  | Data rows (populated automatically by n8n) |

**Column details:**

| Column | Description | Example Values |
|--------|-------------|----------------|
| A - Date | Payment/event date | `2026-03-15` |
| B - Client Name | Derived from Stripe customer email | `Acme` |
| C - Amount (USD) | Payment amount in dollars | `500` |
| D - Revenue Type | `Retainer`, `One-Time`, or `Churn` | `Retainer` |
| E - Payment Status | `Paid` or `Cancelled` | `Paid` |
| F - Subscription Status | `Active`, `Cancelled`, or blank | `Active` |
| G - Stripe ID | Stripe charge or subscription ID | `ch_xxx...` |
| H - Notes | Auto-generated context | `Auto-logged via N8N` |

### Tab 2: Expenses

| Row | Content |
|-----|---------|
| A1  | `YOUR EXPENSES` (title) |
| A2  | `Recurring monthly expenses auto-repeat in P&L for every month from start date onward` |
| A3:J3 | Headers: `Date` · `Category` · `Vendor` · `Amount (USD)` · `Billing Cycle` · `Recurring?` · `Client-Related?` · `Payment Method` · `Annual Equivalent` · `Notes` |
| A4+  | Your expense data (enter manually) |

**Expense categories** (use these exact names for P&L formulas to work):
- Software & Tools
- Domains & Email Infra
- AI & API Costs
- Data & Leads
- Contractors/Freelancers
- Advertising & Marketing
- Legal & Compliance
- Office & Equipment
- Education & Training
- Miscellaneous

**Annual Equivalent formula** (column I, for each expense row starting at row 4):
```
=IF(E4="Monthly",D4*12,IF(E4="Yearly",D4,IF(E4="One-Time",D4,0)))
```

### Tab 3: P&L Summary

This is the core financial view with formulas that auto-calculate from Revenue and Expenses tabs.

**Layout:**

| Row | Column A | Columns B-M (Jan-Dec) | Column N (YTD) |
|-----|----------|----------------------|----------------|
| 1 | `PROFIT & LOSS STATEMENT` | | |
| 4 | *(header)* | `Jan` through `Dec` | `YTD` |
| 6 | `REVENUE` | | |
| 7 | `Client Revenue (Recurring)` | *formula* | `=SUM(B7:M7)` |
| 8 | `One-Time Payments` | *formula* | `=SUM(B8:M8)` |
| 9 | `Total Revenue` | `=B7+B8` | `=SUM(B9:M9)` |
| 11 | `EXPENSES` | | |
| 12-21 | *10 expense categories* | *formula* | `=SUM(B12:M12)` |
| 22 | `Total Expenses` | `=SUM(B12:B21)` | `=SUM(B22:M22)` |
| 24 | `NET PROFIT` | `=B9-B22` | `=SUM(B24:M24)` |
| 25 | `Profit Margin %` | `=IFERROR(B24/B9,0)` | `=IFERROR(N24/N9,0)` |
| 27 | `CUMULATIVE` | | |
| 28 | `Cumulative Revenue` | `=B9` (Jan), `=B28+C9` (Feb+) | |
| 29 | `Cumulative Expenses` | `=B22` (Jan), `=B29+C22` (Feb+) | |
| 30 | `Cumulative Profit` | `=B24` (Jan), `=B30+C24` (Feb+) | |

**Revenue formulas** (for each month column B-M, where `{m}` = month number 1-12):

Row 7 — Recurring revenue:
```
=SUMPRODUCT((MONTH(Revenue!A4:A500)={m})*(Revenue!E4:E500="Paid")*(Revenue!D4:D500<>"One-Time")*(Revenue!D4:D500<>"Churn")*Revenue!C4:C500)
```

Row 8 — One-time revenue:
```
=SUMPRODUCT((MONTH(Revenue!A4:A500)={m})*(Revenue!E4:E500="Paid")*(Revenue!D4:D500="One-Time")*Revenue!C4:C500)
```

**Expense formulas** (rows 12-21, for each category, where `{cat}` = category name, `{m}` = month number):

```
=SUMPRODUCT((Expenses!B4:B500="{cat}")*(Expenses!E4:E500="Monthly")*(Expenses!F4:F500="Yes")*({m}>=MONTH(Expenses!A4:A500))*Expenses!D4:D500)+SUMPRODUCT((Expenses!B4:B500="{cat}")*(Expenses!E4:E500<>"Monthly")*(MONTH(Expenses!A4:A500)={m})*Expenses!D4:D500)
```

This formula does two things:
1. For **recurring monthly expenses**: includes the expense in every month from its start date onward
2. For **one-time expenses**: includes only in the month they occurred

### Tab 4: Executive Summary

| Row | Column A | Column B |
|-----|----------|----------|
| 1 | `YOUR COMPANY — FINANCIAL SUMMARY` | |
| 3 | `AS OF` | `=TEXT(TODAY(),"MMMM D, YYYY")` |
| 5 | `ALL TIME` | |
| 7 | `Total Revenue` | `=SUMPRODUCT((Revenue!E4:E500="Paid")*Revenue!C4:C500)` |
| 9 | `Total Expenses` | *(see below)* |
| 11 | `NET PROFIT` | `=B7-B9` |
| 13 | `PROFIT MARGIN` | `=IFERROR(B11/B7,0)` |
| 16 | `THIS MONTH` | `=TEXT(TODAY(),"MMMM")` |
| 18 | `Revenue this month` | `=SUMPRODUCT((YEAR(Revenue!A4:A500)=YEAR(TODAY()))*(MONTH(Revenue!A4:A500)=MONTH(TODAY()))*(Revenue!E4:E500="Paid")*Revenue!C4:C500)` |
| 19 | `Expenses this month` | `=SUMPRODUCT((Expenses!E4:E500="Monthly")*(Expenses!F4:F500="Yes")*Expenses!D4:D500)+SUMPRODUCT((Expenses!E4:E500<>"Monthly")*(YEAR(Expenses!A4:A500)=YEAR(TODAY()))*(MONTH(Expenses!A4:A500)=MONTH(TODAY()))*Expenses!D4:D500)` |
| 20 | `Profit this month` | `=B18-B19` |
| 21 | `Margin this month` | `=IFERROR(B20/B18,0)` |
| 24 | `RECURRING MONTHLY` | |
| 26 | `MRR` | `=SUMPRODUCT((Revenue!D4:D500="Retainer")*(Revenue!F4:F500="Active")*(Revenue!E4:E500="Paid")*(MONTH(Revenue!A4:A500)=MONTH(TODAY()))*(YEAR(Revenue!A4:A500)=YEAR(TODAY()))*Revenue!C4:C500)` |
| 27 | `Monthly tool costs` | `=SUMPRODUCT((Expenses!E4:E500="Monthly")*(Expenses!F4:F500="Yes")*Expenses!D4:D500)` |
| 28 | `Monthly net cash flow` | `=B26-B27` |
| 30 | `Active Clients` | `=IFERROR(COUNTA(UNIQUE(FILTER(Revenue!B4:B500, Revenue!F4:F500="Active", Revenue!E4:E500="Paid", YEAR(Revenue!A4:A500)=YEAR(TODAY()), MONTH(Revenue!A4:A500)=MONTH(TODAY())))),0)` |

**Total Expenses formula (B9)** — handles recurring expenses across all months since they started:
```
=SUMPRODUCT((Expenses!E4:E500="Monthly")*(Expenses!F4:F500="Yes")*Expenses!D4:D500*((YEAR(TODAY())-YEAR(Expenses!A4:A500))*12+MONTH(TODAY())-MONTH(Expenses!A4:A500)+1))+SUMPRODUCT((Expenses!E4:E500<>"Monthly")*Expenses!D4:D500)
```

### Tab 5: Dashboard

This tab has chart data that references the P&L Summary. For each month column (1-12):

| Row | Content |
|-----|---------|
| 4 | Month names (`Jan` through `Dec`) |
| 5 | `='P&L Summary'!B9` (Total Revenue — adjust column per month) |
| 6 | `='P&L Summary'!B22` (Total Expenses — adjust column per month) |
| 7 | `='P&L Summary'!B24` (Net Profit — adjust column per month) |
| 43 | `='P&L Summary'!B30` (Cumulative Profit — adjust column per month) |

Create 3 charts:
1. **Bar chart**: Monthly Revenue vs Expenses (rows 5-6)
2. **Line chart**: Net Profit Trend (row 7)
3. **Line chart**: Cumulative Profit (row 43)

### Tab 6: Stripe Sync Log

| Row | Content |
|-----|---------|
| A1  | `STRIPE SYNC LOG` (title) |
| A3:F3 | Headers: `Sync Date` · `Payments Found` · `New Payments Added` · `Status` · `Synced By` · `Notes` |

Copy the **Sheet ID** from your Google Sheets URL:
```
https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit
```

---

## Step 3: Create the `.env` File

Create a `.env` file in your project root (**never commit this file**):

```env
# Stripe
STRIPE_API_KEY=rk_live_your_stripe_restricted_key_here

# n8n
N8N_API_KEY=your_n8n_api_key_here
N8N_URL=https://your-workspace.app.n8n.cloud

# Google Sheets
SHEET_ID=your_google_sheet_id_here
GOOGLE_SHEETS_CREDENTIAL_ID=your_n8n_google_credential_id

# Stripe credential in n8n
STRIPE_CREDENTIAL_ID=your_n8n_stripe_credential_id

# Slack
SLACK_BOT_TOKEN=xoxb-your-slack-bot-token
SLACK_CHANNEL_ID=C0YOUR_CHANNEL_ID
```

---

## Step 4: Set Up the n8n Workflows

### Workflow 1: Stripe Event Handler

This workflow has 10 nodes:

**Nodes:**

1. **Stripe Webhook** (Webhook node)
   - Type: `n8n-nodes-base.webhook`
   - Path: `stripe-finance-webhook`
   - Method: `POST`
   - Response Mode: `onReceived`

2. **Route by Event** (Switch node)
   - Routes `charge.succeeded` to payment branch (output 0)
   - Routes `customer.subscription.deleted` to churn branch (output 1)
   - Expression: `{{ $json.body.type }}`

   > **Important**: The Webhook node wraps the POST body under `$json.body`, so all references must use `$json.body.type`, `$json.body.data.object.xxx`, etc. This is the #1 gotcha.

3. **Fetch Customer (Payment)** (HTTP Request node)
   - GET `https://api.stripe.com/v1/customers/{customer_id}`
   - Uses Stripe credential for auth
   - Falls back to `/v1/balance` if no customer ID (one-time payments without a customer)
   - Set **On Error** to "Continue Regular Output" so the workflow doesn't break

4. **Format Payment Data** (Set node) — Maps Stripe data to spreadsheet columns:
   - **Date**: `{{ new Date($("Route by Event").first().json.body.data.object.created * 1000).toISOString().split("T")[0] }}`
   - **Client Name**: `{{ $json.email ? ($json.email.split('@')[1]?.split('.')[0]?.replace(/^./, c => c.toUpperCase())) : ($json.name || 'Unknown') }}`
   - **Amount (USD)**: `{{ $('Route by Event').first().json.body.data.object.amount / 100 }}`
   - **Revenue Type**: `{{ $('Route by Event').first().json.body.data.object.invoice ? 'Retainer' : 'One-Time' }}`
   - **Payment Status**: `Paid` (static)
   - **Subscription Status**: `{{ $('Route by Event').first().json.body.data.object.invoice ? 'Active' : '' }}`
   - **Stripe ID**: `{{ $('Route by Event').first().json.body.data.object.id }}`
   - **Notes**: `{{ 'Auto-logged via N8N' + ($('Route by Event').first().json.body.data.object.invoice ? ' | Invoice: ' + $('Route by Event').first().json.body.data.object.invoice : '') }}`

5. **Log to Revenue** (HTTP Request node)
   - POST to `https://sheets.googleapis.com/v4/spreadsheets/{SHEET_ID}/values/Revenue!A4:H4:append`
   - Query params: `valueInputOption=USER_ENTERED`, `insertDataOption=INSERT_ROWS`
   - Uses Google Sheets OAuth2 credential
   - Body:
     ```
     {{ JSON.stringify({ values: [[ $json["Date"], $json["Client Name"], $json["Amount (USD)"], $json["Revenue Type"], $json["Payment Status"], $json["Subscription Status"], $json["Stripe ID"], $json["Notes"] ]] }) }}
     ```

6. **Slack Payment Alert** (HTTP Request node)
   - POST to `https://slack.com/api/chat.postMessage`
   - Header: `Authorization: Bearer {SLACK_BOT_TOKEN}`
   - Message format: `:moneybag: *New Payment Received* — Client, Amount, Date, Type`

7. **Fetch Customer (Churn)** — Same config as #3 but on the churn branch

8. **Format Churn Data** (Set node):
   - **Date**: Today's date
   - **Amount (USD)**: `0`
   - **Revenue Type**: `Churn`
   - **Payment Status**: `Cancelled`
   - **Subscription Status**: `Cancelled`

9. **Log Churn to Revenue** — Same config as #5

10. **Slack Churn Alert** — Posts cancellation notification with client name and date

**Connections:**
```
Stripe Webhook → Route by Event
Route by Event (output 0) → Fetch Customer (Payment) → Format Payment Data → Log to Revenue → Slack Payment Alert
Route by Event (output 1) → Fetch Customer (Churn) → Format Churn Data → Log Churn to Revenue → Slack Churn Alert
```

### Workflow 2: Monthly Finance Report

4 nodes:

1. **Monthly Schedule** (Schedule Trigger)
   - Cron: `0 9 1 * *` (1st of every month at 9:00 AM UTC)

2. **Read P&L Data** (HTTP Request)
   - GET `https://sheets.googleapis.com/v4/spreadsheets/{SHEET_ID}/values/'P%26L%20Summary'!A7:N25`
   - Uses Google Sheets OAuth2 credential

3. **Build Report** (Code node) — JavaScript that reads previous month's column from P&L data and formats a Slack message with recurring revenue, one-time revenue, total, expenses, net profit, margin, and YTD totals.

<details>
<summary>Full Code Node JavaScript</summary>

```javascript
const values = $input.first().json.values || [];
const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
const now = new Date();
const lastMonth = now.getMonth();
const monthIdx = lastMonth === 0 ? 11 : lastMonth - 1;
const monthName = months[monthIdx];
const colIdx = monthIdx + 1;

const recurring = values[0] ? (values[0][colIdx] || '$0') : '$0';
const oneTime = values[1] ? (values[1][colIdx] || '$0') : '$0';
const totalRev = values[2] ? (values[2][colIdx] || '$0') : '$0';
const totalExp = values[15] ? (values[15][colIdx] || '$0') : '$0';
const netProfit = values[17] ? (values[17][colIdx] || '$0') : '$0';
const margin = values[18] ? (values[18][colIdx] || '0%') : '0%';

const ytdRev = values[2] ? (values[2][13] || '$0') : '$0';
const ytdExp = values[15] ? (values[15][13] || '$0') : '$0';
const ytdProfit = values[17] ? (values[17][13] || '$0') : '$0';

const message = `:bar_chart: *Monthly Finance Report — ${monthName} ${now.getFullYear()}*

:money_with_wings: *Revenue*
• Recurring: ${recurring}
• One-Time: ${oneTime}
• *Total: ${totalRev}*

:receipt: *Expenses*
• *Total: ${totalExp}*

:chart_with_upwards_trend: *Profit*
• *Net Profit: ${netProfit}*
• *Margin: ${margin}*

:calendar: *Year to Date*
• Revenue: ${ytdRev}
• Expenses: ${ytdExp}
• Profit: ${ytdProfit}`;

return [{ json: { message } }];
```
</details>

4. **Send to Slack** (HTTP Request)
   - POST to `https://slack.com/api/chat.postMessage`
   - Sends the formatted report to your channel

---

## Step 5: Register the Stripe Webhook

After activating the n8n workflow, you need to tell Stripe where to send events.

### What is a Stripe Webhook?

A webhook is a URL that Stripe calls every time something happens (payment, cancellation, etc.). Instead of you polling Stripe for updates, Stripe pushes events to your n8n workflow in real-time.

### How to Register It

**Option A: Stripe Dashboard (recommended)**

1. Go to [Stripe Dashboard](https://dashboard.stripe.com) → **Developers** → **Webhooks**
2. Click **Add endpoint**
3. Enter your webhook URL:
   ```
   https://your-workspace.app.n8n.cloud/webhook/stripe-finance-webhook
   ```
4. Select events to listen for:
   - `charge.succeeded` — fires when a payment goes through
   - `customer.subscription.deleted` — fires when a subscription is cancelled
5. Click **Add endpoint**

**Option B: Via Stripe API**

```bash
curl https://api.stripe.com/v1/webhook_endpoints \
  -u "rk_live_YOUR_KEY:" \
  -d "url=https://your-workspace.app.n8n.cloud/webhook/stripe-finance-webhook" \
  -d "enabled_events[0]=charge.succeeded" \
  -d "enabled_events[1]=customer.subscription.deleted"
```

### What the Webhook Needs

- **URL**: Must be publicly accessible (n8n cloud handles this automatically)
- **Events**: `charge.succeeded` and `customer.subscription.deleted`
- **Method**: POST (Stripe always sends POST requests)
- **Content-Type**: `application/json`

---

## Step 6: Test It

Send a test webhook to verify everything works:

```bash
curl -X POST https://your-workspace.app.n8n.cloud/webhook/stripe-finance-webhook \
  -H "Content-Type: application/json" \
  -d '{
    "id": "evt_test_123",
    "type": "charge.succeeded",
    "data": {
      "object": {
        "id": "ch_test_123",
        "amount": 50000,
        "currency": "usd",
        "customer": null,
        "invoice": null,
        "created": 1719792000,
        "status": "succeeded"
      }
    }
  }'
```

You should see:
- A new row in your Revenue tab
- A Slack notification in your channel

**Delete the test row** from Revenue after confirming.

---

## Slack Message Examples

**Payment alert:**
```
:moneybag: New Payment Received

Client: Acme
Amount: $500
Date: 2026-03-15
Type: Retainer
```

**Churn alert:**
```
:rotating_light: Customer Churned

Client: Acme
Date: 2026-04-01
```

**Monthly report:**
```
:bar_chart: Monthly Finance Report — Mar 2026

:money_with_wings: Revenue
• Recurring: $2,500.00
• One-Time: $0.00
• Total: $2,500.00

:receipt: Expenses
• Total: $800.00

:chart_with_upwards_trend: Profit
• Net Profit: $1,700.00
• Margin: 68.0%

:calendar: Year to Date
• Revenue: $7,500.00
• Expenses: $2,400.00
• Profit: $5,100.00
```

---

## Gotchas & Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Webhook returns 404 | Workflow not activated | Activate the workflow in n8n |
| Switch node doesn't match events | Wrong expression path | Use `$json.body.type` not `$json.type` |
| Customer name shows "Unknown" | No customer on the charge | One-time payments without a Stripe Customer object |
| Workflow invisible in n8n UI | Node IDs aren't UUIDs | Recreate with proper `uuid.uuid4()` IDs |
| P&L shows $0 everywhere | Wrong column references in formulas | Payment Status must be column E, use formulas from this guide |
| Executive Summary shows #REF! | Formulas reference a deleted tab | Use Revenue-based formulas from this guide |
| n8n API returns 405 | Wrong HTTP method | Use `POST /workflows/{id}/activate`, not `PATCH` |
| Slack message not posting | Missing scopes | Bot needs `chat:write` and `chat:write.public` |

**Key technical detail**: n8n's Webhook node wraps the POST body under `$json.body`. Every expression in downstream nodes must reference `$json.body.type`, `$json.body.data.object.customer`, etc. If you use a Stripe Trigger node instead, data is at `$json.type` directly — but the Stripe Trigger has reliability issues on n8n cloud.

---

## File Structure

```
.
├── SKILL.md       # This guide
└── LICENSE        # MIT License — 300Growth
```

---

## License

MIT License — Copyright (c) 2026 300Growth. See [LICENSE](LICENSE) for details.

Free to use, modify, and distribute. Attribution appreciated but not required.
