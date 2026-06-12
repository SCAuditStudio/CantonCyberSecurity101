# Web Security 101

## 1. Validate Identity at Every Trust Boundary

- Use a maintained
  [OIDC or OAuth 2.0 implementation](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html).
- Use authorization code flow with PKCE for browser clients.
- Validate tokens according to
  [JWT best practices](https://www.rfc-editor.org/rfc/rfc8725):
  - Signature.
  - Issuer.
  - Audience.
  - Expiration.
  - Not-before time.
  - Intended token use.
- Keep user tokens and machine credentials in separate audiences.
- Map application users to ledger users and party rights on the server.
- Never accept `actAs`, `readAs`, role, owner, or party authority only from the
  browser request body.
- Require
  [phishing-resistant MFA](https://pages.nist.gov/800-63-4/sp800-63b.html)
  for privileged users.

### Simple Example

- Risky:

```javascript
await ledger.submit({
  actAs: req.body.party,
  commands: req.body.commands
});
```

- Safer:

```javascript
const ledgerParty = await rightsForUser(req.auth.sub);
const commands = validateAllowedCommands(req.body.commands);

await ledger.submit({
  actAs: [ledgerParty],
  commands
});
```

## 2. Authorize Every Object and Action

- Enforce
  [authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
  on the server.
- Check permission for every:
  - Contract read.
  - Command submission.
  - File export.
  - Search endpoint.
  - Administrative action.
- Resolve allowed contract IDs from the authenticated user's scope.
- Use deny-by-default policies.
- Give application services only the ledger rights they need.
- Prevent mass assignment of role, party, owner, status, or permission fields.
- Test
  [horizontal object authorization](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
  between users at the same role.
- Test vertical privilege escalation from normal user to administrator.

### Simple Example

```javascript
app.get("/contracts/:id", async (req, res) => {
  const contract = await loadContract(req.params.id);

  if (!canRead(req.auth.sub, contract)) {
    return res.sendStatus(404);
  }

  res.json(toPublicView(contract));
});
```

## 3. Constrain Sessions and Tokens

- Prefer
  [secure, HTTP-only cookies](https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/Cookies)
  for browser sessions.
- Set an appropriate `SameSite` cookie policy.
- Use short-lived access tokens.
- Rotate refresh tokens after use.
- Revoke sessions after:
  - Password changes.
  - MFA changes.
  - Permission changes.
  - Device loss.
  - Account recovery.
- Require recent authentication for key export and administrative actions.
- Protect cookie-authenticated state changes against
  [CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).
- Do not store long-lived sensitive tokens in browser local storage.
- Invalidate server-side sessions during logout.

### Simple Example

```http
Set-Cookie: session=<opaque-id>; Secure; HttpOnly; SameSite=Lax; Path=/
```

## 4. Validate Input and Encode Output

- Apply
  [strict input validation](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
  to every untrusted value.
- Allow expected values instead of blocking known-bad values.
- Set limits for:
  - Length.
  - Numeric range.
  - Format.
  - Enum membership.
  - Array size.
  - Nesting depth.
- Validate data again before ledger submission.
- Use framework output escaping.
- Avoid raw HTML insertion.
- Sanitize rich text with a maintained allowlist library.
- Use parameterized database queries.
- Deploy a restrictive
  [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).

### Simple Example

```javascript
const transferSchema = z.object({
  receiver: z.string().min(1).max(255),
  amount: z.number().positive().max(1_000_000)
});

const transfer = transferSchema.parse(req.body);
```

### Simple Content Security Policy

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'
```

## 5. Harden APIs and Ledger Submission

- Follow
  [REST security guidance](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)
  and use TLS for all external and internal API traffic.
- Authenticate service-to-service requests.
- Allow only required CORS origins, methods, and headers.
- Add rate limits to:
  - Authentication routes.
  - Expensive searches.
  - Exports.
  - Ledger command routes.
- Set request body and upload limits.
- Set request timeouts.
- Enforce pagination limits.
- Use
  [command deduplication](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/command-deduplication.html)
  and idempotency keys for retryable user operations.
- Return generic client errors.
- Keep stack traces and internal service details in protected logs.
- Validate the final command and party set immediately before submission.

### Simple Example

```http
POST /api/transfers
Idempotency-Key: 2f4516dd-3f38-4a71-92f3-f762a3b1a751
Authorization: Bearer <short-lived-token>
Content-Type: application/json
```

## 6. Protect Secrets and the Software Supply Chain

- Never place signing keys or client secrets in browser bundles.
- Store production secrets using a defined
  [secrets-management process](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html).
- Use short-lived workload identity where possible.
- Keep `.env`, private key, backup, and credential files out of Git.
- Pin dependency versions.
- Review lockfile changes.
- Pin third-party CI actions to immutable revisions.
- Run dependency, secret, and container scans in CI.
- Remove unused packages and services.
- Generate an
  [SBOM](https://cheatsheetseries.owasp.org/cheatsheets/Dependency_Graph_SBOM_Cheat_Sheet.html)
  for production releases.
- Verify release provenance before deployment.

### Simple `.gitignore` Entries

```gitignore
.env
.env.*
*.pem
*.key
*.p12
backups/
secrets/
```

## 7. Log Security Decisions Without Leaking Data

- Follow a consistent
  [security logging vocabulary](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Vocabulary_Cheat_Sheet.html)
  and record:
  - Actor.
  - Action.
  - Target identifier.
  - Authorization result.
  - Timestamp.
  - Correlation ID.
- Do not log:
  - Access or refresh tokens.
  - Private keys.
  - Passwords.
  - Onboarding secrets.
  - Complete sensitive contract payloads.
- Alert on:
  - Repeated authorization failures.
  - MFA or password resets.
  - Role and party-right changes.
  - Unusual exports.
  - New administrative sessions.
- Make administrative actions reviewable.
- Use
  [distributed tracing](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/observe/open-tracing.html)
  to connect a web request to its ledger submission.
- Protect log access and retention.

### Simple Example

```json
{
  "event": "ledger_command_rejected",
  "actor": "user-1842",
  "action": "Vault.Withdraw",
  "target": "vault-ref-72",
  "reason": "party_not_authorized",
  "correlation_id": "req-813afc"
}
```

## 8. Threat Model the Whole Application

- Use a structured
  [threat-modeling process](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
  and identify assets before listing controls.
- Include:
  - Ledger authority.
  - Contract data.
  - User identities.
  - API credentials.
  - Administrative functions.
  - Package and deployment authority.
- Draw trust boundaries between:
  - Browser and backend.
  - Backend and Ledger API.
  - Backend and identity provider.
  - Internal services.
  - Application and third-party integrations.
- List attacker types:
  - Unauthenticated internet user.
  - Authenticated malicious user.
  - Compromised employee account.
  - Compromised service.
  - Malicious dependency or vendor.
- Write
  [abuse cases](https://cheatsheetseries.owasp.org/cheatsheets/Abuse_Case_Cheat_Sheet.html)
  for high-value operations.
- Revisit the threat model when adding a new role, integration, or data flow.
- Convert important threats into tests and monitoring rules.

### Simple Threat Table

| Asset | Threat | Control | Detection |
| --- | --- | --- | --- |
| Ledger party rights | User supplies another party ID | Server-side user-to-party mapping | Rejected party mismatch |
| Contract data | IDOR exposes another user's record | Object authorization | Cross-tenant access alert |
| Admin API | Stolen session changes rights | MFA and recent authentication | Privilege-change event |

## 9. Preserve Transaction Intent in the User Interface

- Preserve
  [transaction authorization](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html)
  by displaying the action the server will actually submit.
- Show:
  - Asset or contract involved.
  - Amount and unit.
  - Counterparty.
  - Fees.
  - Deadline.
  - Irreversible consequences.
- Do not hide critical changes behind generic confirmation text.
- Bind confirmation to the exact server-side operation.
- Re-check authorization and current state after confirmation.
- Expire stale confirmations.
- Require stronger confirmation for:
  - Large transfers.
  - New recipients.
  - Party-right changes.
  - Key export.
  - Contract or package upgrades.
- Protect against duplicate clicks and browser retries with
  [command deduplication](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/command-deduplication.html).
- Provide a stable operation ID and final status.
- Do not report success before ledger completion.

### Simple Confirmation Object

```json
{
  "operation_id": "op-7f31",
  "action": "Vault.Withdraw",
  "asset": "USD",
  "amount": "2500.00",
  "receiver": "Acme Treasury",
  "expires_at": "2026-06-12T12:05:00Z"
}
```

- The server stores this object.
- Confirmation refers to `operation_id`.
- The server rejects modified or expired details.

## 10. Enforce Tenant and Party Isolation

- Follow
  [multi-tenant isolation guidance](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
  and treat tenant IDs, organization IDs, and ledger parties as authorization
  attributes, not filters supplied by the client.
- Derive tenant context from the authenticated identity.
- Scope every query by tenant before returning data.
- Include tenant ownership in cache keys.
- Separate tenant-specific:
  - Encryption keys where required.
  - Queues and event subscriptions.
  - Object storage paths.
  - Exports.
  - Audit logs.
- Do not expose sequential internal identifiers when they make enumeration easy.
- Test cross-tenant access for every object type.
- Test background jobs and admin tools, not only public API routes.
- Review whether a broad ledger `readAs` right defeats application-level tenant
  isolation.
- Avoid sharing service identities across tenants when stronger isolation is
  required.

### Simple Example

- Risky:

```sql
SELECT * FROM contracts WHERE id = :contract_id;
```

- Safer:

```sql
SELECT *
FROM contracts
WHERE id = :contract_id
  AND tenant_id = :authenticated_tenant_id;
```

## 11. Secure Webhooks, File Uploads, and Outbound Requests

- Treat webhook payloads as untrusted input.
- Verify webhook signatures over the raw request body.
- Include timestamps and reject old replayed messages.
- Make webhook processing idempotent.
- Allowlist expected event types.
- For
  [file uploads](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html):
  - Limit size.
  - Verify content independently of the filename.
  - Rename files.
  - Store outside the web root.
  - Scan before processing.
- For server-side URL fetching, apply
  [SSRF protections](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html):
  - Allowlist destinations where possible.
  - Block loopback, link-local, and private network targets.
  - Re-check resolved IP addresses.
  - Restrict redirects.
  - Limit response size and time.
- Do not place secrets in webhook URLs.
- Separate inbound integration credentials by vendor.

### Simple Webhook Checks

```text
1. Read raw body.
2. Verify HMAC signature.
3. Verify timestamp is within allowed skew.
4. Reject an already processed event ID.
5. Validate the event schema.
6. Queue processing.
7. Return a generic response.
```

## 12. Design for Abuse and Availability Failures

- Apply
  [denial-of-service controls](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html)
  and rate-limit by more than IP address.
- Consider user, tenant, token, endpoint, and operation cost.
- Place lower limits on:
  - Login and recovery.
  - Expensive searches.
  - Exports.
  - Command submission.
  - Notification generation.
- Set concurrency and queue limits.
- Use backpressure instead of unbounded work queues.
- Apply timeouts to databases, identity providers, Ledger API calls, and vendor
  APIs.
- Use circuit breakers for failing dependencies.
- Bound retries and add jitter.
- Cache only data safe to cache.
- Define degraded behavior when the ledger or identity provider is unavailable.
- Monitor rejection rate, saturation, queue age, and documented
  [application latency and throughput](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/optimise/latency-and-throughput.html).
- Test whether one tenant can exhaust shared resources.

### Simple Retry Policy

```text
Retry only temporary failures:
- Maximum 3 attempts
- Exponential backoff
- Random jitter
- Same idempotency key
- No retry for authorization or validation failure
```

## 13. Protect Caches, Queues, and Background Workers

- Apply the same
  [authorization rules](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
  outside request handlers.
- Do not place complete sensitive contract payloads in broadly accessible
  queues.
- Encrypt sensitive queue and cache data where required.
- Include tenant and authorization scope in cache keys.
- Set short retention for sensitive temporary data.
- Make jobs idempotent.
- Authenticate producers and consumers.
- Restrict which services can publish privileged job types.
- Validate job payloads before processing.
- Use dead-letter queues with protected access.
- Avoid logging complete failed job payloads.
- Test whether replaying a message repeats a transfer or notification.
- Re-authorize long-delayed jobs before performing sensitive actions.

### Simple Job Envelope

```json
{
  "job_id": "job-921",
  "tenant_id": "tenant-17",
  "actor_id": "user-1842",
  "operation": "submit_transfer",
  "operation_id": "op-7f31",
  "created_at": "2026-06-12T12:00:00Z"
}
```

- The worker reloads the operation from trusted storage.
- It does not trust arbitrary commands embedded in the message.

## 14. Build a Security Testing Program

- Use the
  [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/)
  instead of relying on ad hoc testing.
- Add tests for:
  - Authentication bypass.
  - Horizontal and vertical authorization.
  - Session invalidation.
  - CSRF.
  - Input boundaries.
  - Duplicate and replayed operations.
  - Tenant isolation.
  - Rate limits.
  - Sensitive logging.
- Run static analysis and dependency scanning in CI.
- Test the running application using the
  [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/).
- Review APIs against their OpenAPI or AsyncAPI definitions.
- Add regression tests for every security bug.
- Test production-like identity and party mappings in staging.
- Verify security headers and cookie attributes automatically.
- Perform manual testing of high-value business workflows.
- Include background jobs, admin tools, and failure paths.

### Minimal Release Gate

```text
- Unit and integration tests pass
- Authorization regression suite passes
- Dependency and secret scans pass
- Security headers checked
- API schema changes reviewed
- Daml negative tests pass
- High-value workflow manually reviewed
```

## Final Web Checklist

- [ ] Tokens are fully validated.
- [ ] Browser-provided party authority is ignored.
- [ ] Every object and action is authorized server-side.
- [ ] Ledger users follow least privilege.
- [ ] Sessions are short-lived and revocable.
- [ ] CSRF defenses are enabled where required.
- [ ] Inputs use strict schemas and limits.
- [ ] Outputs are safely encoded.
- [ ] CSP and other security headers are configured.
- [ ] API rate, size, timeout, and pagination limits exist.
- [ ] Retryable operations are idempotent.
- [ ] Secrets never reach browser bundles or Git.
- [ ] Dependencies and CI actions are pinned and scanned.
- [ ] Security logs omit sensitive payloads.
- [ ] Alerts cover authentication and authorization abuse.
- [ ] A current threat model covers all trust boundaries.
- [ ] Transaction confirmation is bound to exact server-side intent.
- [ ] Tenant context is derived from authenticated identity.
- [ ] Webhooks, uploads, and outbound requests are constrained.
- [ ] Rate limits account for user, tenant, and operation cost.
- [ ] Queues and workers re-validate sensitive operations.
- [ ] Security regression tests cover every prior security defect.
