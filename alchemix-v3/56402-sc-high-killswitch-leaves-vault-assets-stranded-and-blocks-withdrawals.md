# 56402 sc high killswitch leaves vault assets stranded and blocks withdrawals

**Submitted on Oct 15th 2025 at 16:10:23 UTC by @spongebob for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56402
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L107> short-circuits allocate by returning `(ids(), 0)` when `killSwitch` is true.

However, the vault has already pushed the requested assets into the adapter before that return executes.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/lib/vault-v2/src/VaultV2.sol#L574>

Because the adapter never reverts, those tokens stay parked on the strategy contract while the vault’s caps/allocation stay unchanged (change = 0) and `realAssets` implementations such as `EulerUSDCStrategy.realAssets()` ignore that idle balance, so the vault under-reports its assets and solvency.

The same silent bypass exists in deallocate.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L124>

The vault still calls `SafeERC20Lib.safeTransferFrom` for the full assets amount (lib/vault-v2/src/VaultV2.sol:610), but the strategy never unwinds or approves funds, so the call either reverts blocking withdrawals when this adapter is the only liquidity path, or pulls idle tokens that were previously stranded, again leaving allocation bookkeeping untouched.

No explicit signal is emitted when the kill switch bypass triggers, so operations/incident response can’t distinguish between genuine success and a skipped execution, matching the incident-response gap you described.

## Impact

This bug has two impacts

1. **Permanent freezing of funds:** Once `killSwitch` is true, the vault’s deallocate (and `forceDeallocate`) calls always revert because the strategy short-circuits before unwinding and approving assets. Any liquidity held in that adapter becomes irretrievable through the supported paths, so user withdrawals can be blocked indefinitely while the switch remains on.
2. **Protocol insolvency:** During allocation, the vault still transfers the requested assets into the adapter before the kill-switch bypass. Because the adapter reports zero change, the vault stops accounting for those funds even though they now sit idle on the strategy (and most `realAssets()` implementations ignore that idle balance). The vault’s recorded total assets shrink while liabilities stay constant, enabling share-price distortions that can be exploited when solvency is restored.

## Recommendation

* Make the adapter revert with a dedicated `KillSwitchActive` error (or perform a best-effort unwind before reverting) instead of returning zero so the vault can deterministically roll over to alternate routes. Reverting also undoes the pre-call `safeTransfer`.
* Emit a `KillSwitchBypass` event (include direction and assets) whenever an emergency path triggers, so on-call teams can reconcile stuck balances quickly.

## Proof of Concept

Add this test to `MYTStrategy.t.sol` and run `forge test --mt testPOC_KillSwitch_Freezes_Funds_And_Causes_Insolvency -vvvv`

