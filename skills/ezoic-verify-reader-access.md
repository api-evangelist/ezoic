---
name: Verify an Ezoic Subscriptions reader's access
description: >-
  Check whether an identified reader has paid access to a product/item, and list a
  reader's purchases, using the Ezoic Subscriptions Server-to-Server REST API.
api: openapi/ezoic-subscriptions-openapi.yml
operations: [checkAccess, listProducts, listPurchases]
---

# Verify an Ezoic Subscriptions reader's access

Use the Ezoic Subscriptions REST API from your own backend to gate paid content.

Base URL: `https://api-gateway.ezoic.com/subscriptions/v1`

## Authentication (every request)
- Query params: `developerKey` (your Ezoic API key) and `domain` (an account-owned domain).
- Reader identity: send **exactly one** header — `X-Ezoic-Reader-Email` OR `X-Ezoic-Reader-Token`
  (a Subscriptions session JWT). Never put reader identity in the query string.

## Steps
1. **List products** (optional) — `listProducts` (`GET /products`) to discover valid
   product handles configured for the domain.
2. **Check access** — `checkAccess` (`GET /access?product=<handle>`), or use `item=<key>`
   for a one-time item. Read `data.decision`:
   - `allowed` → serve the content.
   - `login_required` → prompt the reader to sign in.
   - `denied` / `expired` / `revoked` → withhold content; inspect `data.reasonCode`
     (`no_entitlement`, `not_started`, `suspended`, `pending`, `customer_required`,
     `unknown_product`) to choose the message.
3. **List purchases** (optional) — `listPurchases` (`GET /purchases`) to show the reader
   their active subscriptions/purchases (`productKey`/`item`, `status`, `expiresAt`).

## Error handling
- Responses are `{ "success": boolean, ... }`; errors are `{ "success": false, "message": "..." }`
  (custom envelope, not RFC 9457). See `errors/ezoic-problem-types.yml`.
- `400` — missing params or duplicate reader identity headers; send exactly one identity header.
- `403` — Subscriptions disabled for the domain, or invalid reader token.
- Access responses carry `Cache-Control: no-store`; do not cache decisions.
- The API is read-only (all GET); no idempotency key needed. See `conventions/ezoic-conventions.yml`.
