# Stripe Finance Automation

**Fully automated finance tracking system.** Stripe payments and cancellations are logged to Google Sheets in real-time via n8n, with Slack notifications and a monthly P&L report.

Built by [300Growth](https://300growth.com). Free to use.

---

## What You Get

- **Real-time payment logging** — Every Stripe charge automatically creates a row in Google Sheets
- **Churn detection** — Subscription cancellations are logged instantly
- **Slack alerts** — Get notified on every payment and every cancellation
- **Monthly P&L report** — Revenue, expenses, profit, and margins posted to Slack on the 1st of every month
- **Auto-calculated P&L** — Google Sheets formulas handle monthly breakdowns across 12 months
- **Executive dashboard** — MRR, active clients, profit margins, all updating automatically

## How It Works

```
Stripe → n8n Webhook → Google Sheets + Slack
```

Stripe sends webhook events to n8n. n8n routes payments and cancellations, fetches customer info from the Stripe API, formats the data, appends it to Google Sheets, and sends a Slack notification. A separate scheduled workflow posts a monthly finance report.

## What You Need

- [Stripe](https://stripe.com) account
- [n8n](https://n8n.io) account (cloud or self-hosted)
- [Google Sheets](https://sheets.google.com) account
- [Slack](https://slack.com) workspace

## Setup

Full step-by-step guide in **[SKILL.md](SKILL.md)** — covers every API key, every formula, every n8n node configuration, and common gotchas.

## License

MIT — Copyright (c) 2026 300Growth. Free to use, modify, and distribute.
