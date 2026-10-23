# #41345 \[SC-Critical] Calculation of accumulatedDeptRewardsYeet is incorrect lead to user lost of fund

**Submitted on Mar 14th 2025 at 06:04:39 UTC by @coffiasd for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41345
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `StakeV2::accumulatedDeptRewardsYeet()` function calculates the accumulated rewards returned by the zapper. Any excess rewards are deposited into the vault, with the minted shares distributed to stakers as rewards. However, pending withdrawals are not properly accounted for. As a result, excess rewards may be deposited into the vault as rewards, preventing users from withdrawing their staked assets even after the 10-day waiting period has passed.

## Vulnerability Details

StakeV2::accumulatedDeptRewardsYeet():

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

`totalSupply` is used to track user's deposit assets , when user In the `startUnstake` function, the `totalSupply` is decreased by the user's `unStakeAmount` because pending assets are not eligible to earn ongoing rewards.\
This approach is reasonable; however, there is no dedicated value to track the pending withdrawal amount.

Add the following test to StakeV2.test.sol:

```solidity

    function test_handleExceedYeetCalculateRewards() public {
        address owner = address(this);
        address manager = address(this);

        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        StakeV2 stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));

        token.mint(address(this), 100 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);

        // simulate debt by adding excess token0
        token.transfer(address(stakeV2), 50 ether);
        //zapper
        mockZapper.setReturnValues(1, 1); // does not matter

        assertEq(100 ether, token.balanceOf(address(stakeV2)));

        //user unstake
        stakeV2.startUnstake(50 ether);

        //manager swap exceed amount.
        uint256 exceedAmount = 51 ether;
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(exceedAmount, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        //duration passed.
        skip(11 days);

        //user withdraw dos.
        stakeV2.unstake(0);
    }
```

```shell
    │   └─ ← [Return] 
    ├─ [2817] StakeV2::unstake(0)
    │   ├─ [897] MockERC20::transfer(StakeV2_HandleExcessDebt: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 50000000000000000000 [5e19])
    │   │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 49000000000000000000 [4.9e19], 50000000000000000000 [5e19])
    │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 49000000000000000000 [4.9e19], 50000000000000000000 [5e19])
    └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 49000000000000000000 [4.9e19], 50000000000000000000 [5e19])

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.72ms (835.08µs CPU time)

Ran 1 test suite in 817.91ms (1.72ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/StakeV2.test.sol:StakeV2_HandleExcessDebt
[FAIL: ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 49000000000000000000 [4.9e19], 50000000000000000000 [5e19])] test_handleExceedYeetCalculateRewards() (gas: 3694042)
```

From above test we can see the unstake revert due to `ERC20InsufficientBalance` error

## Impact Details

* exceed amount can be deposit into vault as reward
* user can't unstake assets

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol?utm\\_source=immunefi#L148-L150>

## Proof of Concept

## Proof of Concept

```solidity
    function test_handleExceedYeetCalculateRewards() public {
        address owner = address(this);
        address manager = address(this);

        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        StakeV2 stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));

        token.mint(address(this), 100 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);

        // simulate debt by adding excess token0
        token.transfer(address(stakeV2), 50 ether);
        //zapper
        mockZapper.setReturnValues(1, 1); // does not matter

        assertEq(100 ether, token.balanceOf(address(stakeV2)));

        //user unstake
        stakeV2.startUnstake(50 ether);

        //manager swap exceed amount.
        uint256 exceedAmount = 51 ether;
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(exceedAmount, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        //duration passed.
        skip(11 days);

        //user withdraw dos.
        stakeV2.unstake(0);
    }
```
