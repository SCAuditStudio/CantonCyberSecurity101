# OpSec 101

## 1. Inventory and Isolate High-Value Secrets

- Build a
  [key-management inventory](https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html)
  covering:
  - Participant and node identities.
  - Signing keys.
  - OIDC client secrets.
  - CI/CD credentials.
  - Cloud credentials.
  - Database credentials.
  - Backup encryption keys.
  - Recovery material.
- Record the owner and purpose of each secret.
- Separate production, test, development, and personal credentials.
- Use managed KMS or HSM-backed custody where supported.
- Apply
  [cryptographic storage controls](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
  to backups.
- Keep recovery copies outside the primary environment.
- Define rotation and revocation procedures.
- Test recovery before an incident.
- Never send secrets through chat, tickets, or unencrypted documents.

### Simple Secret Register

```text
Secret: Production participant signing key
Owner: Infrastructure security
Storage: Managed HSM
Used by: Production participant only
Rotation: Emergency or scheduled policy
Recovery: Two-person approved backup restore
```

## 2. Use Phishing-Resistant Privileged Access

- Require
  [hardware-backed MFA](https://pages.nist.gov/800-63-4/sp800-63b.html)
  for:
  - Source control.
  - Cloud consoles.
  - CI/CD.
  - Identity providers.
  - Password managers.
  - Production administration.
- Use separate daily and privileged accounts.
- Do not share administrator accounts.
- Apply
  [zero-trust access principles](https://csrc.nist.gov/pubs/sp/800/207/final)
  and grant production access for a limited time.
- Require approval for high-risk access.
- Review service accounts and ledger rights regularly.
- Disable dormant accounts.
- Remove access immediately when roles or vendors change.
- Maintain break-glass accounts with additional monitoring.
- Test account recovery without weakening MFA.

### Simple Access Policy

```text
Production access:
- Named user only.
- Hardware security key required.
- Manager or incident-lead approval required.
- Access expires after four hours.
- Commands and administrative events are logged.
```

## 3. Harden Operator Workstations

- Use dedicated devices or profiles for privileged operations.
- Keep the operating system and browser current.
- Enable full-disk encryption.
- Use endpoint protection appropriate for the organization.
- Minimize
  [browser extensions](https://cheatsheetseries.owasp.org/cheatsheets/Browser_Extension_Vulnerabilities_Cheat_Sheet.html).
- Minimize local production secrets.
- Do not use production administration from unmanaged devices.
- Verify domains before entering credentials.
- Verify release hashes and signatures.
- Follow
  [phishing-resistant verification practices](https://www.cisa.gov/secure-our-world/recognize-and-report-phishing)
  for urgent access, invoice, key, and support requests.
- Treat unsolicited wallet, browser extension, and remote support requests as
  hostile.
- Report suspected phishing immediately.

### Simple Verification Habit

```text
Before a privileged login:
- Open the saved official bookmark.
- Check the full domain.
- Use the password manager to detect domain mismatch.
- Reject unexpected MFA prompts.
- Confirm unusual requests using a second communication channel.
```

## 4. Protect Source and Release Integrity

- Use
  [protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
  for important code and deployment paths.
- Require pull-request review.
- Require passing security and test checks.
- Restrict force pushes and branch deletion.
- Require signed commits or signed release artifacts where practical.
- Pin third-party CI actions to immutable revisions.
- Restrict who can:
  - Change CI workflows.
  - Publish packages.
  - Push container images.
  - Create releases.
  - Deploy production.
- Review changes to lockfiles and build scripts carefully.
- Retain signed
  [artifact attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
  and build provenance.
- Verify the artifact being deployed is the reviewed artifact.

### Simple Branch Rules

```text
main branch:
- Two approvals required.
- Code-owner approval required for Daml and deployment files.
- Tests and security scans required.
- Force push disabled.
- Direct push disabled.
```

## 5. Control CI/CD and Production Deployment

- Use
  [OIDC workload identity](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
  instead of static cloud keys where possible.
- Separate build, deployment, and runtime credentials.
- Require approval for production environments.
- Allow production deployment only from protected branches or signed releases.
- Follow a
  [secure CI/CD process](https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html)
  and scan source, dependencies, containers, and secrets before promotion.
- Deploy immutable versions.
- Record who approved and initiated each deployment.
- Keep test packages and credentials out of production artifacts.
- Maintain a tested rollback procedure.
- Maintain a tested Daml package-upgrade procedure.
- Restrict outbound network access from CI runners.
- Treat self-hosted CI runners as production infrastructure.

### Simple Pre-Commit Secret Check

```bash
git diff --cached --name-only |
  grep -E '(^|/)(\.env|.*\.pem|.*\.key)$' && {
    echo "Refusing to commit secret material"
    exit 1
  }
```

## 6. Harden Validator and Participant Operations

- Follow the
  [Digital Asset security guidance](https://docs.digitalasset.com/operate/3.4/howtos/secure/index.html)
  for the exact Canton release being deployed.
- Keep administrative APIs private.
- Restrict inbound and outbound network paths.
- Use TLS for service communication.
- Use strong OIDC configuration.
- Use distinct token audiences per deployment.
- Apply
  [network segmentation](https://cheatsheetseries.owasp.org/cheatsheets/Network_Segmentation_Cheat_Sheet.html)
  and separate development, test, and production:
  - Identities.
  - Credentials.
  - Databases.
  - Volumes.
  - Network access.
- Back up identities and databases according to current operator guidance.
- Protect onboarding and recovery secrets as one-time high-value credentials.
- Monitor:
  - Service availability.
  - Resource pressure.
  - Database health.
  - Ledger processing lag.
  - Authentication failures.
  - Administrative actions.
- Keep an inventory of versions, endpoints, dependencies, and owners.
- Patch through a tested staging process.

### Simple Environment Separation

```text
Development:
- Development identity
- Development database
- Development OIDC client

Test:
- Test identity
- Test database
- Test OIDC client

Production:
- Production identity
- Production database
- Production OIDC client
- No shared credentials or storage with other environments
```

## 7. Prepare for Credential Compromise

- Build an
  [incident response plan](https://www.cisa.gov/resources-tools/resources/incident-response-plan-irp-basics)
  that defines who can:
  - Revoke users.
  - Rotate keys.
  - Disable CI/CD credentials.
  - Stop deployments.
  - Isolate validator services.
  - Contact hosting and network partners.
  - Communicate with users and customers.
- Keep an incident communication channel outside the affected environment.
- Preserve logs and evidence according to
  [incident response recommendations](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
  before rebuilding systems.
- Rotate dependent credentials, not only the first leaked secret.
- Review recently published packages, images, and releases.
- Review ledger and administrative activity during the exposure window.
- Document legal, regulatory, customer, and partner notification duties.
- Run tabletop exercises for:
  - Source-control compromise.
  - Cloud compromise.
  - Identity-provider compromise.
  - CI/CD compromise.
  - Participant or validator compromise.
- Record lessons and update controls after every exercise or incident.

### Simple First-Hour Checklist

```text
- Assign incident lead and note-taker.
- Move coordination to a known-clean channel.
- Disable the compromised identity or token.
- Preserve logs and affected system snapshots.
- Identify dependent credentials and systems.
- Stop untrusted deployments and releases.
- Determine the exposure window.
- Start controlled rotation and recovery.
```

## 8. Build Tested Backup and Disaster-Recovery Procedures

- Define recovery objectives using
  [contingency-planning guidance](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final):
  - Recovery point objective: acceptable data loss.
  - Recovery time objective: acceptable downtime.
- Inventory all state required for recovery.
- Include:
  - Participant and node identities.
  - Databases.
  - Configuration.
  - Package inventory.
  - Identity-provider configuration.
  - DNS and certificate records.
  - Encryption and recovery keys.
- Keep backups in a separate security boundary.
- Encrypt backups with independently protected keys.
- Make at least one backup copy immutable or offline.
- Restrict deletion and retention-policy changes.
- Monitor backup failures.
- Restore into an isolated environment during tests.
- Verify application and ledger behavior after restore.
- Document which data cannot be reconstructed from network peers.
- Run full recovery exercises, not only file-level restore tests.

### Simple Recovery Matrix

| Component | Backup | Recovery owner | Restore tested |
| --- | --- | --- | --- |
| Participant identity | Encrypted identity export | Node operations | Quarterly |
| PostgreSQL | Encrypted snapshot and log backup | Database team | Monthly |
| OIDC configuration | Versioned protected export | Identity team | Quarterly |
| Deployment config | Protected Git repository | Platform team | Every release |

## 9. Monitor for Compromise and Operational Drift

- Establish normal behavior before writing alerts.
- Monitor:
  - New administrator accounts.
  - MFA resets.
  - Party-right changes.
  - OIDC client changes.
  - CI workflow modifications.
  - New package uploads.
  - Unusual API or ledger submission volume.
  - Backup failures.
  - Disabled logging or security controls.
- Follow
  [security logging guidance](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
  and centralize logs outside the system being monitored.
- Protect log integrity and access.
- Synchronize system clocks.
- Include correlation IDs and
  [distributed tracing](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/observe/open-tracing.html)
  across web, backend, and ledger activity.
- Alert on both successful and failed high-risk actions.
- Review alert ownership and escalation paths.
- Track configuration drift from approved baselines.
- Test alerts during exercises.
- Measure detection and response time.

### Simple Detection Rule

```text
Alert: Privilege change outside approved deployment

Trigger:
- Ledger user rights changed
- No matching approved change ticket

Include:
- Actor
- Target user
- Old and new rights
- Source address
- Correlation ID
- Related administrative session
```

## 10. Harden Cloud, Network, Containers, and Databases

- Use separate cloud accounts or projects for production where practical.
- Deny public access by default.
- Keep administrative interfaces behind controlled access paths.
- Restrict security-group and firewall changes.
- Use private networking for databases and internal services.
- Restrict outbound traffic from sensitive workloads.
- Use separate service identities for separate components.
- Apply
  [container hardening](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
  and run containers as non-root.
- Use read-only filesystems where possible.
- Drop unused Linux capabilities.
- Pin container images by digest.
- Scan images and base layers before deployment.
- Apply
  [Kubernetes workload security](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html),
  including network policies.
- Encrypt data in transit and at rest.
- Restrict direct database administration.
- Audit database role and schema changes.

### Simple Network Zones

```text
Public zone:
- Reverse proxy only

Application zone:
- Web and API services
- Outbound access restricted

Ledger zone:
- Participant and validator services
- No direct public access

Data zone:
- Databases and backup agents
- Reachable only from approved services

Administration zone:
- Time-limited operator access
- Strong MFA and session recording
```

## 11. Manage Vulnerabilities and Patches

- Maintain an inventory of:
  - Operating systems.
  - Container images.
  - Canton and Daml versions.
  - Databases.
  - Identity providers.
  - Application dependencies.
- Subscribe to vendor advisories and the
  [CISA Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog).
- Define remediation deadlines by severity and exposure.
- Test patches in staging before production.
- Prioritize internet-exposed and privilege-boundary vulnerabilities.
- Manage dependency exceptions using
  [vulnerability-management guidance](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerable_Dependency_Management_Cheat_Sheet.html),
  with an owner and expiration date.
- Remove unsupported software.
- Rebuild images regularly even when application code has not changed.
- Verify that patched versions are actually deployed.
- Run authenticated infrastructure scans where appropriate.
- Retest controls after emergency changes.
- Keep a rollback plan that does not reintroduce the vulnerability.

### Simple Remediation Policy

```text
Critical, actively exploited:
- Isolate or mitigate immediately
- Patch as an emergency change

Critical, not known exploited:
- Remediate within 72 hours

High:
- Remediate within 14 days

Exception:
- Named owner
- Compensating control
- Expiration date
```

## 12. Control Vendors and Third-Party Integrations

- Apply
  [cyber supply-chain risk management](https://csrc.nist.gov/projects/cyber-supply-chain-risk-management)
  and inventory vendors that can access:
  - Source code.
  - Cloud environments.
  - CI/CD.
  - Production data.
  - Identity systems.
  - Validator or participant infrastructure.
- Grant vendors named, time-limited accounts.
- Require strong MFA.
- Avoid shared support accounts.
- Limit remote support tools.
- Review vendor OAuth applications and API tokens.
- Restrict third-party network paths.
- Define security notification obligations in contracts.
- Require approval before subcontractors receive access.
- Remove access at the end of the engagement.
- Review vendor activity logs.
- Include vendor compromise in incident exercises.
- Maintain a replacement and exit plan for critical providers.

### Vendor Access Record

```text
Vendor: Managed database provider
Access: Database control plane, no application admin
Authentication: Named SSO accounts with hardware MFA
Approval: Infrastructure lead
Expiration: Contract end date
Logging: Provider audit logs exported to company storage
Emergency contact: Recorded in incident runbook
```

## 13. Protect Domains, Email, and Public Communications

- Protect the domain registrar with hardware-backed MFA.
- Use registry lock for critical domains where available.
- Restrict who can change DNS.
- Monitor certificate transparency logs for unexpected certificates.
- Use automated certificate renewal with protected credentials.
- Configure SPF, DKIM, and DMARC for company email.
- Use a restrictive DMARC policy after validation.
- Protect support and security mailboxes.
- Publish an
  [RFC 9116 `security.txt`](https://www.rfc-editor.org/rfc/rfc9116)
  contact and vulnerability disclosure process.
- Reserve common lookalike domains when risk justifies it.
- Monitor impersonation and fake support accounts.
- Use pre-approved communication templates for incidents.
- Verify public incident statements through legal and incident leadership.

### Simple Domain Checklist

```text
- Registrar account uses hardware MFA
- Registry lock enabled
- DNS changes require approval
- Certificate transparency monitored
- SPF configured
- DKIM configured
- DMARC enforcement enabled
- security.txt published
```

## 14. Secure Hiring, Role Changes, and Offboarding

- Perform appropriate screening for privileged roles.
- Use the
  [CIS Critical Security Controls](https://www.cisecurity.org/controls)
  as a baseline and train staff on:
  - Phishing.
  - Secret handling.
  - Production access.
  - Incident reporting.
  - Data classification.
- Use role-based access templates.
- Review access when an employee changes roles.
- Remove access promptly during offboarding.
- Rotate shared or exposed credentials after departures.
- Recover managed devices and hardware keys.
- Transfer ownership of repositories, cloud resources, and vendor accounts.
- Review recent privileged activity for high-risk departures.
- Disable dormant contractor accounts.
- Keep emergency contact details current.
- Run periodic access recertification with system owners.

### Offboarding Checklist

```text
- Disable SSO and email
- Revoke sessions and API tokens
- Remove source-control access
- Remove cloud and CI/CD access
- Remove ledger and party rights
- Recover devices and hardware keys
- Transfer resource ownership
- Rotate affected shared credentials
- Confirm vendor access removal
```

## 15. Exercise the Security Program

- Run
  [incident response exercises](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
  at least annually and after major architecture changes.
- Include technical, business, legal, communications, and leadership roles.
- Exercise realistic scenarios:
  - Compromised source-control maintainer.
  - Malicious Daml package release.
  - Stolen cloud administrator session.
  - Validator identity loss.
  - Database corruption.
  - Identity-provider outage.
  - Public data-exposure allegation.
- Inject incomplete and conflicting information.
- Require decisions under time pressure.
- Record:
  - Detection gaps.
  - Missing access.
  - Unclear ownership.
  - Slow vendor response.
  - Recovery failures.
- Assign every improvement an owner and deadline.
- Re-test material failures.

### Exercise Success Criteria

```text
- Incident lead assigned within 15 minutes
- Clean communication channel established
- Compromised access revoked
- Evidence preserved
- Business impact assessed
- Recovery owner assigned
- Internal and external communications approved
- Follow-up actions tracked to completion
```

## Final OpSec Checklist

- [ ] High-value secrets are inventoried.
- [ ] Production keys use isolated managed storage where supported.
- [ ] Backups are encrypted and recovery-tested.
- [ ] Privileged accounts require phishing-resistant MFA.
- [ ] Daily and privileged accounts are separate.
- [ ] Production access is approved, time-limited, and logged.
- [ ] Operator workstations are managed and hardened.
- [ ] Important branches and releases are protected.
- [ ] CI/CD uses short-lived identity where possible.
- [ ] Build, deploy, and runtime credentials are separate.
- [ ] Environments do not share identities, databases, or credentials.
- [ ] Validator administrative APIs are private.
- [ ] Monitoring covers authentication and administrative events.
- [ ] Incident roles and clean communication channels are documented.
- [ ] Credential-compromise exercises have been completed.
- [ ] Recovery objectives and required backup components are documented.
- [ ] Full recovery has been tested in an isolated environment.
- [ ] Security logs are centralized and protected.
- [ ] Cloud, network, container, and database baselines are enforced.
- [ ] Vulnerability remediation deadlines are defined and measured.
- [ ] Vendor access is named, limited, logged, and removable.
- [ ] Registrar, DNS, email, and public security contacts are protected.
- [ ] Role changes and offboarding trigger immediate access review.
- [ ] Tabletop exercise findings are assigned and retested.
