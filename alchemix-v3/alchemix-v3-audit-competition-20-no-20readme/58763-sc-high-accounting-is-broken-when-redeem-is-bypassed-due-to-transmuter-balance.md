# 58763 sc high accounting is broken when redeem is bypassed due to transmuter balance

**Submitted on Nov 4th 2025 at 12:17:03 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58763
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When `Transmuter.claimRedemption()` detects that the Transmuter already holds enough MYT to satisfy a matured claim, it skips calling `AlchemistV3.redeem()`. This pays the user from the Transmuter balance but does not reduce `cumulativeEarmarked`, does not increase `_redemptionWeight`, and does not reduce global `totalDebt`. Accounting remains stale, inflating locked collateral and potentially blocking `withdraw()` even though the user’s claim was fulfilled.

## Vulnerability Details

High-level flow:

* Users create positions in `Transmuter` and later call `claimRedemption()`.
* `AlchemistV3.redeem()` is the only path that applies redemption decay/weights, decrements `cumulativeEarmarked`, and reduces global `totalDebt`.
* `Transmuter.claimRedemption()` conditionally calls `redeem()`. If the Transmuter’s MYT balance already covers the matured amount, it bypasses `redeem()` and pays from its own balance.

Key branch in `Transmuter.claimRedemption()` that skips `redeem()` when coverage is available locally:

```solidity
// If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;

if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);
// ... then distribute MYT from Transmuter’s balance and burn synths
```

Consequence of bypass: the Alchemist’s accounting is not updated.

What `AlchemistV3.redeem()` would have done (and is skipped when `amountToRedeem == 0`):

```solidity
_earmark();

uint256 liveEarmarked = cumulativeEarmarked;
if (amount > liveEarmarked) amount = liveEarmarked;

// compute cover and clamp so we never exceed remaining earmarked
uint256 redeemedDebtTotal = amount + coverToApplyDebt;

// Apply decay/weights for redeemed earmarked debt
if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
    _survivalAccumulator = _mulQ128(_survivalAccumulator, ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked);
    _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
}

// Critical accounting updates
cumulativeEarmarked -= redeemedDebtTotal;
totalDebt          -= redeemedDebtTotal;
```

Because `redeem()` is the only place where `cumulativeEarmarked` is reduced and `_redemptionWeight` is increased for redemptions, skipping it leaves the earmarked bucket and weights unchanged while the user is still paid from the Transmuter’s existing balance. `Transmuter` then burns synths and updates the pointer (`setTransmuterTokenBalance()`), but that does not backfill the missing Alchemist-side accounting.

If there is a redemption position and the transmuter doesn't have the required balance and some repayments occur in the same block before the user calls claimRedemption, and these repayments are sufficient to cover the redemption, then the redeem function will not be called. The problem is that all the earmarked debt will not be decreased by the new transmuter balance, even though it is already covered. This breaks the protocol’s accounting.

Downstream effects on withdrawals: `withdraw()` relies on up-to-date `account.debt` and `lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / 1e18`. Since `_redemptionWeight` wasn’t advanced and `cumulativeEarmarked` wasn’t reduced, `_sync()` applies less redemption to user accounts and `account.debt` stays higher than it should. This keeps `lockedCollateral` inflated and can cause `withdraw()` to revert even after a user’s redemption has effectively been fulfilled from Transmuter balance.

As a result, the user receives 100% of the collateral from their redemption, but the protocol remains overcollateralized. For instance if the minimum collateralization ratio is equal to 120%, then 20% of the collateral will remain stuck in the Alchemist contract.

Relevant snippets:

```solidity
// Transmuter pays user from its balance and then burns synths and re-syncs pointer
TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);
TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);
TokenUtils.safeBurn(syntheticToken, amountTransmuted);
alchemist.reduceSyntheticsIssued(amountTransmuted);
alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));
```

