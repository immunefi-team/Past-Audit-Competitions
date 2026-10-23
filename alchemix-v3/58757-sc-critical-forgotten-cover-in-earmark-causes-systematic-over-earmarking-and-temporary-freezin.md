# 58757 sc critical forgotten cover in earmark causes systematic over earmarking and temporary freezing of user collateral

**Submitted on Nov 4th 2025 at 12:06:48 UTC by @algizsec for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58757
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The `_earmark()` function in `AlchemistV3.sol` mishandles how it tracks "cover". When the amount of cover exceeds the required redemptions amount for a given period, the contract advances `lastTransmuterTokenBalance` to the full balance, effectively "forgetting" the unused portion. This inflates `cumulativeEarmarked` and `_earmarkWeight`, leading to systematic over-earmarking of each user’s debt. Subsequently, `redeem()` calls do not clear this inflated earmarking which leads to 2 issues:

1. Users might be denied repaying their debt by burning alAssets (due to artifically lower unearmarked debt). They are either:

* forced to repay using MYT - thus denying them access to their collateral OR
* redeem their alAssets for MYT which will take `timeToTransmute` to mature(20 days set in `Transmuter.t.sol`), which is far more than 24 hours.

2. Liquidations first clear earmarked debt to try and bring the position into healthy state again. The whole earmarked debt is always cleared. When it is inflated due to the bug above, more debt is cleared through `forceRepay` than needed, hence a bigger repayment fee is paid by the user (due to the excess earmarked).

## Vulnerability Details

The root cause is incorrect consumption of observed Transmuter cover inside the `_earmark()` - when `coverInDebt >= amount`, the function sets `lastTransmuterTokenBalance = transmuterCurrentBalance`, effectively advancing the baseline to the full current balance, instead of just by the used portion only. The leftover cover is dropped and cannot be reused to offset future earmarks or redemptions.

```solidity
function _earmark() internal {
       ...

        // Yield the transmuter accumulated since last earmark (cover)
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;

        lastTransmuterTokenBalance = transmuterCurrentBalance;  //@audit setting baseline without considering leftover
       _;
}

```

If `coverInDebt >= amount`, `amount` becomes 0 and the function should consume only `amount` worth of the observed `transmuterDifference` and keep the remaining as future available cover. Instead, the baseline is moved to `transmuterCurrentBalance`, meaning subsequent calls will see `transmuterDifference = 0` until new MYT arrives. The unused portion is therefore "forgotten" from system's perspective.

Because the function treated the whole delta as already consumed, it still performs the earmark bookkeeping (or at least prevents future cover from offsetting earmarks), so `cumulativeEarmarked` and `_earmarkWeight` grow relative to what is economically necessary. This creates divergence between real cover and recorded earmarks.

When `claimRedemption()` is called, it indeed uses the whole Transmuter balance to cover the matured amount and only pulls as much collateral as needed from `AlchemistV3`. This means that `redeem()` will not be called with as much `amount` expected as was marked during `earmark()`, which will leave `cumulativeEarmarked` artificially high.

Later:

1. Users are capped on the amount of `alAssets` they can burn and are forced to `repay` using MYT.

```solidity
function burn(uint256 amount, uint256 recipientId) {
	_;
	
	_checkState((debt = _accounts[recipientId].debt - _accounts[recipientId].earmarked) > 0);

  uint256 credit = amount > debt ? debt : amount;
	
	_;
}
```

2. Liquidated users get their entire inflated `account.earmarked` force repayed which might be an overkill and cost the user a bigger repayment fee.

```solidity
function _liquidate(uint256 accountId) {
	_;
	
	 if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
   }
   
   // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
	  if (account.debt == 0) {
	      feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
	      TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
	      return (repaidAmountInYield, feeInYield, 0);
    }
    
    _;
    
    //if healthy, again, pay repayment fee
    if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
}
```

## Impact Details

Because `cumulativeEarmarked` was inflated, users:

1. Are denied the ability to burn their debt using debt tokens and access their collateral. Instead they need to repay with MYT tokens and user their alAssets to create a redemption in the Transmuter, which will take `timeToTransmute` time to mature (which will be far longer than 24 hours).

