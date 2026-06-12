# Web Security 101

A secure Daml model can still be broken by an unsafe web application. The
backend, browser, identity provider, database, and Ledger API connection are all
part of the security boundary.

## 1. Control Identity and Permissions

- Use a maintained OIDC or OAuth 2.0 library.
- Validate token signature, issuer, audience, expiry, and token type.
- Use short-lived access tokens.
- Require phishing-resistant MFA for administrators.
- Map the authenticated user to ledger parties on the server.
- Never trust `actAs`, `readAs`, tenant ID, role, or owner from the request body.
- Check permission for every read, command, export, search, and admin action.
- Use deny-by-default rules.
- Give each service only the ledger rights it needs.
- Test users with the same role against each other's records.

This mirrors the authorization problem described in our
[Daml vulnerabilities article](https://scauditstudio.com/blog/DamlSmartContractVulnerabilities):
authority must come from trusted state, not caller input.

- **Risky:**

```javascript
await ledger.submit({
  actAs: req.body.party,
  commands: req.body.commands
});
```

- **Safer:**

```javascript
const party = await rightsForUser(req.auth.sub);
const commands = validateAllowedCommands(req.body.commands);

await ledger.submit({
  actAs: [party],
  commands
});
```

## 2. Protect Canton Privacy Outside the Ledger

- Return only the fields the user needs.
- Apply object and tenant checks before loading sensitive data.
- Do not place full contract payloads in broad caches or queues.
- Include tenant and permission scope in cache keys.
- Do not log tokens, private keys, onboarding secrets, or private contract data.
- Use generic errors instead of exposing internal details.
- Protect exports because they can bypass normal page-level limits.
- Review background workers and admin tools for the same privacy rules.
- Keep private data out of analytics and support tools unless approved.
- Match API visibility to the intended Daml observer model.

Canton privacy does not protect data after the application copies it into a
database, log, cache, or browser. Our
[Daml privacy research](https://scauditstudio.com/blog/DamlSmartContractVulnerabilities)
and
[EVM-to-Canton guide](https://scauditstudio.com/blog/EVMtoCantonGuide)
explain why visibility must be reviewed across the whole system.

- **Simple rule:**

```text
Ledger visibility is the maximum allowed visibility.
The web application should usually expose less, never more.
```

## 3. Validate Every Request and Preserve User Intent

- Validate every request with a strict schema.
- Set limits for length, range, format, list size, and nesting.
- Recalculate important values on the server.
- Do not trust a browser-supplied fee, balance, exchange rate, or final amount.
- Show the exact action, amount, asset, counterparty, fee, and deadline before
  confirmation.
- Bind confirmation to a server-side operation ID.
- Expire old confirmations.
- Use idempotency keys for retryable actions.
- Add rate, size, timeout, and pagination limits.
- Protect file uploads and outbound URL requests.
- Report success only after the ledger confirms the operation.

Incorrect economic models often enter through application code, not only Daml.
The ideas in
[Writing Correct Daml Contracts on Canton](https://scauditstudio.com/blog/Writing-Correct-Daml-Contracts-on-Canton)
also apply to server-side fee, share, and settlement calculations.

- **Simple request:**

```json
{
  "operation_id": "op-7f31",
  "action": "Vault.Withdraw",
  "asset": "USD",
  "amount": "2500.00",
  "receiver": "Acme Treasury",
  "expires_at": "2026-06-13T12:05:00Z"
}
```

- The server stores the approved details.
- The browser confirms only the operation ID.
- The server rejects changed or expired details.

## 4. Treat AI Code and Dependencies as Untrusted

- Review AI-generated authentication, authorization, and payment code line by
  line.
- Do not let AI create production permissions or deployment rules without human
  review.
- Do not paste secrets, customer data, private source code, or findings into
  unapproved AI tools.
- Check generated code for missing error handling, unsafe defaults, and made-up
  library functions.
- Require tests based on the security requirement, not only the generated code.
- Pin dependency versions and review lockfile changes.
- Remove unused packages.
- Scan dependencies, containers, source, and secrets in CI.
- Pin third-party CI actions to fixed versions.
- Verify that the deployed artifact is the reviewed artifact.

Our article on
[AI and Formal Verification](https://scauditstudio.com/blog/AI-and-Formal-Verification-In-Defi)
explains why tools cannot prove that business intent is correct. The same issue
appears in web code: a generated endpoint may work while missing the permission
check that matters most.

- **Human review questions:**

```text
- Where does identity come from?
- Where is permission checked?
- Which fields are trusted?
- Can the action be replayed?
- Can one tenant reach another tenant's data?
- What is logged on failure?
```

## 5. Secure Sessions, Releases, and Incident Detection

- Use secure, HTTP-only cookies for browser sessions.
- Protect cookie-based state changes against CSRF.
- Rotate refresh tokens and revoke sessions after account changes.
- Require recent login for key export and admin changes.
- Use TLS for all external and internal traffic.
- Keep production secrets in a secrets manager.
- Separate build, deployment, and runtime credentials.
- Log actor, action, target, result, time, and correlation ID.
- Alert on repeated login failures, permission changes, unusual exports, and
  new admin sessions.
- Trace a web request through the backend to the ledger result.
- Define safe behavior when the identity provider or ledger is unavailable.
- Add regression tests for every security bug.

SCAS recommends preparing the codebase, tests, and team before review in
[Maximizing the Value of Your Smart Contract Audit](https://scauditstudio.com/blog/Maximizing-the-Value-of-Your-Smart-Contract-Audit).
The same preparation makes web application reviews faster and more useful.

- **Simple secure cookie:**

```http
Set-Cookie: session=<opaque-id>; Secure; HttpOnly; SameSite=Lax; Path=/
```

## Final Web Checklist

- [ ] Tokens are fully validated.
- [ ] Administrators use phishing-resistant MFA.
- [ ] Browser-supplied party authority is ignored.
- [ ] Every object and action is authorized on the server.
- [ ] Ledger users and services have minimum rights.
- [ ] Tenant isolation is tested.
- [ ] API responses expose only required fields.
- [ ] Logs and caches exclude private contract data.
- [ ] Inputs use strict schemas and limits.
- [ ] Economic values are recalculated on the server.
- [ ] User confirmation matches the final submitted action.
- [ ] Retryable operations use idempotency.
- [ ] Rate, timeout, upload, and pagination limits exist.
- [ ] AI-generated code received full human review.
- [ ] Secrets never reach Git, browser bundles, or unapproved AI tools.
- [ ] Dependencies and CI actions are pinned and scanned.
- [ ] Sessions are secure, short-lived, and revocable.
- [ ] Security logs can trace requests to ledger results.
- [ ] Alerts cover identity, permission, and export abuse.
- [ ] Security regression tests run before release.
