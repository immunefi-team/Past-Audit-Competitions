# 58522 sc high earmark consumes excess cover inflating cumulativeearmarked

**Submitted on Nov 3rd 2025 at 00:36:13 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58522
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Brief/Intro

`_earmark()` advances the transmuter balance pointer (`lastTransmuterTokenBalance`) to the full current balance every call, regardless of how much of the observed balance delta is actually applied as cover to reduce the earmark amount. This prematurely “consumes” future cover, causing unnecessary growth of `cumulativeEarmarked`, reducing `liveUnearmarked`, and throttling subsequent earmarks even when the transmuter still holds sufficient MYT to offset future maturations.

## Vulnerability Details

Global earmarking computes the matured amount via `Transmuter.queryGraph()` and offsets it by a “cover” derived from the increase in transmuter MYT balance since the last earmark. However, `_earmark()` then updates the pointer `lastTransmuterTokenBalance` to the entire current transmuter balance instead of only accounting for the portion of cover that was actually applied. Any residual, unused cover from the observed delta is effectively discarded for future `_earmark()` calls.

Relevant flow in `AlchemistV3._earmark()`:

```solidity
// delta of transmuter MYT since last earmark
uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance
    ? transmuterCurrentBalance - lastTransmuterTokenBalance
    : 0;

// matured amount over [lastEarmarkBlock+1, block.number]
uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

// cover computed from delta; amount is reduced by cover
uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
amount = amount > coverInDebt ? amount - coverInDebt : 0;

// BUG: pointer advanced to full current balance, even if only part of the delta was actually applied
lastTransmuterTokenBalance = transmuterCurrentBalance;
```

By contrast, `redeem()` advances the pointer only by the cover actually consumed (converted back to yield), which preserves any residual cover for later:

```solidity
// consume the observed cover so it can't be reused, but only the portion actually applied
uint256 usedYield = convertDebtTokensToYield(coverToApplyDebt);
lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;
```

Because `_earmark()` over-advances the pointer, future earmarks do not see the still-available MYT as cover, so `amount` remains higher than it should be, and `cumulativeEarmarked` increases unnecessarily. This reduces `liveUnearmarked = totalDebt - cumulativeEarmarked`, which then clamps future earmarks and distorts weight/survival accounting that depends on these buckets.

Note: `Transmuter.claimRedemption()` eventually calls `setTransmuterTokenBalance()` to resync the pointer to the actual transmuter MYT balance after transfers; however, that does not retroactively restore the prematurely “consumed” cover from prior `_earmark()` calls.

## Impact Details

* Inflates `cumulativeEarmarked`, shrinking `liveUnearmarked = totalDebt - cumulativeEarmarked` and clamping subsequent `_earmark()` amounts.
* Skews weighting/decay attribution that relies on correct earmarked vs unearmarked bucket sizes, impacting fairness of redemption distribution.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1102C1-L1112>

## Proof of Concept

## Proof of Concept

to run the PoC you can copy paste this code in the `IntegrationTest.t.sol` test file and run `forge test --mt test_transmutterBalance_earmark_PoC`

In this PoC, 0xBeef mints 90 alUSD and fully repays his debt. The Transmuter therefore holds 90 yield tokens. Then, 0xDad also mints 90 alUSD and creates a redemption for 45 alUSD. The Transmuter has enough balance, so after the earmark, there is no cumulative earmark — however, the Transmuter’s final balance is set to 90 yield tokens.

After that, 0xDad creates another redemption for 22.5 alUSD. After the earmark, the cumulative earmark is set to 22.5 even though the Transmuter still has enough balance.

```solidity
        function test_transmutterBalance_earmark_PoC() external {
        deal(alUSD, address(0xdad), 0);
        deal(alUSD, address(0xdead), 0);
        uint256 amount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        alchemist.mint(tokenIdFor0xBeef, ((amount *1e18)/ alchemist.minimumCollateralization()), address(0xbeef));
        console.log("balance of beef", IERC20(alUSD).balanceOf(address(0xbeef)));
        vm.roll(block.number + 1);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);

        alchemist.repay(alchemist.convertDebtTokensToYield(amount), tokenIdFor0xBeef);
        vm.stopPrank();
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        // a single position nft would have been minted to 0xdad
        uint256 tokenIdFor0xDad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));

        alchemist.mint(tokenIdFor0xDad, ((amount *1e18)/ alchemist.minimumCollateralization()), address(0xdad));
        console.log("balance of dad: %18e", IERC20(alUSD).balanceOf(address(0xdad)));

        SafeERC20.safeApprove(address(alUSD), address(transmuterLogic), alchemist.totalSyntheticsIssued());
        transmuterLogic.createRedemption(IERC20(alUSD).balanceOf(address(0xdad))/2);
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        alchemist.poke(tokenIdFor0xDad);
        console.log("\n after first createRedemption: \n cumulativeEarmarked after createRedemption: %18e", alchemist.cumulativeEarmarked());
        console.log("lastTransmuterTokenBalance", alchemist.lastTransmuterTokenBalance());

        transmuterLogic.createRedemption(IERC20(alUSD).balanceOf(address(0xdad))/2);
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        alchemist.poke(tokenIdFor0xDad);
        console.log("\n after second createRedemption: \n cumulativeEarmarked after createRedemption: %18e", alchemist.cumulativeEarmarked());
        console.log("lastTransmuterTokenBalance", alchemist.lastTransmuterTokenBalance());
        vm.stopPrank();
        assertGt(alchemist.cumulativeEarmarked(),0);
        }
```