The user may want to access the collateral MYT immediately, but won’t be able to unless they swap the alAssets for MYT on the market (and potentially incur losses on a discount).

2. Bear higher repayment fees due to inflated earmark that is cleared during `forceRepay` during liquidation. **Note:** The `protocolFee` paid during `forceRepay` is also higher and will be felt by the user immediately, but it has to be paid regardless at some point when debt is cleared.

## References

1. \[\_earmark()] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1103-L1112>)
2. \[cumulativeEarmarked not cleared] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L605-L613>)
3. \[burn limit] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L470-L472>)
4. \[forceRepay] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L818-L828>)

## Proof of Concept

## Proof of Concept

The following PoC demonstrates how a large repay in the beginning of maturation of a redemption causes artificall inflation of `cumulativeEarmarked` for the other user.

1. Add the following test to `AlchemistV3.t.sol`

```solidity
function test_PoC_Earmark_Ignores_Leftover_Transmuter_Balance() external {
        uint256 depositAmt = 200e18;
        uint256 mintDebt   = 100e18;

        // User 1 deposits 200 tokens and mints 100 debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
        alchemist.deposit(depositAmt, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, mintDebt, address(0xbeef));
        vm.stopPrank();

        // User 2 deposits 200 tokens and mints 100 debt
        vm.startPrank(address(0xabc));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
        alchemist.deposit(depositAmt, address(0xabc), 0);
        uint256 secondTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xabc), address(alchemistNFT));
        alchemist.mint(secondTokenId, mintDebt, address(0xabc));
        vm.stopPrank();

        // User 1 repays all his debt, tokens go to transmuter
        vm.roll(block.number + 1);
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), mintDebt);
        alchemist.repay(mintDebt, tokenId);
        vm.stopPrank();

        // User 3 creates a redemption for 100 tokens
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintDebt);
        transmuterLogic.createRedemption(mintDebt);
        vm.stopPrank();

        // Snapshot before 1st earmark
        uint256 transmuterBal_before1 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        uint256 lastDiff_before1      = alchemist.lastTransmuterTokenBalance();
        uint256 cumulativeEarmarked_before1  = alchemist.cumulativeEarmarked();

        // 1st earmark: 1/10 of redemption matures. Uses transmuterDiff (positive) to cover matured, leaves leftovers inside transmuter
        vm.roll(block.number + 5_256_000 * 3 / 10);
        alchemist.poke(secondTokenId);

        uint256 transmuterBal_after1 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        uint256 lastDiff_after1      = alchemist.lastTransmuterTokenBalance();
        uint256 cumulativeEarmarked_after1  = alchemist.cumulativeEarmarked();

        console.log("");
        console.log("== BEFORE 1st earmark ==");
        console.log("transmuter balance:                        ", transmuterBal_before1);
        console.log("lastTransmuterTokenBalance:                ", lastDiff_before1);
        console.log("cumulativeEarmarked:                       ", cumulativeEarmarked_before1);

        console.log("");
        console.log("== AFTER 1st earmark (30 tokens matured) ==");
        console.log("transmuter balance:                        ", transmuterBal_after1);
        console.log("lastTransmuterTokenBalance:                ", lastDiff_after1);
        console.log("cumulativeEarmarked (coverage is used):    ", cumulativeEarmarked_after1);

        // Advance so more redemption matures; no one else repays (transmuterDiff will be 0 on next earmark)
        // Snapshot before 2nd earmark
        uint256 transmuterBal_before2 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        uint256 lastDiff_before2      = alchemist.lastTransmuterTokenBalance();
        uint256 cumulativeEarmarked_before2  = alchemist.cumulativeEarmarked();

        // 2nd earmark: transmuterDiff == 0; leftover balance is ignored; new amount gets earmarked (locked)
        vm.roll(block.number + 5_256_000 * 7 / 10);
        alchemist.poke(secondTokenId);

        uint256 transmuterBal_after2 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        uint256 lastDiff_after2      = alchemist.lastTransmuterTokenBalance();
        uint256 cumulativeEarmarked_after2  = alchemist.cumulativeEarmarked();

        console.log("");
        console.log("== AFTER 2nd earmark (70 tokens more matured) ==");
        console.log("transmuter balance (unchanged)             ", transmuterBal_after2);
        console.log("lastTransmuterTokenBalance:                ", lastDiff_after2);
        console.log("cumulativeEarmarked (coverage is ignored): ", cumulativeEarmarked_after2);

        // Position snapshot for context
        (uint256 coll, uint256 debt, uint256 earm) = alchemist.getCDP(secondTokenId);
        console.log("");
        console.log("== POSITION of User 2 ==");
        console.log("collateral (shares):                       ", coll);
        console.log("debt (alToken):                            ", debt);
        console.log("earmarked (alToken):                       ", earm);

        // Summary deltas
        console.log("delta earmarked (after 1st earmark):       ", cumulativeEarmarked_after1 - cumulativeEarmarked_before1);
        console.log("delta earmarked (after 2nd earmark):       ", cumulativeEarmarked_after2 - cumulativeEarmarked_before2);


        vm.roll(block.number + 1);
        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();
        uint256 transmuterBal_afterRedeem = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        uint256 lastDiff_afterRedeem      = alchemist.lastTransmuterTokenBalance();
        uint256 cumulativeEarmarked_afterRedeem  = alchemist.cumulativeEarmarked();
        console.log("");
        console.log("== AFTER REDEEM 100 tokens ==");
        console.log("transmuter balance                         ", transmuterBal_afterRedeem);
        console.log("lastTransmuterTokenBalance:                ", lastDiff_afterRedeem);
        console.log("cumulativeEarmarked (not cleared):         ", cumulativeEarmarked_afterRedeem);
        console.log("transmuter total locked for redemption:    ", transmuterLogic.totalLocked());

        (uint256 coll1,uint256  debt1,uint256  earm1) = alchemist.getCDP(tokenId);
        (uint256 coll2,uint256 debt2,uint256 earm2) = alchemist.getCDP(secondTokenId);

        console.log("");
        console.log("== POSITION of User 1 ==");
        console.log("collateral (shares):                       ", coll1);
        console.log("debt (alToken):                            ", debt1);
        console.log("earmarked (alToken):                       ", earm1);

        console.log("");
        console.log("== POSITION of User 2 ==");
        console.log("collateral (shares):                       ", coll2);
        console.log("debt (alToken):                            ", debt2);
        console.log("earmarked (alToken):                       ", earm2);
    }
```

