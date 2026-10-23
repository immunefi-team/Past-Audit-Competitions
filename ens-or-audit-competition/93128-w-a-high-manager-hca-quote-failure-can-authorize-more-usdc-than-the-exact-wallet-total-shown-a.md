# 93128 w a high manager hca quote failure can authorize more usdc than the exact wallet total shown at checkout

**Submitted on Sep 14th 2026 at 10:43:29 UTC by @anand0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93128
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

During Manager HCA registration, the amount shown at checkout and the amount later authorized for funding are calculated independently.

If the HCA funding quote fails at checkout, Manager falls back to the registration rent and presents it as an exact `Total`, without the `up to` qualifier. Registration is still allowed to continue, and a second funding-quote failure on the click path is caught rather than blocking the flow.

The registration machine then performs its own HCA budget calculation. If that later calculation succeeds with a larger value, the new amount is used for both the EIP-2612 permit and the generated `transferFrom(wallet, HCA, value)` call.

There is no application-level check requiring the later funding amount to be less than or equal to the exact amount previously shown to the user, and Manager does not return the user to checkout to confirm the increased amount.

The deterministic local PoC reproduces:

```
Manager checkout Total:          $5.00 (no "up to")
registration.start totalPrice:   5,000,000
Recovered HCA budget:            7,000,000
EIP-2612 permit value:           7,000,000
transferFrom amount:             7,000,000
Difference:                      2,000,000 USDC units ($2.00)
```

The additional USDC is transferred to the user's own HCA. I am not claiming theft, attacker receipt, permanent loss of funds, or HCA takeover.

## Affected components

Primary asset:

`apps/manager`

Contributing component:

`packages/transaction-manager`

Audited revision:

`63772fd872af472ced58b009499355f3430c2a86`

The relevant production files in the competition checkout were compared against the imported audited source and were byte-identical.

## Root cause

The registration flow has two independent sources of truth for the wallet funding amount:

1. `TokenPickerContent.tsx` displays `funding?.total ?? pricingData`. If the HCA funding quote is unavailable, the value therefore falls back to the registration rent.
2. `PaymentTotalRow.tsx` adds the `up to` qualifier only when funding data exists. When the quote fails, the rent-only fallback is presented as an exact total.
3. The click-time funding lookup catches a quote failure and continues toward `registration.start`.
4. `registration.machine.ts` independently calls `estimateHcaBudget` after registration begins.
5. The resulting HCA budget determines the wallet funding-permit value.
6. `registration.hca.actors.ts` places that same value into both `permit(...)` and `transferFrom(wallet, HCA, value)`.

No check binds the later authorization amount to the amount previously presented by Manager.

Relevant source locations:

* `apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx:155-173,203-233,434-462,575-585`
* `apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx:4-10,24-40`
* `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts:101-158`
* `packages/transaction-manager/src/machines/registration/registration.machine.ts:604-648,655-738`
* `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:689-721,817-845`

## Impact

This maps to the listed Web/App impact for displaying an incorrect transaction amount that the user then approves.

In the reproduced case, Manager presents `$5.00` as the exact wallet Total, but the same registration flow later constructs an EIP-2612 permit and `transferFrom` for `$7.00`, without returning to Manager for confirmation of the updated amount.

The selected program impact is categorized as Critical. Final severity remains subject to ENS/Immunefi triage, particularly because:

* the wallet may independently display the `$7.00` permit value before signing; and
* the additional funds are transferred to the user's own HCA rather than to an attacker.

## Distinction from QA-03

This does not share the published QA-03 root cause.

QA-03 describes an Explorer display-only discrepancy between per-year pricing and the displayed registration/extension total.

H41 occurs in the Manager HCA funding flow:

```
HCA funding quote fails
→ Manager shows exact rent-only Total
→ registration remains allowed
→ registration machine independently re-quotes
→ larger permit value
→ larger transferFrom amount
```

The issue is therefore an amount-binding failure between Manager checkout and execution, rather than the published Explorer pricing-display discrepancy.

## Limitations

* I have not demonstrated an attacker-controlled method for causing the HCA quote failure.
* The user's wallet may independently expose the larger permit value before signature.
* The additional funds remain in the user's own HCA.
* Testing was local and non-broadcast.
* Quote and wallet boundaries were reproduced using deterministic local fixtures.
* The continuous test exercises the production component → event → UI machine → registration machine → permit → calldata path.
* The Foundry test is an isolated ERC-20 debit semantic control; it is not execution of the complete ENS registration bundle.

## Suggested remediation

