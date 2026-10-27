# 60023 sc insight unchecked address 0 validator in unstake&#x20;

**Submitted on Nov 17th 2025 at 16:38:21 UTC by @Oxb4b for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60023
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

## Description

### Brief / Intro

The Stargate contract's `unstake()` function calls `protocolStakerContract.getValidation(delegation.validator)` unconditionally, even when `delegation.validator` is `address(0)` for never-delegated tokens. While the protocol currently handles `address(0)` gracefully by returning `VALIDATOR_STATUS_UNKNOWN`, this pattern violates defensive programming best practices and introduces unnecessary external contract calls.

Recommendation: add a semantic null-check to ensure protocol functions are only called for delegated positions. This strengthens code robustness against future protocol changes and reduces unnecessary gas consumption.

## Vulnerability Details

**File:** Stargate.sol, `unstake()` function (lines 231-321)

### Current Implementation (excerpt)

```solidity
// Line 260-263: Called unconditionally
(, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
    delegation.validator  // Can be address(0) for never-delegated tokens
);

// Line 266-269: Condition checks for EXITED or PENDING
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    // Line 271-273: Another unconditional external call with address(0)
    (, , , uint32 oldCompletedPeriods) = $
        .protocolStakerContract
        .getValidationPeriodDetails(delegation.validator);
    
    // Line 276-282: Updates effective stake
    _updatePeriodEffectiveStake(...);
}
```

### What's Currently Validated

* delegation.status is one of: ACTIVE, PENDING, or NONE
* Token is past maturity period
* Delegation withdrawal handling (if PENDING or EXITED)

### What's NOT Validated

* delegation.validator is not checked against `address(0)` before external call
* No semantic check that validator exists when querying protocol
* No defensive guard against future protocol behavior changes

### Why This Matters

* Semantic correctness: validator status checks should only apply for delegated tokens. For never-delegated tokens, validator address is semantically meaningless.
* Future-proofing: If protocol changes to reject `address(0)`, current code breaks. Defensive check prevents breakage.
* Code clarity: Readers understand that validator checks only apply to delegated positions.
* External call safety: Follows best practice of validating inputs before external interactions.

## Proof Tests

Gist: <https://gist.github.com/tharunbethina/349a58b25be14c80a6b6398d7894a0e3>

{% stepper %}
{% step %}

### Test: unstake an NFT of level 1 if it's mature and not delegated

* Setup and snapshot: retrieve the level spec for levelId = 1, record user and Stargate contract VET balances before staking.
* Stake and mature: call the helper `stakeAndMatureNFT(...)` to stake an NFT and advance blocks until maturity; obtain `tokenId`.
* Verify delegation and unstake: confirm the token’s delegation status is `0` (NONE), then call `unstake(tokenId)` as the user and wait for the transaction. Expect the `TokenBurned` event with `(user, levelId, tokenId, vet amount)`.
* Post-unstake assertions: ensure the token no longer exists (ownerOf reverts with ERC721NonexistentToken) and that both the user and Stargate contract balances are restored to pre-stake values.

This test exists in: Stake.test.ts → it("Should be able to unstake an NFT of level 1 if its mature and not delegated")
{% endstep %}

{% step %}

### Test: protocol behavior for address(0)

* Call `protocolStakerContract.getValidation(address)` using `address(0)` and capture the returned tuple/struct.
* Observed output: Status code `0n` (i.e., `VALIDATOR_STATUS_UNKNOWN`).
  {% endstep %}

{% step %}

### Proof: unstake() on never-delegated token calls getValidation(address(0))

1. Stake an NFT and mature it WITHOUT delegating → `delegation.validator = address(0)`.
2. Call `getValidation(address(0))` directly → returns status code `0n` (UNKNOWN).
3. Call `unstake(tokenId)` → executes successfully, proving contract handles `address(0)` currently.
4. Console output confirms: validator is `address(0)` → passed to `getValidation()` → returns UNKNOWN status.
   {% endstep %}
   {% endstepper %}

## Impact Details

### Security Best Practices

Adding a semantic check that `delegation.validator != address(0)` before calling protocol functions:

* Adds semantic correctness and avoids querying protocol for meaningless addresses.
* Follows "Check Before External Call" pattern and prevents future regressions if protocol tightens validation.
* Avoids potential runtime breaks if protocol begins to revert on invalid addresses.

### Code Optimizations and Enhancements

* Eliminating the unconditional call to `getValidation(address(0))` saves approximately \~4,200 gas per never-delegated token unstake.
* Network-wide impact: gas savings scale with transaction volume; measurable daily reductions accumulate over time.

## References

### Code Locations

* `unstake()` function: Lines 232-321
* `_getDelegationDetails()`: Lines 581-611
* Protocol interface calls: Lines 261-263, 271-273

### Test Coverage

* Existing test confirms never-delegated tokens successfully unstake (no revert): Stake.test.ts → `it("Should be able to unstake an NFT of level 1 if its mature and not delegated")`
* Protocol gracefully handles `address(0)` → returns `UNKNOWN` status

## Proof of Concept / When This Insight is Valuable

<details>

<summary>Scenario 1 — Protocol Evolution</summary>

* Current: Protocol returns `UNKNOWN` for `address(0)` (graceful).
* Future: Protocol v2 adds validation and reverts on `address(0)`.
* Impact: Stargate's `unstake()` breaks for never-delegated tokens.
* Value: Defensive null-check prevents this breaking change.
* Detection Time: Development / testing, not production.

</details>

<details>

<summary>Scenario 2 — Code Maintenance</summary>

* A developer adds new code path in `unstake()` that queries `delegation.validator`.
* If they don't realize `address(0)` is possible, they may introduce logic that incorrectly assumes a nonzero validator, leading to bugs.
* Value: Semantic checks make assumptions explicit and reduce maintenance bugs.

</details>

<details>

<summary>Scenario 3 — Gas Cost Reduction</summary>

* A measurable fraction of staking positions may be never-delegated.
* Each `unstake()` of a never-delegated token currently wastes \~4,200 gas.
* Over time and volume, savings are significant for users and network efficiency.

</details>

## Suggested Fix (conceptual)

* Before calling `protocolStakerContract.getValidation(delegation.validator)` and related protocol functions, check whether `delegation.validator != address(0)`. Only query protocol for non-zero validators.
* Keep existing logic for handling `VALIDATOR_STATUS_UNKNOWN` if you prefer to preserve current behavior, but avoid making the external call when `delegation.validator` is `address(0)`.

{% hint style="info" %}
Do not change any protocol return assumptions in this report — the suggestion is to add a defensive check to future-proof and optimize gas usage. The recommendation preserves current behavior while preventing potential future breakage if the protocol changes to disallow `address(0)`.
{% endhint %}