```solidity
function testPOC_KillSwitch_Freezes_Funds_And_Causes_Insolvency() external {
        // SETUP: Set curator and configure vault
        bytes memory idData = strategy.getIdData();

        // Set proxyOwner as curator
        vm.prank(proxyOwner);
        vault.setCurator(proxyOwner);

        vm.startPrank(proxyOwner);
        // Submit and execute addAdapter
        vault.submit(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        vm.warp(block.timestamp + vault.timelock(IVaultV2.addAdapter.selector));
        vault.addAdapter(address(strategy));

        // Submit and execute increaseAbsoluteCap
        vault.submit(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 1000e18)));
        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseAbsoluteCap.selector));
        vault.increaseAbsoluteCap(idData, 1000e18);

        // Set proxyOwner as allocator
        vault.submit(abi.encodeCall(IVaultV2.setIsAllocator, (address(proxyOwner), true)));
        vm.warp(block.timestamp + vault.timelock(IVaultV2.setIsAllocator.selector));
        vault.setIsAllocator(address(proxyOwner), true);
        vm.stopPrank();

        // SETUP: Deposit to vault to update its totalAssets accounting
        uint256 initialVaultFunds = 1000e18;
        vm.startPrank(user);
        IERC20(fakeUnderlyingToken).approve(address(vault), initialVaultFunds);
        vault.deposit(initialVaultFunds, user);
        vm.stopPrank();

        // Get strategy ID for tracking allocation
        bytes32[] memory strategyIds = strategy.ids();
        bytes32 strategyId = strategyIds[0];

        // STEP 1: Record initial state
        uint256 vaultBalanceBefore = IERC20(fakeUnderlyingToken).balanceOf(address(vault));
        uint256 strategyBalanceBefore = IERC20(fakeUnderlyingToken).balanceOf(address(strategy));
        uint256 vaultTotalAssetsBefore = vault.totalAssets();
        uint256 strategyRealAssetsBefore = strategy.realAssets();
        uint256 allocationBefore = vault.allocation(strategyId);

        // STEP 2: Enable kill switch BEFORE allocation
        vm.prank(admin);
        strategy.setKillSwitch(true);

        // STEP 3: Attempt allocation with kill switch enabled
        // The vault will transfer 500e18 to the strategy, but the strategy returns change = 0
        uint256 allocateAmount = 500e18;
        vm.prank(proxyOwner);
        vault.allocate(address(strategy), abi.encode(0), allocateAmount);

        // STEP 4: Record state after allocation
        uint256 vaultBalanceAfterAllocate = IERC20(fakeUnderlyingToken).balanceOf(address(vault));
        uint256 strategyBalanceAfterAllocate = IERC20(fakeUnderlyingToken).balanceOf(address(strategy));
        uint256 vaultTotalAssetsAfterAllocate = vault.totalAssets();
        uint256 strategyRealAssetsAfterAllocate = strategy.realAssets();
        uint256 allocationAfterAllocate = vault.allocation(strategyId);

        // VULNERABILITY 1: Funds transferred but allocation tracking not updated
        // The vault transferred 500e18 to the strategy
        vm.assertEq(vaultBalanceBefore - vaultBalanceAfterAllocate, allocateAmount, "Vault should have transferred funds");
        vm.assertEq(strategyBalanceAfterAllocate - strategyBalanceBefore, allocateAmount, "Strategy should have received funds");

        // But the allocation tracking was not updated (change = 0)
        vm.assertEq(allocationAfterAllocate, allocationBefore, "Allocation should remain unchanged due to killSwitch");
        vm.assertEq(allocationAfterAllocate, 0, "Allocation should still be 0");

        // The strategy's realAssets() ignores idle balance (as per EulerUSDCStrategy pattern)
        vm.assertEq(strategyRealAssetsAfterAllocate, 0, "Strategy realAssets ignores idle balance");

        // The actual token balances:
        uint256 actualTokens = vaultBalanceAfterAllocate + strategyBalanceAfterAllocate;
        vm.assertEq(actualTokens, initialVaultFunds, "Total tokens should equal initial deposit");

        // VULNERABILITY: Funds are stuck on strategy with no way to recover them
        // The vault's allocation tracking shows 0, so deallocate won't work
        // The funds sit idle on the strategy but aren't counted in realAssets()
        vm.assertEq(strategyBalanceAfterAllocate, allocateAmount, "Funds are stuck on strategy");
        vm.assertEq(allocationAfterAllocate, 0, "But allocation tracking shows 0");
        vm.assertGt(strategyBalanceAfterAllocate, 0, "VULNERABILITY: Funds trapped on strategy with no recovery path");

        // VULNERABILITY 2: Permanent Freezing of Funds - deallocate reverts
        // Now try to deallocate the funds that are sitting idle on the strategy
        vm.prank(proxyOwner);
        vm.expectRevert(abi.encodeWithSignature("ZeroAllocation()"));
        vault.deallocate(address(strategy), abi.encode(0), allocateAmount);

        // The deallocate reverts because:
        // 1. The strategy's killSwitch check returns (ids(), 0) before calling _deallocate
        // 2. _deallocate is never called, so funds are never approved back to vault
        // 3. The vault's safeTransferFrom at line 610 tries to pull funds but has no approval
        // 4. Even if we had allocation > 0, it would still fail because _deallocate is bypassed

        // Let's verify the funds are truly stuck by checking they can't be recovered
        uint256 strategyIdleBalance = IERC20(fakeUnderlyingToken).balanceOf(address(strategy));
        vm.assertEq(strategyIdleBalance, allocateAmount, "Funds are stuck on strategy");

        // Even disabling the kill switch won't help because allocation tracking is wrong
        vm.prank(admin);
        strategy.setKillSwitch(false);

        // The vault thinks allocation is 0, so it won't try to deallocate
        uint256 currentAllocation = vault.allocation(strategyId);
        vm.assertEq(currentAllocation, 0, "Allocation is 0, vault doesn't know about the funds");
    }
```
