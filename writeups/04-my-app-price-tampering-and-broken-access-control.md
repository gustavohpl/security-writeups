# Case Study — Securing My Own App (newburguer): Price Tampering & Broken Access Control

- **Target:** `newburguer` — a food-ordering web app I built and operate.
  Source: https://github.com/gustavohpl/newburguer-lanches
- **Stack:** React SPA + Supabase Edge Functions (Deno + Hono), KV store, PIX/PagSeguro payments
- **Category:** Business Logic / Broken Access Control (OWASP A01, A04)
- **Severity:** Critical (2 findings)
- **Status:** ✅ Found, fixed, **deployed to production, and verified live**

> Authorized by ownership. This is my own application — I tested it, fixed the
> issues, deployed the fix, and confirmed it live. Payment tokens and secrets
> are configured via environment variables (not in the repo).

## 1. Context & architecture

The app is a multi-tenant delivery storefront. The frontend is a React SPA;
the backend is a single Supabase Edge Function (`make-server-dfe23da2`) built
with Hono, split into sub-routers (`routes_orders`, `routes_auth`,
`routes_config`, …) over a KV store. Checkout is a **public guest flow**
(customers order without an account); store management (order status,
cancellation, cleanup) is meant to be **admin-only**, gated by a
session-token middleware (`requireAdmin`, header `X-Admin-Token`).

Two issues broke those trust boundaries.

## 2. Methodology

I ran my own testing harness against a local copy and mapped the API surface,
then reasoned about the app's invariants (what *should* always be true):
"an order total can never be negative", "only an admin can destroy orders".
I probed each invariant directly rather than relying on scanner signatures —
which is exactly how both findings surfaced, and how I discarded a false one
(see §6).

## 3. Finding 1 — Client-controlled order total (price tampering)

### Analysis
`POST /orders` sanitized the *text* fields but then persisted the order by
spreading the raw request body — including the money fields — straight into
storage, with **no server-side re-pricing**:

```ts
// BEFORE — server trusts client-sent totals
const order = { ...body, id, orderId, status: body.status || 'pending', createdAt };
await kv.set(`order:${orderId}`, order);
```

So `total` was whatever the client sent. A crafted request with a negative
quantity/total was accepted and the order created — free (or profitable)
checkout.

### PoC
```bash
curl -X POST 'https://<app>/functions/v1/make-server-dfe23da2/orders' \
  -H 'Content-Type: application/json' -H 'apikey: <anon>' -H 'Authorization: Bearer <anon>' \
  -d '{"items":[{"productId":"p1","name":"Combo","quantity":1,"price":50}],"total":-13.37}'
# BEFORE: 200, order created with total = -13.37
```

### Fix
A server-side validation + re-pricing module (`order_validation.tsx`) makes
the server the source of truth:

- Quantities must be integers in `1..99`.
- Every money field must be finite and `>= 0` (kills negative totals).
- Each item price is floored at the **catalog** price (`product:<id>`) — the
  client cannot underprice an item.
- The coupon discount is **re-derived server-side** from the coupon record
  (type/value/active/uses/expiry), never trusted from the client.
- Total is recomputed as `max(0, subtotal - discount) + deliveryFee`, and a
  divergence from the client's total is logged as a tamper attempt.

```ts
// AFTER — server recomputes and overrides the money fields
const pricing = await validateAndPriceOrder(body);
if (!pricing.ok) return error(c, pricing.message, pricing.status || 400);
body.subtotal = pricing.subtotal;
body.discount = pricing.discount;
body.deliveryFee = pricing.deliveryFee;
body.total = pricing.total;               // authoritative
```

### Live verification (after deploy)
```
POST /orders  {"quantity":-1, "total":-13.37}
-> HTTP 400  {"error":"Quantidade inválida ... inteiro entre 1 e 99."}
```

## 4. Finding 2 — Admin order endpoints missing authorization

### Analysis
Three state-changing endpoints under `/admin/…` were registered **without**
the `requireAdmin` middleware that guards the rest of the admin surface — so
any anonymous caller could invoke them:

```ts
// BEFORE — no auth guard
router.post('/admin/migrate-scale', async (c) => { /* ... */ });
router.delete('/admin/orders/clear-all', async (c) => { /* wipes ALL orders */ });
router.put('/admin/orders/:id/cancel', async (c) => { /* cancels any order */ });
```

`clear-all` deletes every active **and** archived order — an anonymous
denial-of-service against the business.

### Fix
```ts
// AFTER — same token middleware already used by /orders/history
router.post('/admin/migrate-scale', requireAdmin, async (c) => { /* ... */ });
router.delete('/admin/orders/clear-all', requireAdmin, async (c) => { /* ... */ });
router.put('/admin/orders/:id/cancel', requireAdmin, async (c) => { /* ... */ });
```

### Live verification (after deploy)
```
POST /admin/migrate-scale   (no admin token)
-> HTTP 401  {"error":"Autenticação necessária: token de admin não fornecido"}
```

## 5. Deployment

Fixed in both server copies, committed (`b690dc0`), and deployed with the
Supabase CLI:

```
supabase functions deploy make-server-dfe23da2 --project-ref <ref>
# Deployed Functions: make-server-dfe23da2  (version 13)
```

The two live checks above confirm the fix is active in production.

## 6. A finding I *dismissed* (and why)

An automated pass suggested a "mass-assignment → set `role`/`isAdmin` → admin
takeover" chain. I traced the auth model before acting: the backend has **no**
`role`/`isAdmin` field at all — admin access is gated by server-issued session
tokens (`admin_session:<token>`), and no user-writable field maps to
privilege. So the reported chain **does not exist** in this codebase. I did
not "fix" a non-issue. Verifying a finding against the actual code — instead
of trusting a scanner label — is the difference between a real report and
noise.

## 7. Lessons

- Never trust client-supplied money; recompute totals server-side and floor
  invariants (`total >= 0`, `quantity >= 1`).
- Authorization is per-route — a path prefix like `/admin/` is not a guard;
  the middleware has to actually be attached.
- Confirm every finding against the code before reporting or remediating.

## Residual hardening (tracked)
- Validate delivery fee against configured delivery sectors (currently floored at `>= 0`).
- Rate-limit the guest checkout (already present on reviews).

## References
- OWASP Top 10 — A01:2021 Broken Access Control, A04:2021 Insecure Design
- CWE-602 (client-side enforcement of server security), CWE-862 (missing authorization)
