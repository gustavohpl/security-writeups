# My Own App (React + Supabase) — Price Tampering & Broken Access Control (Found & Fixed)

- **Target:** A food-ordering web application I built and operate (React SPA + Supabase Edge Functions / Deno + Hono). Details sanitized — no live host, project reference, or keys are included.
- **Category:** Business Logic / Broken Access Control (OWASP A01, A04)
- **Severity:** Critical
- **Status:** ✅ Fixed (remediation applied to the codebase)

> Authorized by ownership: this is my own application. I security-tested it,
> found the issues below, and fixed them. This write-up shows the full loop —
> discovery, impact, and remediation — with the sensitive infrastructure
> details removed.

## Context

The checkout flow is a public (guest) endpoint — customers order without an
account — while store management (order status, cancellation, cleanup) is
meant to be admin-only. Two issues broke those assumptions.

---

## Finding 1 — Client-controlled order total (price tampering)

### The bug

The `POST /orders` handler persisted the order using the totals **sent by the
client**, without recomputing them from trusted catalog prices:

```ts
// BEFORE (vulnerable) — server trusts client-sent totals
const order = { ...body, id, orderId, status: body.status || 'pending', createdAt: ... };
await kv.set(`order:${orderId}`, order);
```

Because `body` was spread straight from the request, a crafted request could
set an arbitrary `total` — including a **negative** one. A negative quantity /
negative total was accepted and the order created, i.e. the customer could
check out for free or at an attacker-chosen price.

### Proof of concept

```bash
curl -s -X POST 'https://<APP>/functions/v1/<server>/orders' \
  -H 'Content-Type: application/json' \
  -d '{"items":[{"productId":"p1","name":"Combo","quantity":1,"price":50}],
       "total":-13.37,"deliveryType":"pickup"}'
# -> order created with total = -13.37
```

### Impact

- Direct financial loss: goods ordered for free or at a negative/arbitrary price.
- No authentication required (guest checkout).

### The fix

Added a server-side validation + re-pricing module
(`order_validation.tsx`) that makes the server the source of truth:

- Quantities must be integers in `1..99`.
- All monetary fields must be finite and `>= 0` (kills negative totals).
- Each item price is floored at the **catalog** price (`product:<id>`) — the
  client cannot send a price below the current one.
- The coupon discount is **re-derived server-side** from the coupon record
  (type/value/active/uses/expiry), never trusted from the client.
- The total is recomputed as `max(0, subtotal - discount) + deliveryFee` and a
  mismatch against the client total is logged as a tamper attempt.

```ts
// AFTER (fixed) — server recomputes and overrides the money fields
const pricing = await validateAndPriceOrder(body);
if (!pricing.ok) return error(c, pricing.message, pricing.status || 400);
if (pricing.tampered) console.warn('⚠️ [SECURITY] client total != server total');
body.subtotal = pricing.subtotal;
body.discount = pricing.discount;
body.deliveryFee = pricing.deliveryFee;
body.total = pricing.total;                 // authoritative
```

---

## Finding 2 — Admin order endpoints missing authorization

### The bug

Several state-changing endpoints under `/admin/…` were registered **without
the `requireAdmin` middleware**, so any anonymous caller could invoke them:

```ts
// BEFORE (vulnerable) — no auth guard
router.post('/admin/migrate-scale', async (c) => { /* ... */ });
router.delete('/admin/orders/clear-all', async (c) => { /* wipes ALL orders */ });
router.put('/admin/orders/:id/cancel', async (c) => { /* cancels any order */ });
```

### Proof of concept

```bash
# Anonymous request wipes every active and archived order
curl -s -X DELETE 'https://<APP>/functions/v1/<server>/admin/orders/clear-all'
```

### Impact

- Anonymous data destruction: `clear-all` deletes all active **and** archived
  orders; `cancel` cancels arbitrary orders — a denial-of-service against the
  business.

### The fix

Applied the existing token-based `requireAdmin` middleware (the same guard
already used by `/orders/history`) to all three endpoints:

```ts
// AFTER (fixed)
router.post('/admin/migrate-scale', requireAdmin, async (c) => { /* ... */ });
router.delete('/admin/orders/clear-all', requireAdmin, async (c) => { /* ... */ });
router.put('/admin/orders/:id/cancel', requireAdmin, async (c) => { /* ... */ });
```

---

## Verification

- Syntax of the modified Edge Function validated.
- Negative totals / non-positive quantities are now rejected with `400`.
- Admin endpoints return `401` without a valid admin token.

## Residual hardening (tracked)

- Validate the delivery fee against the configured delivery sectors (currently
  only floored at `>= 0`).
- Move the guest checkout behind a rate limit (already present on reviews).

## References

- OWASP Top 10 — A01:2021 Broken Access Control, A04:2021 Insecure Design
- OWASP — Business Logic Vulnerability; CWE-602 (client-side trust), CWE-862 (missing authorization)
