# 60311 sc high double effective stake decrement freezes unstake permanently after validator exit

**Submitted on Nov 21st 2025 at 08:45:14 UTC by @TianYu4n for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60311
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Calling requestDelegationExit records a first effective-stake decrement, util validator status changed to EXITED, USER try to call unstake , and enters the EXITED/PENDING branch and applies a second decrement for the same period, causing an underflow (panic 0x11) before any transfers.

## Vulnerability Details

* First decrement: `requestDelegationExit` calls `_updatePeriodEffectiveStake`for the next period (Stargate.sol #L568).
* Second decrement: `unstake` in the `currentValidatorStatus == EXITED || status == PENDING` branch calls `_updatePeriodEffectiveStake` again Stargate.sol (#L266-#L283), leading to underflow when the checkpointed value is already zero.

## Impact Details

Once triggered, every `unstake` attempt reverts; delegations cannot be changed, so the user’s staked VET remains locked in the staking contract. Funds are not stolen but are permanently frozen.

Affected flow: stake + delegate → requestDelegationExit (first decrement) → validator becomes EXITED → any unstake reverts on second decrement.

## References

add a status judge before call \_updatePeriodEffectiveStake in unstake.

```solidity
bool shouldDecrement = (delegation.status == DelegationStatus.PENDING) ||
    (currentValidatorStatus == VALIDATOR_STATUS_EXITED && delegation.status == DelegationStatus.NONE); // or some other scenery can do decreament
if (shouldDecrement) {
    _updatePeriodEffectiveStake(..., false);
} else {
    // status == EXITED has been decreased in requestDelegationExit, skip
}
```

## Link to Proof of Concept

<https://gist.github.com/2298233831/6f45209c73b685fd619c933c08abe16b>

## Proof of Concept

## Proof of Concept

```typescript
it("poc: double decrement keeps reverting unstake even on repeated attempts", async () => {
        const userAddress = user.address;
        const fmt = (wei: bigint) => ethers.formatEther(wei);

        const balUserStart = await ethers.provider.getBalance(userAddress);
        const balContractStart = await ethers.provider.getBalance(stargateContract);
        console.log(
            `start balances => user: ${fmt(balUserStart)} VET, contract: ${fmt(balContractStart)} VET`
        );

        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        console.log(`tokenId minted: ${tokenId.toString()}`);

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        console.log("delegated to validator");

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        console.log("validator completedPeriods set to 2 (delegation ACTIVE)");

        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        console.log("requested exit (first decrement applied)");

        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();
        console.log("validator forced to EXITED before unstake");

        const balUserPreUnstake = await ethers.provider.getBalance(userAddress);
        const balContractPreUnstake = await ethers.provider.getBalance(stargateContract);
        console.log(
            `before unstake => user: ${fmt(balUserPreUnstake)} VET, contract: ${fmt(balContractPreUnstake)} VET`
        );

        for (let i = 1; i <= 3; i++) {
            try {
                console.log(`unstake attempt ${i}`);
                await stargateContract.connect(user).unstake(tokenId);
                console.log("unexpectedly succeeded");
            } catch (error: any) {
                console.log(`unstake attempt ${i} reverted: ${error.shortMessage || error.message}`);
            }
        }

        const balUserAfter = await ethers.provider.getBalance(userAddress);
        const balContractAfter = await ethers.provider.getBalance(stargateContract);
        console.log(
            `after failed unstakes => user: ${fmt(balUserAfter)} VET, contract: ${fmt(balContractAfter)} VET`
        );
    });
```
