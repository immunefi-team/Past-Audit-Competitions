# 57752 sc medium aave and euler incentives for myt will be lost due to unimplemented claimrewards function

**Submitted on Oct 28th 2025 at 17:15:02 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57752
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The MYT utilizes AAVE as an underlying strategy to generate yield for depositors both on OP and Arbitrum. AAVE offers incentives—such as staking or liquidity mining rewards—to users who supply assets to the protocol. These rewards are usually distributed in the form of additional tokens (e.g., AAVE or other governance tokens) and can be claimed by users participating in Aave’s incentive programs (<https://aave.com/docs/primitives/incentives>). Even though the base `MYTStrategy` defines a function to claim rewards from strategies, this is unimplemented for AAVE which leads to all rewards being lost. Note that this is also an issue in other strategies such as Euler which use an off-chain reward distribution system.

## Vulnerability Details

In all AAVE strategies, MYT allocates assets by supplying them to the v3 instance.

```
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(usdc), address(pool), amount);
        @> pool.supply(address(usdc), amount, address(this), 0);
        return amount;
    }
```

AAVE allows for liquidity mining rewards for depositors to also be claimed on-chain (eg for arbitrum, from the contract here <https://arbiscan.io/address/0x929EC64c34a17401F460460D4B9390518E5B473e>)

All AAVE strategies however have an unimplemented `_claimRewards` function which never allows for claiming, making rewards to be stuck forever.

```
    /// @dev override this function to claim all available rewards from the respective
    /// protocol of this strategy
    function _claimRewards() internal virtual returns (uint256) {}
```

Euler, which uses an off-chain reward distribution system, commonly Merkl also have this function unimplemented which also causes rewards to be lost. When i reached out to the Euler team to confirm, this was their response:

<https://discord.com/channels/742749441697513632/1021868855557116006/1428339375594278993>

```
all the rewards at the moment are distributed either by merkl or by brevis incentra. hence, you need to integrate them directly and there's nothing euler-specific in that process. if you design your contract correctly, yes, you can claim into it. I don't know about incentra but merkl allows some sort of delegation so that you can allow EOA to harvest rewards for your contract and you only need to implement rewards handling (i.e. to be able to recover them from your contract after they're claimed)
```

Since no rewards handling is present in any Euler strategies as well, all rewards are also assumed to be lost.

## Impact Details

All token incentive rewards are lost, which can cause a significant portion of the yield generated for depositors to be stuck in the contract. Furthermore the contract is not upgradeable so all rewards will be permanently lost due to this.

## References

* <https://aave.com/docs/primitives/incentives>
* <https://arbiscan.io/address/0x929EC64c34a17401F460460D4B9390518E5B473e>
* <https://discord.com/channels/742749441697513632/1021868855557116006/1428339375594278993>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol#L38>

## Proof of Concept

## Proof of Concept

1. Add this interface to the top of the file in `AaveV3ARBUSDCStrategy.t.sol`.

```
interface IRewardsController {
    function getAllUserRewards(address[] calldata assets, address user)
        external view returns (address[] memory, uint256[] memory);
}
```

12 Add this test to the end of the file in `AaveV3ARBUSDCStrategy.t.sol`

```
    function test_POC_aave_rewards_permanently_lost() public {
        address AAVE_REWARDS_CONTROLLER = 0x929EC64c34a17401F460460D4B9390518E5B473e;
        address ARB_TOKEN = 0x912CE59144191C1204E64559FE8253a0e49E6548;

        uint256 allocateAmount = 1000e6;
        uint256 rewardAmount = 100e18;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, allocateAmount);

        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, allocateAmount, "", address(vault));

        address[] memory assets = new address[](1);
        assets[0] = AAVE_V3_USDC_ATOKEN;

        address[] memory rewardsList = new address[](1);
        rewardsList[0] = ARB_TOKEN;

        uint256[] memory rewardsAmounts = new uint256[](1);
        rewardsAmounts[0] = rewardAmount;

        vm.mockCall(
            AAVE_REWARDS_CONTROLLER,
            abi.encodeWithSignature("getAllUserRewards(address[],address)", assets, strategy),
            abi.encode(rewardsList, rewardsAmounts)
        );

        (address[] memory returnedRewards, uint256[] memory returnedAmounts) =
            IRewardsController(AAVE_REWARDS_CONTROLLER).getAllUserRewards(assets, strategy);


        console.log("Returned rewards: ", returnedRewards[0]);
        console.log("Returned amounts: ", returnedAmounts[0]);  
        assertEq(returnedRewards[0], ARB_TOKEN);
        assertEq(returnedAmounts[0], rewardAmount);

        console.log("Claiming rewards...");
        uint256 claimedRewards = IMYTStrategy(strategy).claimRewards();
        console.log("Claimed rewards: ", claimedRewards);
        assertEq(claimedRewards, 0);
        if (claimedRewards == 0) console.log("No way to claim rewards, rewards are permanently lost");

        vm.clearMockedCalls();
        vm.stopPrank();
    }
```

3. Run the test

```
forge test --match-test test_POC_aave_rewards_permanently_lost -vv
```

4. Observe the logs. No rewards will be able to be claimed

```
[PASS] test_POC_aave_rewards_permanently_lost() (gas: 469030)
Logs:
  Returned rewards:  0x912CE59144191C1204E64559FE8253a0e49E6548
  Returned amounts:  100000000000000000000
  Claiming rewards...
  Claimed rewards:  0
  No way to claim rewards, rewards are permanently lost
```
