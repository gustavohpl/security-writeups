# OWASP Juice Shop — SQL Injection Authentication Bypass

- **Target:** OWASP Juice Shop (local instance, `http://127.0.0.1:3000`)
- **Category:** Injection / Broken Authentication (OWASP A03, A07)
- **Severity:** Critical
- **Status:** Confirmed (reproduced)

> Authorized environment: OWASP Juice Shop is an intentionally vulnerable
> application, run locally for training.

## Summary

The login endpoint builds its SQL query by concatenating the user-supplied
`email` field directly into the statement. Injecting a tautology in that
field bypasses authentication entirely and returns a valid session token for
the first user in the database (the administrator), with no valid password.

## Steps to reproduce

Send a login request with a SQL injection payload in the `email` field:

```bash
curl -s -X POST http://127.0.0.1:3000/rest/user/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"'"'"' OR 1=1--","password":"anything"}'
```

The `email` value is the payload `' OR 1=1--`. The response is a `200` with an
`authentication` object containing a valid JWT and the admin user record:

```json
{ "authentication": { "token": "eyJhbGciOi...", "umail": "admin@juice-sh.op" } }
```

The trailing `--` comments out the password check, so the `password` field is
irrelevant. The query resolves to the first matching row, which is the admin
account.

## Impact

- Full authentication bypass — log in as any/first user without credentials.
- The returned account is the administrator, granting privileged access to
  the application and its data.
- Because the flaw is in the SQL layer, it also enables data exfiltration via
  UNION-based injection on the same parameter.

## Related findings in the same assessment

Several REST endpoints returned sensitive data with no authentication
(broken access control):

- `GET /api/Feedbacks`, `GET /api/Products`, `GET /api/SecurityQuestions`
- `GET /rest/user/whoami`, `GET /api/Recycles`, `GET /api/Quantitys`

These compound the SQLi by exposing user and system data to anonymous callers.

## Remediation

- Use parameterized queries / prepared statements (or the ORM's binding
  layer) — never concatenate user input into SQL.
- Enforce server-side authentication that verifies a password hash; a login
  query must never return a session on a string match alone.
- Apply authorization checks on every REST endpoint that returns user or
  system data.

## References

- OWASP Top 10 — A03:2021 Injection
- OWASP Cheat Sheet — SQL Injection Prevention
- CWE-89: SQL Injection