The amount confirmed by the user in Manager should be persisted and enforced as a maximum for subsequent funding authorization.

If a later HCA budget calculation exceeds the last confirmed amount, return the user to checkout and require explicit confirmation of the new total before signing the permit.

If an HCA funding quote is unavailable, Manager should not present the rent-only fallback as the exact amount that will leave the wallet.

## Proof of Concept

## Environment

Testing was performed locally and without broadcasting transactions to a public network. No audited production source was modified.

```
Node.js 22
pnpm 10
Vitest 4.1.5
Foundry
Audited revision: 63772fd872af472ced58b009499355f3430c2a86
```

Attached evidence:

* `h41-manager-evidence.png`
* `h41-controls-evidence.png`

The PoC uses evidence-only local test files:

```
apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.h41-bridge.test.tsx

packages/transaction-manager/src/machines/registration/H41-funding-fail-open.test.ts
```

These test files do not modify production code.

## Test 1 — Manager checkout → registration machine → funding calldata

Run:

```bash
cd apps/manager

./node_modules/.bin/vitest run \
  src/features/register-v2/workflow/pricing/components/TokenPickerContent.h41-bridge.test.tsx \
  --reporter=verbose
```

The test exercises the following sequence in one mounted process:

1. Mount the production `TokenPickerContent` with a real `QueryClient`.
2. Run the real `registrationV2UiMachine` and its child `registrationMachine`.
3. Make the checkout HCA budget query fail.
4. Verify that Manager displays exact `Total $5.00`.
5. Verify that the `up to` qualifier is absent.
6. Verify that Register remains enabled.
7. Click the production Register button.
8. Make the click-time budget lookup fail again.
9. Capture the production `registration.start` event.
10. Verify `totalPrice = 5,000,000`.
11. Allow the registration machine's independent budget estimator to recover with `7,000,000`.
12. Verify that the funding-permit actor receives `value = 7,000,000`.
13. Execute the real `submitFundingAndCommitActor`.
14. Decode the generated permit and `transferFrom` calls.
15. Verify that both use `7,000,000`.

Observed result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

Captured values:

```json
{
  "displayedTotalA": "5000000",
  "capturedStartPriceA": "5000000",
  "capturedOwner": "0x1111111111111111111111111111111111111111",
  "capturedHca": "0xaaaa000000000000000000000000000000000001",
  "capturedToken": "USDC",
  "machineBudgetB": "7000000",
  "permitValueB": "7000000",
  "transferFromAmountB": "7000000"
}
```

The continuous bridge was run three consecutive times successfully.

### Causal control

The second test in the same file uses a successful checkout HCA quote of `7,000,000`.

The production component then displays:

```
up to $7.00
```

This control isolates the exact `$5.00` presentation to the quote-failure path.

## Test 2 — transaction-manager funding control

Run:

```bash
cd packages/transaction-manager

./node_modules/.bin/vitest run \
  src/machines/registration/H41-funding-fail-open.test.ts \
  --reporter=dot
```

Observed result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The test independently confirms:

```
checkout amount A:       5,000,000
machine budget B:        7,000,000
permit value:            7,000,000
transferFrom amount:     7,000,000
difference B - A:        2,000,000
```

The generated funding sequence is therefore:

```
USDC.permit(wallet, HCA, 7,000,000, ...)
USDC.transferFrom(wallet, HCA, 7,000,000)
```

while Manager previously presented an exact wallet Total of `5,000,000`.

## Test 3 — ERC-20 debit semantic control

Run:

```bash
forge test --root poc/h41-foundry -vv
```

Observed result:

```
[PASS] testExactDecodedTransferFromDebitsB()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The isolated ERC-20 control confirms:

```
wallet balance before - wallet balance after = 7,000,000
HCA balance after                           = 7,000,000
```

This test demonstrates the debit semantics of the decoded `transferFrom` amount. It is not a public USDC transaction and does not execute the complete ENS registration bundle.

## Additional verification

```
Continuous Manager test:              PASS (2/2)
Focused transaction-manager test:     PASS (1/1)
Local ERC-20 control:                 PASS (1/1)
Manager production build:             PASS
transaction-manager tsc --noEmit:     PASS
Biome check on continuous test:       PASS
git diff --check:                      PASS
```

## Expected vs observed

Expected:

```
later permit/transfer amount <= exact amount confirmed in Manager
```

Observed:

```
exact Manager Total:       5,000,000
later permit/transfer:     7,000,000
difference:                2,000,000
```

The reproduced flow therefore shows that the execution-time HCA funding amount is not bounded by the exact wallet Total presented by Manager before registration proceeds.