2. Run test with: `forge test --mt test_PoC_Earmark_Ignores_Leftover_Transmuter_Balance -vv`
3. Output:

```bash
  == BEFORE 1st earmark ==
  transmuter balance:                         100000000000000000000
  lastTransmuterTokenBalance:                 0
  cumulativeEarmarked:                        0
  
  == AFTER 1st earmark (30 tokens matured) ==
  transmuter balance:                         100000000000000000000
  lastTransmuterTokenBalance:                 100000000000000000000
  cumulativeEarmarked (coverage is used):     0
  
  == AFTER 2nd earmark (70 tokens more matured) ==
  transmuter balance (unchanged)              100000000000000000000
  lastTransmuterTokenBalance:                 100000000000000000000
  cumulativeEarmarked (coverage is ignored):  70000000000000000000
  
  == POSITION of User 2 ==
  collateral (shares):                        200000000000000000000
  debt (alToken):                             100000000000000000000
  earmarked (alToken):                        70000000000000000000
  delta earmarked (after 1st earmark):        0
  delta earmarked (after 2nd earmark):        70000000000000000000
  
  == AFTER REDEEM 100 tokens ==
  transmuter balance                          0
  lastTransmuterTokenBalance:                 0
  cumulativeEarmarked (not cleared):          70000000000000000000
  transmuter total locked for redemption:     0
  
  == POSITION of User 1 ==
  collateral (shares):                        200000000000000000000
  debt (alToken):                             0
  earmarked (alToken):                        0
  
  == POSITION of User 2 ==
  collateral (shares):                        200000000000000000000
  debt (alToken):                             100000000000000000000
  earmarked (alToken):                        70000000000000000000
```
