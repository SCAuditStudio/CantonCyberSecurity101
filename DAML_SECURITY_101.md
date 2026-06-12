# Daml Security 101

Daml removes many low-level smart contract risks, but it cannot fix a bad
security model. The most serious Daml issues usually involve authority, privacy,
economic rules, or workflows that look correct but fail in edge cases.

## 1. Make Authority and Consent Clear

- List every party and the exact actions that party may take.
- Keep these roles separate:
  - A **signatory** approves the contract and is a stakeholder.
  - An **observer** can see the contract.
  - A **controller** approves a specific choice.
- Take controllers from trusted contract fields, not from caller input.
- Review every choice before asking a party to become a signatory.
- Use a
  [propose-and-accept workflow](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/propose-accept.html)
  when both parties must agree.
- Check who can archive a contract and whether archival removes another party's
  rights.
- Avoid broad delegation such as a generic "act as this party" choice.

The SCAS article on
[common Daml vulnerabilities](https://scauditstudio.com/blog/DamlSmartContractVulnerabilities)
shows how weak signatories, caller-controlled choices, and broad delegation can
turn valid Daml code into an unsafe workflow.

- **Risky example:** the caller chooses who authorizes the withdrawal.

```daml
choice Withdraw : ()
  with
    requestor : Party
    amount : Decimal
  controller requestor
  do
    pure ()
```

- **Safer example:** the contract defines the authorized party.

```daml
choice Withdraw : ContractId Vault
  with
    amount : Decimal
  controller owner
  do
    assertMsg "Invalid amount" (amount > 0.0 && amount <= balance)
    create this with balance = balance - amount
```

## 2. Treat Privacy as a Core Security Rule

- An observer can see the full contract payload, not only selected fields.
- Keep observer lists as small as possible.
- Trace privacy through nested `fetch`, `exercise`, and `create` operations.
- Check who learns data from the full transaction, not only the top-level
  contract.
- Treat prices, balances, counterparties, account IDs, and business terms as
  private unless they are intentionally public.
- Split one large contract into smaller contracts when different parties need
  different data.
- Make sure the web API and logs do not expose more data than the Daml model.
- Test with parties that have partial visibility, not only administrators.

Privacy is especially important on Canton. Our
[Daml vulnerability research](https://scauditstudio.com/blog/DamlSmartContractVulnerabilities)
explains how transaction structure and disclosure can reveal data to a party
that was not meant to receive it. The
[EVM-to-Canton guide](https://scauditstudio.com/blog/EVMtoCantonGuide) also
explains why public-chain privacy assumptions do not transfer cleanly to
Canton.

- **Simple pattern:** separate private trade terms from public status.

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

## 3. Prove the Economic Model

- Write the economic rules before writing the choices.
- Put permanent state rules in `ensure`.
- Validate choice inputs with `assertMsg`.
- Use consuming choices for one-time rights, redemptions, and state changes.
- Reject zero, negative, oversized, duplicate, and self-dealing inputs.
- Define units for every number.
- Define how fees and shares are rounded.
- Multiply before dividing when early division would lose precision.
- Check conservation after minting, burning, splitting, merging, and transfers.
- Test whether repeating or splitting an operation creates extra value.
- Treat contract keys and concurrent submissions as part of the economic model.

Incorrect economic models are a recurring audit issue. Code can compile and
still create value from rounding, allow repeated redemption, or accept invalid
states. Our guides on
[writing correct Daml contracts](https://scauditstudio.com/blog/Writing-Correct-Daml-Contracts-on-Canton)
and
[writing a token contract](https://scauditstudio.com/blog/WritingTokenContractInDAML)
show how to make these rules explicit.

- **Simple example:** protect amount and conservation rules.

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
        assertMsg "Split must be below balance" (splitAmount < amount)
        left <- create this with amount = splitAmount
        right <- create this with amount = amount - splitAmount
        pure (left, right)
```

## 4. Review AI-Generated Code as Untrusted Code

- Do not assume AI understands the business agreement behind a template.
- Give every AI-generated change a named human owner.
- Review every generated `signatory`, `observer`, `controller`, `ensure`, and
  `nonconsuming` line.
- Look for made-up APIs, outdated Daml syntax, and code that only appears
  correct.
- Check whether AI added a caller-controlled party or a broad observer list.
- Check whether AI changed a consuming choice into a repeatable choice.
- Do not paste private contracts, credentials, customer data, or audit findings
  into unapproved AI tools.
- Require tests written from the business rules, not tests that only confirm the
  generated implementation.
- Use AI to support review, never as the final reviewer.

SCAS discusses this problem in
[AI and Formal Verification in DeFi](https://scauditstudio.com/blog/AI-and-Formal-Verification-In-Defi):
automation can help, but it does not prove that the product requirements or
economic model are correct. Our
[correct Daml contracts article](https://scauditstudio.com/blog/Writing-Correct-Daml-Contracts-on-Canton)
also highlights mistakes that compile cleanly but fail at the workflow level.

- **AI review prompt for humans:**

```text
For every changed template:
- Who can create it?
- Who can see it?
- Who can exercise each choice?
- Can any action be repeated?
- Can any value become negative or increase unexpectedly?
- Does the code match the written business rule?
```

## 5. Test the Full Workflow, Not Only the Contract

- Add a negative test for every authorization rule.
- Exercise every sensitive choice as the wrong party.
- Test privacy with non-observers and partially informed parties.
- Test zero, negative, maximum, duplicate, and boundary values.
- Test repeated and concurrent submissions.
- Test exactly at deadline boundaries.
- Test old and new package versions together before an upgrade.
- Keep test packages and test credentials out of production.
- Map application users to ledger parties on the server.
- Never accept `actAs`, `readAs`, or party authority directly from a browser.
- Use stable operation IDs and
  [command deduplication](https://docs.digitalasset.com/build/3.4/sdlc-howtos/applications/develop/command-deduplication.html)
  for retries.
- Confirm ledger completion before sending success notifications.

Our
[Canton audit service](https://scauditstudio.com/solutions/canton-audit)
reviews the Daml model, application boundary, privacy design, and participant
infrastructure together. The
[writing correct Daml contracts](https://scauditstudio.com/blog/Writing-Correct-Daml-Contracts-on-Canton)
article provides more examples of failure cases worth testing.

- **Simple negative test:**

```daml
submitMustFail stranger do
  exerciseCmd vaultCid Withdraw with amount = 10.0
```

## Final Daml Checklist

- [ ] Every party and role is documented.
- [ ] Every signatory is necessary.
- [ ] Every observer needs the full contract payload.
- [ ] Every controller comes from trusted contract state.
- [ ] Every multi-party obligation uses clear consent.
- [ ] No party can remove another party's rights unexpectedly.
- [ ] Sensitive data has been traced through the full transaction.
- [ ] Private and public data are separated where needed.
- [ ] Every permanent state rule is enforced with `ensure`.
- [ ] Every numeric input has clear bounds and units.
- [ ] Rounding and conservation rules are tested.
- [ ] Every one-time action is consuming.
- [ ] Contract keys and concurrent submissions are handled safely.
- [ ] AI-generated code received full human review.
- [ ] Wrong-party and partial-visibility tests exist.
- [ ] Repeated, concurrent, and boundary cases are tested.
- [ ] Application users map to minimum ledger rights.
- [ ] Package upgrades and mixed versions are tested.
- [ ] SCAS research and current Digital Asset documentation were reviewed.
- [ ] Manual security review was completed before production.
