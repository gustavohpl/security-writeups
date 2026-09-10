# Security Lab Write-ups

Hands-on web application security write-ups from **deliberately vulnerable
training environments**. Each write-up documents a vulnerability I found,
how I confirmed it, its impact, and how to fix it.

> ⚠️ **Scope & authorization.** Every target here is an intentionally
> vulnerable lab built for security training, tested locally or under the
> lab provider's own rules of engagement. No third-party or production
> systems were touched.

## Environments

| # | Environment | Type | Headline finding |
|---|-------------|------|------------------|
| 1 | [OWASP Juice Shop](writeups/01-juice-shop-sqli-auth-bypass.md) | Local (Docker) | SQL injection → authentication bypass |
| 2 | [OWASP WebGoat (GraphQL)](writeups/02-webgoat-graphql-broken-auth.md) | Local (Docker) | GraphQL broken authentication |
| 3 | [PortSwigger Web Security Academy](writeups/03-portswigger-business-logic-negative-quantity.md) | Vendor lab | Business-logic flaw (negative quantity) |

## Methodology

Each engagement followed the same phases: recon and surface mapping →
authenticated/unauthenticated enumeration → vulnerability identification →
controlled proof-of-concept → impact analysis → remediation notes. Findings
were validated by reproducing them, not just flagged by a scanner.

## About

Independent web application security practice. These labs cover the OWASP
Top 10 categories — injection, broken authentication, broken access control,
and business-logic flaws.
