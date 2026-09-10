# Security Lab Write-ups

Hands-on web application security write-ups. Each one documents a vulnerability
I found, how I confirmed it, its impact, and how to fix it.

> ⚠️ **Scope & authorization.** Every target here is either an intentionally
> vulnerable training application (tested locally or under the lab provider's
> rules of engagement) or **my own application** (authorized by ownership).
> No third-party or production systems belonging to others were touched.

## Write-ups

| # | Target | Type | Headline finding |
|---|--------|------|------------------|
| 1 | [OWASP Juice Shop](writeups/01-juice-shop-sqli-auth-bypass.md) | Training lab (local) | SQL injection → authentication bypass |
| 2 | [OWASP WebGoat (GraphQL)](writeups/02-webgoat-graphql-broken-auth.md) | Training lab (local) | GraphQL broken authentication |
| 3 | [PortSwigger Web Security Academy](writeups/03-portswigger-business-logic-negative-quantity.md) | Vendor lab | Business-logic flaw (negative quantity) |
| 4 | [newburguer — my own app (case study)](writeups/04-my-app-price-tampering-and-broken-access-control.md) | My application | Price tampering + broken access control — **found, fixed, deployed & verified** |

## Methodology

Each engagement followed the same phases: recon and surface mapping →
enumeration → vulnerability identification → controlled proof-of-concept →
impact analysis → remediation. Findings were validated by reproducing them,
not just flagged by a scanner. Write-up #4 is a full loop on an app I built —
through the code-level fix, a production deploy, and live verification.

## About

Independent web application security practice covering the OWASP Top 10 —
injection, broken authentication, broken access control, and business-logic
flaws — plus remediation on my own codebase.
