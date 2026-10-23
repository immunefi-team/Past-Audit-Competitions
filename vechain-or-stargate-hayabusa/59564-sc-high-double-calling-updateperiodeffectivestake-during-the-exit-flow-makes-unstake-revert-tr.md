# 59564 sc high double calling updateperiodeffectivestake during the exit flow makes unstake revert trapping staked vet&#x20;

**Submitted on Nov 13th 2025 at 15:30:43 UTC by @niffylord for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59564
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

If a delegator signals exit while their validator is still active and the validator subsequently exits before the delegator calls `unstake`, `Stargate.sol` subtracts the delegator’s effective stake twice. The second subtraction underflows, so `unstake` reverts forever and the user’s VET remains locked.

### Vulnerability Details

* `requestDelegationExit` immediately reduces the validator checkpoint for the exiting token via `_updatePeriodEffectiveStake(..., false)` to ensure future rewards omit the delegator.【F:packages/contracts/contracts/Stargate.sol†L523-L569】
* Later, `unstake` checks the validator status. If it sees `VALIDATOR_STATUS_EXITED`, it runs `_updatePeriodEffectiveStake(..., false)` again, assuming the reduction has not yet happened.【F:packages/contracts/contracts/Stargate.sol†L260-L283】
* Because checkpoints store deltas, the second subtraction reuses the same `(validator, tokenId, period)` entries. When the validator had little remaining stake, the subtraction underflows and Solidity panics (0x11), reverting the entire transaction. No retry can succeed because the same branch keeps triggering.

### Impact Details

{% hint style="danger" %}

* Any delegator whose validator exits between `requestDelegationExit` and `unstake` loses the ability to withdraw the principal VET; `unstake` reverts every time.
* Rewards also remain unclaimable because the contract burns the NFT only after a successful `unstake`.
* Multiple users delegating to the same validator can be griefed as soon as that validator decides to exit, causing a protocol-wide denial of service on capital.
* Chosen impact category: Permanent freezing of funds (Critical) — user-controlled VET becomes unreachable indefinitely.
  {% endhint %}

### Suggested Mitigations

* Track whether `_updatePeriodEffectiveStake` already ran for the exiting delegation (e.g., store a boolean flag per token or delegation) and skip the second decrease in `unstake`.
* Alternatively, recompute the current checkpoint value in `unstake` and only subtract when the stored amount is greater than or equal to the token’s effective stake.
* Refactor the flow so validator status transitions trigger the checkpoint adjustment once, removing duplicate calls from user entry points.

### References

* `Stargate.sol` `requestDelegationExit`: first checkpoint decrease.【F:packages/contracts/contracts/Stargate.sol†L523-L569】
* `Stargate.sol` `unstake`: second checkpoint decrease on exited validators.【F:packages/contracts/contracts/Stargate.sol†L260-L283】
* `_updatePeriodEffectiveStake`: raw subtraction without underflow guard.【F:packages/contracts/contracts/Stargate.sol†L993-L1012】
* Unit test reproducer: `Delegation.test.ts` “should revert unstake after requesting exit if validator exits before unstake.”【F:packages/contracts/test/unit/Stargate/Delegation.test.ts†L205-L231】

## Proof of Concept

{% stepper %}
{% step %}

### Step

Stake an NFT and delegate it to a validator that is currently active.
{% endstep %}

{% step %}

### Step

Advance periods so the delegation is active, then call `requestDelegationExit`.
{% endstep %}

{% step %}

### Step

Advance one more period and mark the validator status as `EXITED`.
{% endstep %}

{% step %}

### Step

Call `unstake`; it attempts the second subtraction and reverts with panic `0x11`.
{% endstep %}
{% endstepper %}

Relevant test from `packages/contracts/test/unit/Stargate/Delegation.test.ts`:

```
205:243:packages/contracts/test/unit/Stargate/Delegation.test.ts
    it("should revert unstake after requesting exit if validator exits before unstake", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();

        // Delegate to an active validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        // Make the delegation active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            2
        );
        await tx.wait();

        // Request exit while validator is still active (first decrease)
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // Advance one more period so the delegation can exit and then force the validator to exit too
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            3
        );
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();

        // Unstake attempts to decrease the effective stake a second time and underflows
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(0x11);
    });
// ... existing code ...
```

## Testing

* Regression covered by `packages/contracts/test/unit/Stargate/Delegation.test.ts`.
* To reproduce: `yarn contracts:test:unit` (fails before fix with panic `0x11`, passes once mitigation is applied).
