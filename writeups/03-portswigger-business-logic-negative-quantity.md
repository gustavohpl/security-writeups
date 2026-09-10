# PortSwigger Web Security Academy — High-Level Business-Logic Flaw (Negative Quantity)

- **Target:** PortSwigger Web Security Academy lab ("High-level logic vulnerability"), ephemeral instance on `*.web-security-academy.net`
- **Category:** Business Logic (OWASP A04 Insecure Design)
- **Severity:** Critical
- **Status:** Confirmed (reproduced)

> Authorized environment: PortSwigger Web Security Academy labs are provided
> for hands-on testing under PortSwigger's rules of engagement. The instance
> URL is ephemeral and unique per session.

## Summary

The shopping-cart endpoint trusts the client-supplied item `quantity` without
validating that it is a positive integer. Submitting a **negative quantity**
drives the cart total negative (observed at **-12033.0**), which the checkout
logic treats as a credit/discount rather than rejecting it — the classic
"buy expensive items for free (or profit)" business-logic flaw.

## Steps to reproduce

1. Add a low-cost product to the cart to establish a valid session/cart.
2. Add an expensive product with a **negative quantity** so the negative line
   value cancels (and exceeds) the price of the item you actually want:

```bash
curl -s -X POST "https://<LAB-ID>.web-security-academy.net/cart" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --cookie "session=<SESSION>" \
  --data 'productId=1&redir=PRODUCT&quantity=-100'
```

3. Observe the cart total go **negative** (`-12033.0`) — the server accepted
   the negative quantity and recomputed the total without a floor at zero.
4. The negative line item offsets the price of the desired product, allowing
   checkout at or below `0`.

## Impact

- Direct financial loss: goods obtained for free or at attacker-chosen
  (negative) prices.
- No authentication or special privileges required beyond a normal shopping
  session.
- Illustrates why input that is syntactically valid can still be
  business-invalid — a scanner won't catch this; it requires reasoning about
  the application's intended invariants.

## Related observations

- **SAST:** a DOM-XSS sink (`innerHTML`) was identified in the client-side
  code — user-controlled input reaching `innerHTML` without sanitization.
- Product pages exposed sequential integer IDs (`/product?productId=1..20`),
  the pattern that enables IDOR/BOLA testing on object references.

## Remediation

- Validate `quantity` server-side: reject non-positive and non-integer values.
- Recompute cart and order totals server-side from trusted catalog prices;
  never trust a client-supplied total, and floor computed totals at zero.
- Add invariant checks (total ≥ 0, quantity ≥ 1) as explicit business rules,
  and cover them with tests.

## References

- OWASP Top 10 — A04:2021 Insecure Design
- OWASP — Business Logic Vulnerability
- CWE-840: Business Logic Errors; CWE-20: Improper Input Validation
