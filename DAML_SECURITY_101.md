# Daml Security 101

## 1. Start With the Security Model

- List every party in the workflow.
- Define which actions each party may authorize.
- Define which contract data and transaction consequences each party may see.
- Separate the core concepts in the
  [Daml authorization model](https://docs.digitalasset.com/build/3.4/explanations/daml/daml-authorization.html):
  - **Signatory:** authorizes contract creation and is a stakeholder.
  - **Observer:** sees the contract but does not authorize its creation.
  - **Controller:** authorizes a specific choice.
- Draw the party and workflow model before implementing templates.
- Review authorization and
  [Canton ledger privacy](https://docs.digitalasset.com/build/3.4/explanations/canton/canton-ledger-privacy.html)
  together because a workflow can be
  correctly authorized and still expose sensitive data.

### Simple Example

- Write the intended policy before writing the template:

```text
Owner:
- Can create and withdraw from the vault.
- Can see the balance.

Auditor:
- Can see the vault.
- Cannot withdraw or change the owner.

Unrelated party:
- Cannot see or exercise the vault.
```

## 2. Bind Controllers to Trusted Contract Data

- Prefer controllers derived from template fields, following established
  [authorization patterns](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/authorization.html).
- Do not let a caller nominate the party that authorizes a sensitive choice.
- If a flexible controller is required:
  - Check it against an allowlist or trusted role.
  - Bind it to a party already recorded in contract state.
  - Add a clear assertion explaining the policy.
- Test every choice using an unauthorized party.
- Treat contract visibility and exercise authority as different properties.

### Simple Example

- Risky: the caller supplies the controller.

```daml
choice Withdraw : ContractId Vault
  with
    requestor : Party
    amount : Decimal
  controller requestor
  do
    create this with balance = balance - amount
```

- Safer: authority comes from the contract.

```daml
choice Withdraw : ContractId Vault
  with
    amount : Decimal
  controller owner
  do
    assertMsg "Invalid withdrawal" (amount > 0.0 && amount <= balance)
    create this with balance = balance - amount
```

## 3. Make Consent Explicit

- Remember that signatories provide authority to the contract.
- Review every choice before accepting a signatory role.
- Check which party can archive the contract.
- Check whether archival can remove another party's rights.
- Use the
  [propose-and-accept pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/propose-accept.html)
  when multiple parties must consent.
- Require explicit acceptance for:
  - Transfers.
  - Material term changes.
  - New obligations.
  - Contract upgrades.
- Avoid treating an observer role as protection against unilateral archival.

### Simple Example

- Model an offer separately from the accepted agreement:

```daml
template Offer
  with
    seller : Party
    buyer : Party
    amount : Decimal
  where
    signatory seller
    observer buyer

    choice Accept : ContractId Agreement
      controller buyer
      do
        create Agreement with ..
```

## 4. Trace Privacy Through the Entire Transaction

- Keep observer lists minimal under Canton's
  [privacy model](https://docs.digitalasset.com/build/3.4/explanations/canton/canton-ledger-privacy.html).
- Add an observer only when the party needs the complete contract payload.
- Trace nested `fetch`, `exercise`, and `create` operations, including any use
  of
  [explicit contract disclosure](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/explicit-contract-disclosure.html).
- Review who learns data through the complete transaction tree.
- Avoid combining unrelated privacy scopes in one transaction.
- Split contracts when different audiences need different fields.
- Treat pricing, counterparties, balances, and identifiers as sensitive unless
  the business model says otherwise.
- Test workflows using realistic partial visibility.

### Simple Example

- Keep private terms separate from a broadly visible status:

```daml
template TradeTerms
  with
    buyer : Party
    seller : Party
    privatePrice : Decimal
  where
    signatory buyer, seller

template TradeStatus
  with
    operator : Party
    viewers : [Party]
    status : Text
  where
    signatory operator
    observer viewers
```

## 5. Protect State Transitions and Invariants

- Use consuming choices for one-time rights and state replacement.
- Use nonconsuming choices only when repeated execution is intentional.
- Put permanent contract-state
  [constraints](https://docs.digitalasset.com/build/3.4/tutorials/smart-contracts/constraints.html)
  in `ensure`.
- Validate choice inputs with `assert` or `assertMsg`.
- Reject:
  - Zero or negative amounts where not meaningful.
  - Amounts above balances or limits.
  - Self-dealing where roles must differ.
  - Empty or duplicate identifiers.
  - Invalid rates and deadlines.
- Re-check invariants whenever a choice creates replacement contracts.
- Assert conservation after transfers, splits, merges, minting, and burning.

### Simple Example

```daml
template Token
  with
    issuer : Party
    owner : Party
    amount : Decimal
  where
    signatory issuer
    observer owner
    ensure amount > 0.0

    choice Split : (ContractId Token, ContractId Token)
      with splitAmount : Decimal
      controller owner
      do
        assertMsg "Split must be positive" (splitAmount > 0.0)
        assertMsg "Split exceeds balance" (splitAmount < amount)
        left <- create this with amount = splitAmount
        right <- create this with amount = amount - splitAmount
        pure (left, right)
```

## 6. Treat Contract Keys as Scoped Coordination

- Derive
  [contract-key](https://docs.digitalasset.com/build/3.4/reference/daml/contract-keys.html)
  maintainers from the key itself.
- Make every maintainer a signatory.
- Document which parties can resolve each key.
- Do not interpret an invisible lookup result as proof that no contract exists.
- Handle concurrent submissions explicitly.
- Do not assume contract-key uniqueness is a universal global database
  constraint.
- Use a consuming coordination contract when competing creations must be
  serialized.
- Use
  [command deduplication](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/command-deduplication.html)
  and application idempotency for retried submissions.

### Simple Example

- Include the maintaining party in the key:

```daml
template Account
  with
    bank : Party
    accountNumber : Text
    owner : Party
  where
    signatory bank
    observer owner
    key (bank, accountNumber) : (Party, Text)
    maintainer key._1
```

## 7. Bound Time, Delegation, and Upgrades

- Treat
  [ledger time](https://docs.digitalasset.com/build/3.4/explanations/daml/daml-time.html)
  as an input with an allowed tolerance.
- Test exactly at deadline boundaries.
- Avoid placing a high-value decision on a single exact timestamp.
- Scope
  [delegated authority](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/delegation.html)
  by:
  - Allowed action.
  - Allowed target.
  - Expiration time.
  - Maximum delegation depth.
- Avoid generic `ExecuteAs` or unrestricted recursive delegation.
- Require affected parties to accept material
  [smart contract upgrades](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/upgrade/smart-contract-upgrades.html).
- Test stale contract references.
- Test workflows that involve multiple package versions.
- Document the emergency path for disabling or replacing a broken workflow.

### Simple Example

- Bound recursive delegation:

```daml
template DelegatedRight
  with
    owner : Party
    delegate : Party
    remainingDepth : Int
  where
    signatory owner
    observer delegate
    ensure remainingDepth >= 0

    choice SubDelegate : ContractId DelegatedRight
      with newDelegate : Party
      controller delegate
      do
        assertMsg "Delegation depth exceeded" (remainingDepth > 0)
        create this with
          delegate = newDelegate
          remainingDepth = remainingDepth - 1
```

## 8. Test Security Properties

- Add a
  [negative test](https://docs.digitalasset.com/build/3.4/tutorials/smart-contracts/tests.html)
  for every authorization boundary.
- Exercise every choice as the wrong party.
- Test observer and non-observer visibility.
- Test zero, negative, maximum, duplicate, and boundary values.
- Test concurrent and repeated submissions.
- Test consuming and nonconsuming behavior.
- Test conservation of value after every asset operation.
- Test deadline behavior before, at, and after the boundary.
- Keep test-only packages and credentials out of production releases.
- Run [CantonGuard](https://github.com/SCAuditStudio/CantonGuard) as an
  additional review pass.
- Do not treat automated review as a replacement for manual security analysis.

### Simple Example

```daml
unauthorizedWithdrawalMustFail = script do
  owner <- allocateParty "Owner"
  stranger <- allocateParty "Stranger"

  vaultCid <- submit owner do
    createCmd Vault with owner; balance = 100.0

  submitMustFail stranger do
    exerciseCmd vaultCid Withdraw with amount = 10.0
```

## 9. Protect the Application-to-Ledger Trust Boundary

- Treat the application backend as part of the authorization system.
- Do not accept `actAs`, `readAs`, party IDs, or command authority directly from
  an untrusted client.
- Resolve the authenticated application user to approved ledger rights through
  [server-side application authorization](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/secure/authorization.html).
- Give each service the minimum ledger rights required for its function.
- Separate:
  - Human user identities.
  - Machine service identities.
  - Administrative identities.
  - Batch and automation identities.
- Validate the final command payload immediately before submission.
- Bind command IDs to the authenticated actor and business operation.
- Use
  [command deduplication](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/command-deduplication.html)
  for retried operations.
- Reject commands that reference contracts outside the user's authorized
  business scope, even if the ledger would reject them later.
- Record enough metadata to connect an application request to its ledger
  submission and result.

### Simple Example

```text
Authenticated web user
        |
        v
Application authorization policy
        |
        v
Approved ledger party and command type
        |
        v
Ledger API submission
```

- The browser proposes an action.
- The server decides which party may authorize it.
- The ledger independently enforces Daml authorization.

## 10. Review Arithmetic and Economic Integrity

- Daml's
  [`Numeric` type](https://docs.digitalasset.com/build/3.4/reference/daml/stdlib/DA-Numeric.html)
  prevents silent integer overflow, but it does not prevent incorrect
  financial formulas.
- Define units for every numeric field.
- Avoid mixing:
  - Percentages and decimal fractions.
  - Minor and major currency units.
  - Prices and quantities.
  - Gross and net values.
- Multiply before dividing when early division would lose precision.
- Define a rounding policy for fees, interest, shares, and distributions.
- Define who receives rounding remainders.
- Reject division by zero before the calculation.
- Bound rates, leverage, fees, slippage, and quantities.
- Assert conservation where value should not be created or destroyed.
- Test very small values, very large values, and repeated operations.
- Check whether splitting and recombining an asset changes total value.

### Simple Example

- Risky:

```daml
-- Early division can produce the wrong allocation.
share = (userStake / totalStake) * totalReward
```

- Safer:

```daml
assertMsg "Total stake must be positive" (totalStake > 0.0)
share = (userStake * totalReward) / totalStake
```

### Review Questions

- Can a user gain value by splitting one operation into many small operations?
- Can rounding repeatedly favor the same party?
- Can a negative value reverse the intended direction of a transfer?
- Does every mint, burn, split, merge, and settlement preserve its stated
  invariant?

## 11. Design Safe Interfaces and Package Boundaries

- Treat an interface as a security contract between packages.
- Document the authority and visibility assumptions of each interface choice.
- Do not assume every implementation has the same signatories or observers.
- Keep interface choices narrow and business-specific.
- Avoid interfaces that expose generic arbitrary execution.
- Review every package that can implement a trusted interface.
- Pin and review
  [package dependencies](https://docs.digitalasset.com/build/3.4/tutorials/smart-contracts/dependencies.html).
- Do not upload test, experimental, or
  [unreviewed DARs](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/build/how-to-build-dar-files.html)
  to production.
- Confirm that an application resolves the intended package and template
  version.
- Test mixed-version workflows when multiple package versions can coexist.
- Treat package vetting and upload rights as privileged production operations.

### Simple Example

```text
Transferable interface promise:
- A controller may request a transfer.
- The implementation must preserve quantity.
- The receiver must explicitly accept when required.
- The implementation must not disclose private terms to unrelated parties.
```

- The type signature alone does not express all four properties.
- Tests and documentation must enforce the remaining security contract.

## 12. Handle Failure, Exceptions, and Partial Workflows

- Remember the
  [Daml exception model](https://docs.digitalasset.com/build/3.4/tutorials/smart-contracts/exceptions.html):
  a failed transaction is atomic and its ledger effects are not partially
  committed.
- Do not assume external side effects are rolled back with the ledger
  transaction.
- Keep email, payment, webhook, and database effects idempotent.
- Send external notifications after confirming ledger completion.
- Do not expose internal exception details to untrusted clients.
- Distinguish:
  - Business rejection.
  - Authorization failure.
  - Missing visibility.
  - Contention or stale state.
  - Temporary infrastructure failure.
- Retry only errors known to be safe to retry.
- Use stable operation IDs across retries.
- Test workflows interrupted between off-ledger and on-ledger steps.
- Design reconciliation jobs for operations that cross system boundaries.

### Simple Example

```text
Unsafe order:
1. Send "transfer complete" email.
2. Submit ledger transfer.
3. Ledger transfer fails.

Safer order:
1. Create operation ID.
2. Submit ledger transfer with deduplication.
3. Confirm completion.
4. Send idempotent notification for that operation ID.
```

## 13. Plan Contract and Application Migrations

- Inventory active templates before a
  [smart contract upgrade](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/upgrade/smart-contract-upgrades.html).
- Define whether old contracts remain usable, are migrated, or are retired.
- Preserve authorization and privacy properties across versions.
- Require appropriate party consent when migration changes obligations.
- Avoid silently broadening observer sets during migration.
- Verify that keys remain unique and maintainers remain valid.
- Test:
  - Old application with old contracts.
  - New application with old contracts.
  - New application with new contracts.
  - Mixed-version transactions.
- Keep rollback and forward-fix procedures separate.
- Do not delete old package dependencies while active contracts still require
  them.
- Monitor migration progress and unresolved contracts.
- Re-run security tests against the final
  [managed package set](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/manage-daml-packages.html).

### Migration Checklist

```text
- Package reviewed and approved
- Upgrade compatibility checked
- Active-contract inventory captured
- Party consent requirements documented
- Privacy differences reviewed
- Mixed-version tests passed
- Rollback or forward-fix path tested
- Production package hashes recorded
```

## 14. Perform a Structured Manual Review

- Review the system by property, not only file by file, using
  [CantonGuard](https://github.com/SCAuditStudio/CantonGuard) as a supporting
  review pass.
- Build these tables:
  - Party and role matrix.
  - Template signatory and observer matrix.
  - Choice controller matrix.
  - Asset and value-flow matrix.
  - Contract-key and maintainer matrix.
  - External integration matrix.
- Trace every privileged choice from controller to final effects.
- Trace every sensitive field to all parties that can observe it.
- Identify all choices that create contracts using another party's authority.
- Identify all nonconsuming choices and justify repeatability.
- Identify every archive without a replacement.
- Search for unbounded delegation, stale contract IDs, and caller-supplied
  controllers.
- Compare implementation behavior with product requirements and the documented
  [Daml vulnerability patterns](https://github.com/SCAuditStudio/SCASArticles/blob/main/DamlSmartContractVulnerabilities.md).
- Record assumptions that cannot be proven from code.
- Review tests for missing negative and partial-visibility cases.

### Minimal Review Table

| Template | Choice | Controller | Assets affected | New authority | Privacy change |
| --- | --- | --- | --- | --- | --- |
| `Vault` | `Withdraw` | `owner` | Balance decreases | None | Receiver may learn amount |
| `Offer` | `Accept` | `buyer` | Agreement created | Buyer becomes signatory | Seller and buyer see terms |

## Final Daml Checklist

- [ ] Every party and role is documented.
- [ ] Every signatory is necessary.
- [ ] Every observer needs the complete payload.
- [ ] Every controller is bound to trusted state.
- [ ] Every flexible controller is validated.
- [ ] Every one-time action is consuming.
- [ ] Every state invariant is enforced.
- [ ] Every numeric input is bounded.
- [ ] Every key maintainer is derived from the key.
- [ ] Every privacy-sensitive transaction tree is reviewed.
- [ ] Every delegation is narrowly scoped.
- [ ] Every upgrade requires appropriate consent.
- [ ] Every security boundary has a negative test.
- [ ] Application identities map to minimum ledger rights.
- [ ] Command retries use deduplication and idempotency.
- [ ] Arithmetic and rounding invariants are tested.
- [ ] Package and interface trust assumptions are documented.
- [ ] External side effects are reconciled with ledger completion.
- [ ] Mixed-version and migration paths are tested.
- [ ] A party, choice, privacy, key, and asset-flow review was completed.
- [ ] CantonGuard and manual review have been completed.
