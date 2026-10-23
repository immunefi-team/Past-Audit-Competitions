# 60419 sc high double decrease of effective stake leads to dos and permanent loss of funds

**Submitted on Nov 22nd 2025 at 12:42:38 UTC by @Oxodus for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60419
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Theft of unclaimed yield

## Description

## Brief/Intro

An issue in the `requestDelegationExit()` and `unstake()` functions allows the effective stake of a validator to be decreased twice for the same delegation. When a user with an ACTIVE delegation calls `requestDelegationExit()` and the validator subsequently exits, calling `unstake()` will either cause an underflow revert (DoS) or corrupt the validator's accounting. This results in users being permanently unable to unstake their tokens and retrieve their staked VET, leading to complete loss of funds.

## Vulnerability Details

The vulnerability stems from the fact that both `requestDelegationExit()` and `unstake()` call `_updatePeriodEffectiveStake()` with `_isIncrease = false` under certain conditions, without proper guards to prevent double-decreasing.

In the `requestDelegationExit()` function, when a delegation has ACTIVE status, the function:

1. Calls `signalDelegationExit()` to mark the exit (line 555)
2. then decreases the effective stake

```solidity
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

The `delegationId` mapping is NOT cleared in this case, meaning the token still has a valid delegation record.

In the `unstake()` function, when unstaking a token with EXITED status and an exited validator:

1. Withdraws the delegation
2. Gets the validator status
3. Decreases effective stake again if validator is EXITED

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false // decrease
    );
}
```

The `_updatePeriodEffectiveStake()` function contains the following snippet, which will underflow and revert due to `currentValue < effectiveStake`

```solidity
uint256 updatedValue = _isIncrease
    ? currentValue + effectiveStake
    : currentValue - effectiveStake;
```

## Impact Details

The revert above effectively causes a DoS and locks the user's VET in the contract permanently, additionally the issue also breaks the Rewards calculation mechanism, causing future delegators to this validator receive inflated rewards, with time causing protocol insolvency and delegators losing their yield too

## References

<https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L266-L282> <https://github.com/vechain/stargate-contracts/blob/jose/update-contracts-to-hayabusa/packages/contracts/contracts/Stargate.sol#L547-L568>

## Proof of Concept

## Proof of Concept

* Modify the line `"test:thor-solo"` in `packages/contracts` to `"test:thor-solo": "hardhat test --network vechain_solo --grep \"double decrease\"",` , to allow runnng of only the poc test
* Then paste the following test in the `Delegation.test.ts`

```ts
it("double decrease of effective stake leads to DoS when requestDelegationExit then unstake with exited validator", async () => {
  // Add second validator to keep blockchain running
  const secondValidator = (await ethers.getSigners())[5];

  const validatorStake = ethers.parseEther("25000000"); // 25M VET (minimum required)
  await protocolStakerContract.connect(secondValidator).addValidation(
    secondValidator.address,
    12, // Valid period value
    { value: validatorStake }
  );

  // Fast-forward to activate second validator
  const [period, start] =
    await protocolStakerContract.getValidationPeriodDetails(deployer.address);
  await fastForwardValidatorPeriods(Number(period), Number(start), 0);

  const levelId = 1;
  const levelSpec = await stargateNFTContract.getLevel(levelId);
  const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;

  // Stake NFT
  await stargateContract
    .connect(user)
    .stake(levelId, { value: levelVetAmountRequired });
  const tokenId = await stargateNFTContract.getCurrentTokenId();

  // Mature and delegate
  await mineBlocks(Number(levelSpec.maturityBlocks));
  await stargateContract.connect(user).delegate(tokenId, deployer.address);

  // Activate delegation
  const [periodDuration, startBlock] =
    await protocolStakerContract.getValidationPeriodDetails(deployer.address);
  await fastForwardValidatorPeriods(
    Number(periodDuration),
    Number(startBlock),
    0
  );

  let delegationStatus = await stargateContract.getDelegationStatus(tokenId);
  expect(delegationStatus).to.equal(2n);

  // Get initial effective stake
  const [, , , completedPeriods] =
    await protocolStakerContract.getValidationPeriodDetails(deployer.address);
  const effectiveStakeBefore =
    await stargateContract.getDelegatorsEffectiveStake(
      deployer.address,
      completedPeriods + 2n
    );

  // Request exit (FIRST DECREASE)
  await stargateContract.connect(user).requestDelegationExit(tokenId);

  const effectiveStakeAfterExit =
    await stargateContract.getDelegatorsEffectiveStake(
      deployer.address,
      completedPeriods + 2n
    );
  expect(effectiveStakeAfterExit).to.be.lt(effectiveStakeBefore);

  // Make delegation EXITED
  await fastForwardValidatorPeriods(Number(periodDuration), Number(startBlock));
  delegationStatus = await stargateContract.getDelegationStatus(tokenId);
  expect(delegationStatus).to.equal(3n);

  // Exit first validator (second validator keeps blockchain running)
  await protocolStakerContract.connect(deployer).signalExit(deployer.address);
  await fastForwardValidatorPeriods(Number(periodDuration), Number(startBlock));

  const [, , , , validatorStatus] = await protocolStakerContract.getValidation(
    deployer.address
  );
  expect(validatorStatus).to.equal(3);

  // Attempt unstake (SECOND DECREASE - should revert)
  await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;
});
```

The above Poc gives the following output

```
@repo/contracts:test:thor-solo:
@repo/contracts:test:thor-solo:
@repo/contracts:test:thor-solo:   shard-i2: Stargate: Delegation
@repo/contracts:test:thor-solo:     ✔ double decrease of effective stake leads to DoS when requestDelegationExit then unstake with exited validator (7409ms)
@repo/contracts:test:thor-solo:
@repo/contracts:test:thor-solo:
@repo/contracts:test:thor-solo:   1 passing (10s)
@repo/contracts:test:thor-solo:
```
