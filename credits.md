---
title: "Credits and Billing"
description: "Scala uses a credit-based billing model. Your account has a credit balance that is consumed as you run simulations. You can check your balance at any time,..."
---

Scala uses a credit-based billing model. Your account has a credit balance that is consumed as you run simulations. You can check your balance at any time, and the platform will prevent new simulation launches when your balance reaches zero.

This page covers the common workflow — checking your balance, reviewing spending, and understanding the 402 rejection. For daily breakdowns, line-item inspection, and event history, see [Deeper Cost Analysis](#deeper-cost-analysis). Full response schemas and every query parameter for every endpoint live in the auto-generated [Billing API Reference](./api-reference/billing.md).

## How Credits Work

Credits are consumed based on the resources your simulations use — primarily cluster size, instance type, and runtime. Larger clusters with more capable instance types cost more credits per hour. Credits are deducted as simulations progress, and your balance updates accordingly.

Contact your account administrator for details on your platform's specific rates.

## Checking Your Balance

Query your credit balance:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/billing/balance" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response (abbreviated):

```json
{
  "balances": [
    {
      "accountName": "production",
      "currentBalance": "4250.00",
      "updatedAt": "2026-04-16T14:30:00Z"
    }
  ],
  "pagination": { "count": 1, "hasMore": false }
}
```

The `currentBalance` field determines whether you can launch new simulations. The response includes every account on your platform; if you have many accounts, follow the `nextCursor` pagination described in the [Billing API Reference](./api-reference/billing.md).

## Understanding Your Overall Spend

For a high-level view of spending over a period:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/billing/costs/summary?startDate=2026-04-01&endDate=2026-04-15" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

This returns totals aggregated by rate and workspace — the fastest way to answer "what did I spend this month?"

For a daily breakdown or line-item inspection, see [Deeper Cost Analysis](#deeper-cost-analysis).

## When You Run Out of Credits

If your credit balance reaches zero or goes negative, the platform rejects new simulation launches with **402 Payment Required**:

```json
{
  "error": {
    "code": "PAYMENT_REQUIRED",
    "message": "Insufficient credit balance. Add credits before launching simulations."
  }
}
```

### What stops and what keeps running

- **Already-running simulations continue.** They are not terminated and will complete normally, even if this pushes your balance negative.
- **New launches are blocked** until your balance is positive again.
- **Overdraft**: a running simulation can push your balance below zero by up to the cost of that simulation's full runtime. The platform does not currently enforce a hard overdraft cap, but all consumption is tracked and reflected in `currentBalance` once the simulation completes. Extreme negative balances can occur if you launch a very large, long-running simulation on a near-empty balance — the gate only checks at launch time.

### Resuming Operations

Adding credits is a manual process — contact your account administrator. Once credits are applied, simulation launches resume immediately. No manual re-enablement step is required.

---

## Deeper Cost Analysis

These endpoints serve power-user workflows (auditing, spend attribution, troubleshooting). The examples below show the most common filters; the full parameter list for each endpoint is in the [Billing API Reference](./api-reference/billing.md).

### Daily Cost Breakdown

See spending per day, with optional workspace or user attribution:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/billing/costs?startDate=2026-04-01&endDate=2026-04-15&workspaceName=prod" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response (abbreviated):

```json
{
  "costs": [
    {
      "date": "2026-04-15",
      "rateName": "cluster",
      "cost": "20.00",
      "workspaceName": "prod"
    }
  ],
  "pagination": { "count": 1, "hasMore": false }
}
```

Filters: `workspaceName`, `userName`, `eventType` (e.g., `cluster`, `workspace`).

> **Date range cap**: `/costs`, `/costs/summary`, and `/events` accept date windows up to 366 days per request. For longer windows, paginate by month.

> **Note on `userName`**: any caller with valid M2M service account credentials can filter costs by any user on the platform. (All billing endpoints require M2M credentials; regular user API tokens receive a 403.) If you want per-user spend scoped to the individual user or admins only, contact your account administrator — this is a platform-level policy decision, not an API capability.

### Active and Historical Billing Items

Billing items represent what is currently (or was) being billed. Useful for answering "what's running up charges right now?":

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/billing/items?status=active&eventType=cluster" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Billing Event History

Event stream covering resource lifecycle transitions (start, stop):

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/billing/events?startDate=2026-04-01&endDate=2026-04-15" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Filters: `workspaceName`, `userName`, `eventType`, date range.

All date parameters are optional. When omitted, `startDate` defaults to 30 days ago and `endDate` defaults to today.

---

## Error Responses

| Status | Code | When |
|--------|------|------|
| 401 | `UNAUTHORIZED` | Missing or invalid authentication token |
| 402 | `PAYMENT_REQUIRED` | Credit balance is zero or negative (simulation launch only) |
| 403 | `FORBIDDEN` | Balance, cost, and event queries require service account credentials |
| 502 | `BAD_GATEWAY` | Billing service is temporarily unavailable |

Error responses follow the shared [ErrorResponse shape](./api-reference/index.md) — an `error` object with `code` and `message`. The 402 message is deliberately generic; the exact balance figure is logged server-side for audit but not echoed to the caller.

## Next Steps

- [Authentication](./authentication.md) — obtaining an API token
- [Billing API Reference](./api-reference/billing.md) — every endpoint, every parameter, full response schemas
- [Simulations API Reference](./api-reference/simulations.md) — launch endpoint details
