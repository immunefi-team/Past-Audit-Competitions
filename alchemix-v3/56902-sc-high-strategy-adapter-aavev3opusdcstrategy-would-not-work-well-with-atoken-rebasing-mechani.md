# 56902 sc high strategy adapter aavev3opusdcstrategy would not work well with atoken rebasing mechanism

**Submitted on Oct 21st 2025 at 16:47:14 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56902
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/AaveV3OPUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

Strategy implementation of AAVE does not handle the growing/rebase amount of aToken (example: aUSDC). the `allocate` and `deallocate` is done in term of USDC and does not consider that the aToken in `AaveV3OPUSDCStrategy` and other AAVE adapter implementation is rebasing. thus making the rebasing amount stuck inside the strategy.

## Vulnerability Details

Morpho VaultV2 have the allocate and deallocate that later would be calling `MYTStrategy` .

Function `allocate` would supply the amount provided into AAVE pool, thus receiving aToken.

For `deallocate`, the strategy would withdraw from AAVE pool, burning the aToken and receive the asset back.

The issue lies in how `deallocate` can only provide specific amount because in the Morpho VaultV2 `deallocateInternal` would check the return value of `deallocate` from adapter and deduct it into its internal cap allocation:

```solidity
    function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

@>      (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
@>          _caps.allocation = (int256(_caps.allocation) + change).toUint256();
        }

        SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
        emit EventsLib.Deallocate(msg.sender, adapter, assets, ids, change);
        return ids;
    }

```

so in this scenario, it is impossible to withdraw more amount than what is used at the `allocation` call.

consider a scenario where the strategy would deposit 100 USDC into the pool:

1. MYT VaultV2 calling `allocate` would supply 100 USDC into the pool, in return the strategy receive 100 aUSDC
2. after some time 100 aUSDC became 105 aUSDC from rebase
3. now the MYT VaultV2 can only `deallocate` 100 USDC from the adapter which as shown below would be used to withdraw from the pool 100 USDC amount which in turn would burn same amount of aToken and receive the underlying.

`AaveV3OPUSDCStrategy` :

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        // withdraw exact underlying amount back to this adapter
        pool.withdraw(address(usdc), amount, address(this));
        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
        return amount;
    }
```

1. this however leaving the Strategy with 5 aUSDC that stuck inside the contract with no way to be used/withdraw.
2. if for example, the VaultV2 try to withdraw all the aUSDC, it would throw underflow because the allocation check in `VaultV2::deallocateInternal`

## Impact Details

Loss of AAVE yield of aToken from supplying into the pool.

this is a crucial issue because of how the AlchemistV3 and Transmuter relied on yield generation of strategy to work. if this issue is not fixed, there are potency of protocol breaking.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/AaveV3OPUSDCStrategy.sol#L38-L51>

<https://github.com/morpho-org/vault-v2/blob/7c04afc9817e8c1edf548367007cdbdfd9fb2866/src/VaultV2.sol#L604-L610>

## Proof of Concept

## Proof of Concept

there are two scenario here:

1. where the vault only withdraw the allocated amount and showing that the remaining rebased aToken is left in the contract.
2. vault try to withdraw all the aToken, which would underflow.

after configuring the .env, apply diff to `src/test/strategies/AaveV3OPUSDCStrategy.t.sol`

```diff
diff --git a/src/test/strategies/AaveV3OPUSDCStrategy.t.sol b/src/test/strategies/AaveV3OPUSDCStrategy.t.sol
index 3a6e95e..c892467 100644
--- a/src/test/strategies/AaveV3OPUSDCStrategy.t.sol
+++ b/src/test/strategies/AaveV3OPUSDCStrategy.t.sol
@@ -60,4 +60,50 @@ contract AaveV3OPUSDCStrategyTest is BaseStrategyTest {
         IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
     }
+
+    function test_strategy_deallocate_rebaseToken() public {
+        // Because aUSDC and USDC is 1:1, allocate and deallocate would be same amount in this PoC
+        uint256 amountToAllocate = 100e6;
+        uint256 amountToDeallocate = amountToAllocate;
+
+        // allocate
+        vm.startPrank(vault);
+        deal(USDC, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
+        require(initialRealAssets > 0, "Initial real assets is 0");
+        console.log(initialRealAssets); // 99 999 999
+
+        // warp to rebase the aUSDC held. 99.99 aUSDC -> 105 aUSDC
+        vm.warp(block.timestamp + 365 days);
+        uint256 realAssetsNow = IMYTStrategy(strategy).realAssets();
+        require(realAssetsNow > initialRealAssets);
+        console.log(realAssetsNow); // 105 003 249
+
+        // snapshot before branching
+        uint256 id = vm.snapshot();
+
+        // if we deallocate the original 100e6 amount from amountToDeallocate
+        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
+        (bytes32[] memory strategyIds, int256 change) = IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
+        assertApproxEqAbs(change, -int256(amountToDeallocate), 1 * 10 ** testConfig.decimals);
+        assertGt(strategyIds.length, 0, "strategyIds is empty");
+        assertEq(strategyIds[0], IMYTStrategy(strategy).adapterId(), "adapter id not in strategyIds");
+        uint256 finalRealAssets = IMYTStrategy(strategy).realAssets();
+        require(finalRealAssets < initialRealAssets, "Final real assets is not less than initial real assets");
+        // there are stuck balance inside Strategy which cant be withdrawed
+        assert(finalRealAssets != 0);
+        console.log("1st branch: deallocate the original 100e6 succeed");
+        console.log("stuck aUSDC", finalRealAssets);
+
+        vm.revertTo(id);
+        // if we try to deallocate the max amount possible, it would overflow
+        uint256 maxAmount = IMYTStrategy(strategy).previewAdjustedWithdraw(realAssetsNow);
+        prevAllocationAmount2 = abi.encode(amountToAllocate);
+        vm.expectRevert();
+        (strategyIds, change) = IMYTStrategy(strategy).deallocate(prevAllocationAmount2, maxAmount, "", address(vault));
+        console.log("2nd branch: the deallocate the max amount possible from realAssetsNow would revert of overflow");
+    }
+
 }

```

`forge test --mt test_strategy_deallocate_rebaseToken`

```bash
[PASS] test_strategy_deallocate_rebaseToken() (gas: 695749)
Logs:
  99999999
  105003249
  1st branch: deallocate the original 100e6 succeed
  stuck aUSDC 5003248
  2nd branch: the deallocate the max amount possible from realAssetsNow would revert of underflow

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 228.82ms (2.17ms CPU time)
```

the rebase aToken would stuck inside the strategy with no method to withdraw it. if the vault try to `deallocate` more amount to account the rebased amount, it would revert.
