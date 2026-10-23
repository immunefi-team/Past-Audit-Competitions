# 60597 sc low hasrequestedexit returns true for not just requested exits but also delegations that are already exited

**Submitted on Nov 24th 2025 at 09:01:17 UTC by @Brainiac5 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60597
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Description:**\
The `hasRequestedExit` function is intended to indicate whether a user has requested a delegation exit. However, its current implementation only checks if `endPeriod != type(uint32).max`, which is also true for delegations that have already exited. This can lead to false positives, where the function returns `true` even if the delegation is no longer active and has already exited.

The current implementation:

```solidity
function hasRequestedExit(uint256 _tokenId) external view returns (bool) {
    StargateStorage storage $ = _getStargateStorage();

    // get end period of the delegation
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];

    // If no delegation exists, exit was never requested
    if (delegationId == 0) {
        return false;
    }

    // Fetch only the period details (single external call)
    (, uint32 endPeriod) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

    // endPeriod is set to type(uint32).max when delegation is created
    // It changes to a specific period number when exit is requested
    return endPeriod != type(uint32).max;
}
```

**Impact:**

* Users and dApps may incorrectly interpret exited delegations as having a pending exit request.
* Can cause UI confusion, incorrect protocol logic, or misreporting of delegation status.

**Suggested Fix:**\
Update the function to check that the `endPeriod` is not only set, but also in the future relative to the validator's completed periods. This ensures that only active exit requests are reported as true.

```solidity
function hasRequestedExit(uint256 _tokenId) external view returns (bool) {
    StargateStorage storage $ = _getStargateStorage();

    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) {
        return false;
    }

    (, uint32 endPeriod) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
    (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);
    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(validator);

    // Only return true if endPeriod is set and is in the future (exit requested, not yet completed)
    return endPeriod != type(uint32).max && endPeriod > completedPeriods + 1;
}
```

## Proof of Concept

## Proof of Concept

> Add the test below to `Stake.test.ts`

```
    it.only("should show hasRequestedExit returns true for already exited delegation (endPeriod <= completedPeriod + 1)", async () => {
        // Stake and delegate NFT
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId = await stargateNFTMockContract.getCurrentTokenId();
        await stargateContract.connect(user).delegate(tokenId, validator.address);

        // Set validator to ACTIVE and advance periods
        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 10);

        // Request delegation exit
        await stargateContract.connect(user).requestDelegationExit(tokenId);

        // Advance periods so delegation is exited
        await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 20);

        // Now delegation is exited, and endPeriod is set
        // hasRequestedExit will return true, and status is EXITED
        const hasRequestedExit = await stargateContract.hasRequestedExit(tokenId);
        const status = await stargateContract.getDelegationStatus(tokenId);

        expect(hasRequestedExit).to.be.true;
        expect(status).to.equal(DELEGATION_STATUS_EXITED);
    });
```
