# #42152 \[SC-Critical] \`StakeV2::accumulatedDeptRewardsYeet\` fails to account for pending vesting withdrawals which could cause contract insolvency

**Submitted on Mar 21st 2025 at 10:34:08 UTC by @kmm for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42152
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `StakeV2::accumulatedDeptRewardsYeet` function does not account tokens that are currently getting vested, which could cause the manager to user those funds to mint vault shares. Causing insolvency in the redeeming of staking tokens.

## Vulnerability Details

The function `StakeV2::executeRewardDistributionYeet`, is used to convert excess Yeet amount, into vault shares to distribute to the stakeholders.

The amount is validated against `StakeV2::accumulatedDeptRewardsYeet`.

```solidity
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
        uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute"); //@review
```

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        //@audit - `totalSupply` is reduced when creating an unstake escrow, which could cause user funds to be acumulated as debt?
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

When users want to withdraw they must create a request via `StakeV2::startUnstake`.

```solidity
function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount; //@review - Total Supply is deducted

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }
```

Within the function `totalSupply` is deducted but the tokens are not sent anywhere and the vesting for the user starts.

In this situation the manager is completely allowed to use these tokens, and cause insolvency and loss of funds for a lot of users.

## References

## Proof of Concept

## Proof of Concept

1. `totalSupply` is 1000, balanceOf is 1001
2. User creates a unstake vesting for 500 tokens
   * `totalSupply=500`
3. The manager is not aware of this and thinks there are 501 excess tokens
4. The manager reinvests them bringing balanceOf=500, totalSupply=500
5. The user after that claims his vesting, bringing `balanceOf=0`, `totalSupply=500`.
6. Users cannot withdraw due to insufficientBalance
7. The manager can no longer reinvest, due to underflow in `accumulatedDeptRewardsYeet`
