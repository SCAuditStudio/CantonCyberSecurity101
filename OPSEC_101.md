# OpSec 101

Operational security protects the people, credentials, systems, and recovery
processes around a Canton application. A strong Daml model cannot help if an
administrator, CI pipeline, or participant identity is compromised.

## 1. Protect Keys and Privileged Access

- Inventory signing keys, participant identities, OIDC secrets, cloud
  credentials, CI tokens, database passwords, and recovery material.
- Record an owner and purpose for every secret.
- Keep production, test, development, and personal credentials separate.
- Use managed KMS or HSM-backed storage where supported.
- Require hardware-backed MFA for source control, cloud, CI, identity systems,
  password managers, and production access.
- Use separate daily and administrator accounts.
- Give production access for a limited time.
- Avoid shared administrator accounts.
- Define rotation, revocation, and recovery procedures.
- Encrypt backups and store recovery copies away from the main environment.

The
[SCAS OpSec guide for 2026](https://scauditstudio.com/blog/opsec-best-practices-for-2026)
provides a wider set of controls for identity, devices, secrets, and recovery.

- **Simple secret record:**

```text
Secret: Production participant identity
Owner: Infrastructure security
Storage: Managed secure key store
Used by: Production participant only
Rotation: Emergency procedure
Recovery: Two-person approved restore
```

## 2. Reduce Human and AI Risk

- Use managed devices for privileged work.
- Enable full-disk encryption and automatic updates.
- Minimize browser extensions and local production secrets.
- Verify domains before entering credentials.
- Reject unexpected MFA prompts.
- Confirm urgent access, invoice, key, and support requests through a second
  channel.
- Do not paste secrets, customer data, private code, or findings into
  unapproved AI tools.
- Treat AI-generated shell commands, infrastructure code, and recovery steps as
  untrusted until reviewed.
- Train staff to report mistakes quickly without hiding them.
- Remove access immediately when a person changes role or leaves.

Our
[Web3 OpSec incident review](https://scauditstudio.com/blog/Web3OpSecHacks)
shows that phishing, fake tools, stolen sessions, and poor secret handling cause
serious losses. Our
[AI and Formal Verification article](https://scauditstudio.com/blog/AI-and-Formal-Verification-In-Defi)
also explains why automation must remain under human control.

- **Before a privileged action:**

```text
- Open the saved official address.
- Check the full domain.
- Confirm the requested action.
- Reject unexpected MFA prompts.
- Verify unusual requests through another channel.
```

## 3. Control Code, AI Changes, and Releases

- Protect the main branch.
- Require pull-request and code-owner review.
- Require tests and security checks before merging.
- Review AI-generated code as carefully as external code.
- Restrict who can change CI workflows, publish packages, push images, and
  deploy production.
- Use short-lived workload identity instead of static cloud keys where
  possible.
- Separate build, deployment, and runtime credentials.
- Pin third-party actions and dependencies.
- Scan source, secrets, dependencies, containers, and infrastructure code.
- Deploy immutable versions.
- Record who approved and started each production deployment.
- Keep a tested rollback and Daml package-upgrade process.

The SCAS article
[Maximizing the Value of Your Smart Contract Audit](https://scauditstudio.com/blog/Maximizing-the-Value-of-Your-Smart-Contract-Audit)
explains why clean scope, clear documentation, and meaningful tests matter
before an audit. Those same practices reduce release risk.

- **Simple branch policy:**

```text
main branch:
- Two approvals required
- Daml and deployment changes require an owner review
- Tests and security scans must pass
- Direct push and force push are disabled
```

## 4. Harden Canton Infrastructure and Recovery

- Follow the current Digital Asset guidance for the exact Canton release.
- Keep participant and validator admin APIs private.
- Restrict inbound and outbound network access.
- Use TLS and strong OIDC settings.
- Use different token audiences for different deployments.
- Do not share identities, databases, volumes, or credentials across
  development, test, and production.
- Back up participant identities, databases, configuration, and identity
  provider settings.
- Define acceptable data loss and downtime.
- Test a full restore in an isolated environment.
- Monitor service health, database health, resource use, ledger lag, login
  failures, and admin changes.
- Patch through a staging process.
- Keep a clear inventory of versions, endpoints, owners, and dependencies.

The
[SCAS Canton Validator Guide](https://scauditstudio.com/blog/CantonValidatorGuide)
covers deployment, identity, backups, monitoring, and common setup failures.
The [Canton audit service](https://scauditstudio.com/solutions/canton-audit)
also reviews participant-node and infrastructure security.

- **Simple environment rule:**

```text
Development, test, and production each have:
- Their own identity
- Their own database
- Their own OIDC client
- Their own secrets
- Their own storage
```

## 5. Detect, Respond, and Learn

- Centralize logs outside the system being monitored.
- Alert on new administrators, MFA resets, permission changes, CI changes,
  package uploads, backup failures, and disabled security controls.
- Keep system clocks synchronized.
- Define who can revoke users, rotate keys, stop deployments, isolate services,
  and contact partners.
- Keep an incident communication channel outside the affected environment.
- Preserve evidence before rebuilding systems.
- Rotate related credentials, not only the first leaked secret.
- Review recent releases, packages, and ledger activity.
- Protect the domain registrar, DNS, certificates, support email, and public
  security contact.
- Run tabletop exercises for source-control, cloud, identity-provider,
  validator, and database compromise.
- Assign every exercise finding an owner and deadline.

The
[SCAS Web3 OpSec incidents article](https://scauditstudio.com/blog/Web3OpSecHacks)
is useful for exercise scenarios, while
[OpSec Best Practices for 2026](https://scauditstudio.com/blog/opsec-best-practices-for-2026)
provides preventive controls.

- **First-hour incident checklist:**

```text
- Assign an incident lead and note-taker.
- Move to a known-clean communication channel.
- Disable compromised access.
- Preserve logs and system snapshots.
- Stop untrusted deployments.
- Identify related credentials and systems.
- Start controlled rotation and recovery.
```

## Final OpSec Checklist

- [ ] Every high-value secret has an owner and purpose.
- [ ] Production keys use protected storage where supported.
- [ ] Production, test, development, and personal credentials are separate.
- [ ] Privileged accounts require hardware-backed MFA.
- [ ] Daily and administrator accounts are separate.
- [ ] Production access is approved, limited, and logged.
- [ ] Operator devices are managed, encrypted, and updated.
- [ ] Staff verify unusual requests through a second channel.
- [ ] Secrets and private data are blocked from unapproved AI tools.
- [ ] AI-generated commands and infrastructure changes receive human review.
- [ ] Important branches and releases are protected.
- [ ] Build, deployment, and runtime credentials are separate.
- [ ] Security scans run before production release.
- [ ] Canton admin APIs are private.
- [ ] Environments do not share identity, storage, databases, or secrets.
- [ ] Backups are encrypted and full recovery is tested.
- [ ] Monitoring covers identity, deployment, package, and admin changes.
- [ ] Incident roles and clean communication channels are documented.
- [ ] Related credentials are rotated after compromise.
- [ ] Tabletop exercise findings are assigned and retested.
