# #42623 \[SC-Critical] Potential Loss of Staked Tokens During Unstaking, Incorrect calculation of excess tokens in\`accumulatedDeptRewardsYeet\`

**Submitted on Mar 25th 2025 at 03:37:48 UTC by @KaptenCrtz for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42623
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `accumulatedDeptRewardsYeet` function in the `StakeV2` contract incorrectly calculates excess rewards by subtracting `totalSupply` from the contract's `stakingToken` balance. This calculation does not account for tokens locked during the vesting period, leading to the potential misallocation of staked tokens. If exploited, this could result in the permanent loss of user funds, as these tokens are sent to the zapper and cannot be recovered.

## Vulnerability Details

The vulnerability lies in the following code snippet from the `accumulatedDeptRewardsYeet` function:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

This function is designed to calculate the "excess rewards" in the contract, which are then distributed to the vaults via the `executeRewardDistributionYeet` function. However, the calculation is flawed because it assumes that any tokens in the contract exceeding the totalSupply are excess rewards. This assumption is incorrect in scenarios where tokens are locked during the vesting period. The `totalSupply` variable represents the total amount of tokens currently staked by users. Ideally, any tokens in the contract exceeding this amount should be considered as rewards or excess tokens that can be distributed.\
The issue arises during the locking period when users initiate the unstaking process. During this period:

1. The totalSupply is reduced to reflect the user's unstaked amount.
2. However, the actual tokens remain in the contract until the vesting period ends.\
   If the `accumulatedDeptRewardsYeet` function is called during this locked period, it incorrectly considers the locked tokens as "excess rewards." These tokens are then sent to the zapper via the `executeRewardDistributionYeet` function, leaving no way to recover them for the user. This results in a permanent loss of user funds.\
   For example:
3. A user unstakes 100 tokens, initiating a 10-day vesting period.
4. The totalSupply is reduced by 100, but the tokens remain in the contract.
5. A manager calls executeRewardDistributionYeet, which uses accumulatedDeptRewardsYeet to calculate excess rewards.
6. The 100 locked tokens are sent to the zapper, leaving the user unable to reclaim them after the vesting period.

## Impact Details

The impact of this vulnerability is severe:

1. **Loss of User Funds**: Locked tokens during the vesting period can be permanently lost if misallocated as excess rewards. Unable to claim tokens after locking period ends.
2. **Financial Instability**: If a significant number of tokens are misallocated, it could lead to substantial financial losses for both users and the protocol.

For example, if multiple users initiate unstaking and the manager repeatedly calls `executeRewardDistributionYeet`, a large portion of staked tokens could be lost to the zapper. This could result in significant financial damage and loss of user confidence in the protocol.

## Referrences

[link 1](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149).

[link 2](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L255).

[link 3](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L158)

## Proof of Concept

## Proof of Concept

```javascript
function test_DistribututeUnstakedToken() public {
        address owner = address(this);
        address manager = address(this);
        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        StakeV2 stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));

        token.mint(address(this), 100 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);
        // @POC started unstake of 20 ether 
        stakeV2.startUnstake(20 ether);

        // simulate debt by adding excess token0
        token.transfer(address(stakeV2), 50 ether);
        //zapper
        mockZapper.setReturnValues(1, 1); // does not matter

        stakeV2.depositReward{
                value: 1 ether
            }();
        //@POC even after unstaking no change in the balnceOf contract
        assertEq(100 ether, token.balanceOf(address(stakeV2)));

        stakeV2.executeRewardDistributionYeet(
           //@POC swaping 70 ether instead of 50 still passes
            IZapper.SingleTokenSwap(70 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        assertEq(30 ether, token.balanceOf(address(stakeV2)));
        assertEq(70 ether, token.balanceOf(address(mockZapper)));
    }
```

Paste this above test in `StakeV2.test.sol::StakeV2_HandleExcessDebt`,
