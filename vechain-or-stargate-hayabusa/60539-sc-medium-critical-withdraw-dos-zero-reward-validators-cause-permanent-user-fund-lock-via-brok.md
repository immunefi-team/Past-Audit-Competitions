# 60539 sc medium critical withdraw dos zero reward validators cause permanent user fund lock via broken reward claim logic

**Submitted on Nov 23rd 2025 at 22:04:29 UTC by @uzemy for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60539
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A flaw in the state-transition logic of the Stargate staking flow causes permanent fund lock when a validator reports zero rewards. Because `claimRewards()` fails to advance the `lastClaimedPeriod` under zero-reward conditions, users cannot clear accumulated period gaps beyond `maxClaimablePeriods`. Once this occurs, all attempts to unstake revert indefinitely, leaving user funds permanently inaccessible. The issue is triggerable under normal protocol conditions and requires no special permissions.

## Vulnerability Details

The issue originates from the reward claiming logic failing to advance `lastClaimedPeriod` when the user accrues zero rewards. In `_claimRewards()`, the function exits early if `claimableAmount == 0`, preventing the normal state update:

```solidity
uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
if (claimableAmount == 0) {
    return; // lastClaimedPeriod not updated
}
$.lastClaimedPeriod[_tokenId] = lastClaimablePeriod;
```

Because the claim state never advances, the user’s unclaimed-period range remains unchanged. Subsequent calls to `delegate()` and `unstake()` enforce the `maxClaimablePeriods` constraint:

```solidity
if (_exceedsMaxClaimablePeriods($, _tokenId)) {
    revert MaxClaimablePeriodsExceeded();
}
```

The `_exceedsMaxClaimablePeriods` computation depends on the span between the first and last claimable periods:

```solidity
 if (lastClaimablePeriod - firstClaimablePeriod >= $.maxClaimablePeriods) {
            return true;
   }
```

If a validator produces enough consecutive zero-reward periods to exceed `maxClaimablePeriods`, the user becomes permanently blocked:

* `_claimRewards()` cannot reduce the gap (because it returns early),
* `_exceedsMaxClaimablePeriods()` continues to revert, and
* `unstake()` becomes permanently inaccessible.

No privileged access is required, normal delegation to a low-performance validator is sufficient to trigger the lock. The result is a permanent denial-of-service on user withdrawals.

## Impact Details

This vulnerability enables a **permanent and irreversible loss of user access to staked VET funds**, triggered under normal protocol conditions without requiring any privileged role, malicious validator, or manipulation of external systems. If a validator produces zero rewards for more than `maxClaimablePeriods`, `_claimRewards()` fails to advance `lastClaimedPeriod` due to an early return, while `unstake()` continues to enforce the period gap invariant. Once this invariant is violated, all future attempts to unstake permanently revert with `MaxClaimablePeriodsExceeded`, even if rewards later resume.

This results in:

* Permanent freezing of funds. (Permanent inability for affected users to withdraw their staked VET, effectively freezing their assets)
* Permanent freezing of unclaimed yield (VTHO rewards)

## Proof of Concept

## Proof of Concept

\#Instructions.

* Place this snippet after the last `it` block in "packages/contracts/test/unit/Stargate/Delegation.test.ts"
* run `yarn contracts:test:unit`

```typescript
    it.only("POC: Critical - Permanent Lock when rewards are 0", async () => {
        const zeroValidator = otherAccounts[4]; //Zero Reward Validator

        // KEY: Pass '0' to explicitly configure the mock to generate zero rewards.
        // This forces the claimableAmount == 0 path in Stargate.sol, triggering the bug.
        tx = await protocolStakerMock.addValidation(zeroValidator.address, 0);
        await tx.wait();

        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();

        tx = await protocolStakerMock.helper__setValidatorStatus(
            zeroValidator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        const MAX_CLAIMABLE = 3; //Reduce maxClaimablePeriods for sake of test
        await stargateContract.connect(deployer).setMaxClaimablePeriods(MAX_CLAIMABLE);

        // Stake and Delegate
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId = await stargateNFTMock.getCurrentTokenId();

        await stargateContract.connect(user).delegate(tokenId, zeroValidator.address);
        log("\n[POC] Staked and Delegated to Zero Reward Validator.");

        // TRIGGER: Create Backlog & Transition State
        // A. Create the huge gap of unclaimed periods (e.g., 10 periods > 3 max).
        await protocolStakerMock.helper__setValidationCompletedPeriods(zeroValidator.address, 10);

        // B. Request Exit (Transitions ACTIVE -> PENDING EXIT)
        // This is necessary to satisfy the internal state machine and bypass 'InvalidDelegationStatus' in unstake.
        await stargateContract.connect(user).requestDelegationExit(tokenId);

        // C. Advance time to finalize the exit (PENDING EXIT -> EXITED)
        await protocolStakerMock.helper__setValidationCompletedPeriods(zeroValidator.address, 12);
        log("[POC] Time advanced. Exit Finalized. Backlog exists.");

        //  The Trap (Initial Unstake Reverts)
        // Unstake must fail because the gap (10 periods) is > Max (3).
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithCustomError(
            stargateContract,
            "MaxClaimablePeriodsExceeded"
        );
        log("[POC] Step 1: Unstake correctly failed initially due to period gap.");

        // ATTEMPT FIX: Call claimRewards
        // This call executes, but the bug in _claimRewards (returns early on 0 rewards)
        // prevents the `lastClaimedPeriod` state variable from updating.
        await stargateContract.connect(user).claimRewards(tokenId);
        log("[POC] Step 2: claimRewards called (attempting to clear backlog).");

        // ASSERT: Permanent Lock
        // If the bug exists, the backlog was not cleared, and unstake fails again.
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithCustomError(
            stargateContract,
            "MaxClaimablePeriodsExceeded"
        );
        log("[POC] Step 3: Unstake failed again. Funds are permanently locked.");
    });
```
