# 57812 sc medium no function to claim aave incentives

**Submitted on Oct 29th 2025 at 01:32:24 UTC by @grearlake for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57812
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Aave provides **Incentives** (e.g., staking rewards or liquidity mining rewards, seeing here: <https://aave.com/docs/primitives/incentives>) to users who supply assets to the protocol. These incentives are typically distributed in the form of additional tokens (e.g., AAVE or other governance tokens) and can be claimed by users who interact with Aave's incentive mechanisms.

In the current implementation of the protocol, there is no functionality to claim these incentives. This is a **missing feature** that could prevent users from accessing the full benefits of supplying assets to Aave.

## Vulnerability Details

In Arbitrum, `AaveV3ARBUSDCStrategy` and `AaveV3ARBWETHStrategy` contract used to supply tokens to Aave:

```
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(pool), amount);
    pool.supply(address(usdc), amount, address(this), 0);
    return amount;
}
```

However, it does not provide a method for users to claim the incentives that Aave distributes to suppliers. In Arbitrum, the aave rewards contract is: <https://arbiscan.io/address/0x929EC64c34a17401F460460D4B9390518E5B473e>

Currently, this contract is still available for rewards claiming.

Since these contracts are **non-upgradeable** and does not include functionality to claim Aave incentives in its initial design, these rewards will **forever be unclaimable**.

## Impact Details

Unable to claim the incentives provided by Aave, resulting in lost reward

## Proof of Concept

## Proof of Concept

Run this test on MYTStrategyTest.t.sol:

```
function test_allocatorOnlyCanAllocateAndDellocate() public {
    vm.prank(address(vault));
    strategy.allocate(abi.encode(0), 100e18, bytes4(0x00000000), address(allocator));

    vm.prank(address(vault));
    strategy.deallocate(abi.encode(100e18), 50e18, bytes4(0x00000000), address(allocator));
}
```