```solidity
// withdraw() enforces collateralization using current (stale) debt
uint256 lockedCollateral = convertDebtTokensToYield(_accounts[tokenId].debt) * minimumCollateralization / FIXED_POINT_SCALAR;
_checkArgument(_accounts[tokenId].collateralBalance - lockedCollateral >= amount);
```

```solidity
// _sync() uses _redemptionWeight to unwind redeemed debt; if not advanced, account.debt remains too high
uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
// ... compute redeemedTotal from exposure and earmark deltas ...
account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;
```

Root cause: `Transmuter.claimRedemption()` conditionally bypasses `AlchemistV3.redeem()` when `amountToRedeem == 0`, causing Alchemist’s redemption-side accounting (`cumulativeEarmarked`, `_redemptionWeight`, `totalDebt`) to remain unmodified even though the user’s redemption was served from Transmuter balance.

## Impact Details

* Accounting desync: `cumulativeEarmarked` not reduced; `_redemptionWeight` not increased; `totalDebt` not reduced.
* Inflated `account.debt` after `_sync()`, inflating `lockedCollateral` and potentially reverting `withdraw()`.
* System appears more indebted/earmarked than reality; follow-on flows (e.g., earmark clamping, decay attribution) skewed.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L230-L232>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L592-L603>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L632-L634>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L606-L610>

## Proof of Concept

## Proof of Concept

You can run the coded POC by copy pasting this code in the `IntegrationTest.t.sol` and running `forge test --mt test_claimRedemption_locked_POC`

```solidity
        function test_claimRedemption_locked_POC() external {
        deal(alUSD, address(0xdad), 0);
        deal(alUSD, address(0xdead), 0);
        uint256 amount = 100e18;
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xdad), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xDad = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xDad, ((amount *1e18)/ alchemist.minimumCollateralization()), address(0xdad));

        SafeERC20.safeApprove(address(alUSD), address(transmuterLogic), alchemist.totalSyntheticsIssued());
        transmuterLogic.createRedemption(IERC20(alUSD).balanceOf(address(0xdad)));
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        alchemist.poke(tokenIdFor0xDad);
        uint256 stateBefore = vm.snapshotState();
        SafeERC20.safeTransfer(address(vault), address(transmuterLogic), 91e18);

        transmuterLogic.claimRedemption(1);
        // SafeERC20.safeTransfer(address(vault), address(transmuterLogic), 40e18);
        uint256 cumulativeEarmark_After_Claim_With_Transfer=alchemist.cumulativeEarmarked();
        uint256 totalDebt_After_Claim_With_Transfer=alchemist.totalDebt();
        uint256 totalSyntheticsIssued_After_Claim_With_Transfer=alchemist.totalSyntheticsIssued();
        vm.roll(block.number + 1);
        vm.expectRevert(IllegalArgument.selector);
        alchemist.withdraw(9.1e18, address(0xdad), tokenIdFor0xDad);
        vm.revertTo(stateBefore);
        transmuterLogic.claimRedemption(1);
        uint256 cumulativeEarmark_After_Claim_Without_Transfer=alchemist.cumulativeEarmarked();
        uint256 totalDebt_After_Claim_Without_Transfer=alchemist.totalDebt();
        uint256 totalSyntheticsIssued_After_Claim_Without_Transfer=alchemist.totalSyntheticsIssued();

        assertEq(cumulativeEarmark_After_Claim_Without_Transfer,0);
        assertEq(totalDebt_After_Claim_Without_Transfer,0);
        assertEq(totalSyntheticsIssued_After_Claim_With_Transfer,0);
        assertEq(totalSyntheticsIssued_After_Claim_Without_Transfer,totalSyntheticsIssued_After_Claim_With_Transfer);

        assertEq(cumulativeEarmark_After_Claim_With_Transfer,90000000000000000009);
        assertEq(totalDebt_After_Claim_With_Transfer,90000000000000000009);

        vm.roll(block.number + 1);
        alchemist.withdraw(9.1e18, address(0xdad), tokenIdFor0xDad);
        vm.stopPrank();
                }

```
