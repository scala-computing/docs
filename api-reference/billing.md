---
title: "Billing"
description: "Billing balance, costs, items, events, and rates (feature-gated: enable_billing_endpoints)"
---

Billing balance, costs, items, events, and rates (feature-gated: enable_billing_endpoints)

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/billing/balance` | Get credit balances |
| GET | `/api/v1/billing/costs` | Query daily costs |
| GET | `/api/v1/billing/costs/summary` | Get cost summary |
| GET | `/api/v1/billing/items` | List billing items |
| GET | `/api/v1/billing/items/{id}` | Get billing item |
| GET | `/api/v1/billing/events` | List billing events |
| GET | `/api/v1/billing/rates` | List rates |

---

## Get credit balances

<span class="api-method api-method-get">GET</span> `/api/v1/billing/balance`

Returns credit balances for the platform. All billing endpoints require M2M (service account) credentials.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `cursor` | query | string | No | Pagination cursor |
| `limit` | query | integer | No | Max results per page (default 100, max 1000) |

### Responses

**200** - Credit balances

```json
{
  "balances": [
    {
      "accountId": "string",
      "accountName": "string",
      "accountType": "string",
      "currentBalance": "string",
      "totalCredits": "string",
      "totalDebits": "string",
      "currency": "string",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid request (e.g., expired or malformed cursor)

**401** - Unauthorized

**403** - Forbidden

**502** - Billing service unavailable

---

## Query daily costs

<span class="api-method api-method-get">GET</span> `/api/v1/billing/costs`

Returns daily cost breakdown with date range and workspace filtering. Date ranges cannot exceed 366 days.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `startDate` | query | string | Yes | Start date (YYYY-MM-DD) |
| `endDate` | query | string | Yes | End date (YYYY-MM-DD) |
| `workspaceName` | query | string | No | Filter by workspace name |
| `userName` | query | string | No | Filter by user name |
| `cursor` | query | string | No | Pagination cursor |
| `limit` | query | integer | No | Max results per page (default 50, max 100) |

### Responses

**200** - Daily costs

```json
{
  "costs": [
    {
      "dailyCostId": 1,
      "billingItemId": 1,
      "accountName": "string",
      "accountId": null,
      "platformName": "string",
      "itemName": "string",
      "rateId": 1,
      "rateName": "string",
      "rateAmount": "string",
      "cost": "string",
      "costCurrency": "USD",
      "quantity": 1,
      "workspaceName": null,
      "workspaceId": null,
      "sandboxName": null,
      "sandboxId": null,
      "userName": null,
      "userId": null,
      "date": "string",
      "details": "..."
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid request (date range too wide, inverted, or malformed cursor)

**401** - Unauthorized

**403** - Forbidden

---

## Get cost summary

<span class="api-method api-method-get">GET</span> `/api/v1/billing/costs/summary`

Returns aggregated cost summary by currency for a date range. Date ranges cannot exceed 366 days.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `startDate` | query | string | Yes | Start date (YYYY-MM-DD) |
| `endDate` | query | string | Yes | End date (YYYY-MM-DD) |

### Responses

**200** - Cost summary

```json
{
  "startDate": "string",
  "endDate": "string",
  "USD": {
    "total": "1234.56",
    "byRate": {},
    "byWorkspace": {}
  },
  "CREDIT": null
}
```

**400** - Invalid request (date range too wide or inverted)

**401** - Unauthorized

**403** - Forbidden

---

## List billing items

<span class="api-method api-method-get">GET</span> `/api/v1/billing/items`

Returns billing line items (active/closed) for the platform.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceName` | query | string | No | Filter by workspace name |
| `status` | query | string (enum) | No | Filter by item status |
| `eventType` | query | string | No | Filter by event type (cluster, workspace, storage, etc.) |
| `cursor` | query | string | No | Pagination cursor |
| `limit` | query | integer | No | Max results per page (default 50, max 100) |

### Responses

**200** - Billing items

```json
{
  "items": [
    {
      "billingItemId": 1,
      "accountName": "string",
      "accountId": null,
      "platformName": "string",
      "itemName": "string",
      "rateName": "string",
      "eventType": "string",
      "eventAction": "string",
      "quantity": 1,
      "workspaceName": null,
      "workspaceId": null,
      "sandboxName": null,
      "sandboxId": null,
      "userName": null,
      "userId": null,
      "startTimestamp": "2024-01-15T10:30:00Z",
      "endTimestamp": null,
      "status": "active",
      "details": "...",
      "metadata": "..."
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid request (e.g., malformed cursor)

**401** - Unauthorized

**403** - Forbidden

---

## Get billing item

<span class="api-method api-method-get">GET</span> `/api/v1/billing/items/{id}`

Returns a single billing item by ID. Returns 404 when the item does not belong to the caller's platform/account scope to avoid leaking cross-tenant existence.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | integer | Yes | Billing item ID |

### Responses

**200** - Billing item

```json
{
  "billingItemId": 1,
  "accountName": "string",
  "accountId": null,
  "platformName": "string",
  "itemName": "string",
  "rateName": "string",
  "eventType": "string",
  "eventAction": "string",
  "quantity": 1,
  "workspaceName": null,
  "workspaceId": null,
  "sandboxName": null,
  "sandboxId": null,
  "userName": null,
  "userId": null,
  "startTimestamp": "2024-01-15T10:30:00Z",
  "endTimestamp": null,
  "status": "active",
  "details": {},
  "metadata": {}
}
```

**401** - Unauthorized

**403** - Forbidden

**404** - Item not found

---

## List billing events

<span class="api-method api-method-get">GET</span> `/api/v1/billing/events`

Returns billing event history for the platform. Date ranges cannot exceed 366 days.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceName` | query | string | No | Filter by workspace name |
| `userName` | query | string | No | Filter by user name |
| `eventType` | query | string | No | Filter by event type (cluster, workspace, storage, etc.) |
| `startDate` | query | string | No | Start date filter |
| `endDate` | query | string | No | End date filter |
| `cursor` | query | string | No | Pagination cursor |
| `limit` | query | integer | No | Max results per page (default 50, max 100) |

### Responses

**200** - Billing events

```json
{
  "events": [
    {
      "billingEventId": 1,
      "accountName": "string",
      "accountId": null,
      "platformName": "string",
      "eventName": "string",
      "eventType": "string",
      "eventAction": "string",
      "rateName": "string",
      "workspaceName": null,
      "workspaceId": null,
      "sandboxName": null,
      "sandboxId": null,
      "userName": null,
      "userId": null,
      "previousCount": 1,
      "currentCount": 1,
      "totalBillableCount": 1,
      "eventTimestamp": "2024-01-15T10:30:00Z",
      "receivedAt": "2024-01-15T10:30:00Z",
      "eventHash": null,
      "details": "..."
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid request (date range too wide, inverted, or malformed cursor)

**401** - Unauthorized

**403** - Forbidden

---

## List rates

<span class="api-method api-method-get">GET</span> `/api/v1/billing/rates`

Returns the rate schedule (price list) for the platform. Paginated per API-0012; consumers must iterate until `pagination.hasMore` is false.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `date` | query | string | No | Filter rates active on this date |
| `userName` | query | string | No | Filter by user name (audit-logged) |
| `cursor` | query | string | No | Pagination cursor |
| `limit` | query | integer | No | Max results per page (default 50, max 100) |

### Responses

**200** - Rate schedule

```json
{
  "rates": [
    {
      "rateId": 1,
      "accountId": null,
      "accountName": "string",
      "platform": "string",
      "rateName": "string",
      "rateType": "...",
      "rateTypeRaw": "string",
      "amount": "string",
      "period": "month",
      "minimumPeriod": null,
      "tierStart": 1,
      "tierEnd": null,
      "startDate": "string",
      "endDate": null,
      "metadata": "..."
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid request (e.g., malformed cursor)

**401** - Unauthorized

**403** - Forbidden

---
