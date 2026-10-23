# 58635 sc high cumulativeearmarked is not subtracted in forcerepay&#x20;

**Submitted on Nov 3rd 2025 at 18:10:51 UTC by @Tarnishedx0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58635
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`cumulativeEarmarked` is not subtracted in `_forceRepay()`. When `Transmuter` redeems based on inflated earmarks, protocol transfers non-existent `collateral` which leads to insolvency.

## Vulnerability Details

In the `_forceRepay()` function called during liquidation, `account.earmarked` is subtracted by:

```solidity
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

...

```

But the `cumulativeEarmarked` is not subtracted like in the `repay()` function:

```solidity
#repay()

        uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
        cumulativeEarmarked -= earmarkPaidGlobal;

```

## Impact Details

In the `_earmark()` function:

```solidity
    function _earmark() internal {
...

        uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;

...

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }
```

* `liveUnearmarked` will be `deflated` which will increase `_survivalAccumulator` and `_earmarkWeight`.

Similarly, In the `redeem()` function, `_survivalAccumulator` and `_redemptionWeight` will be `inflated`.

```solidity
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();
        uint256 liveEarmarked = cumulativeEarmarked;

...

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

...

```

`_sync()` function will use inflated weights to calculate incorrect `account.earmarked` and `account.debt`.

```solidity
    function _sync(uint256 tokenId) internal {
...
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);
        // User exposure at last sync used to calculate newly earmarked debt pre redemption
        uint256 userExposure = account.debt > account.earmarked ? account.debt - account.earmarked : 0;
        uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, _earmarkWeight - account.lastAccruedEarmarkWeight);

        // Earmark survival at last sync
        // Survival is the amount of unearmarked debt left after an earmark
        uint256 earmarkSurvival = PositionDecay.SurvivalFromWeight(account.lastAccruedEarmarkWeight);
        if (earmarkSurvival == 0) earmarkSurvival = ONE_Q128;
        // Decay snapshot by what was redeemed from last sync until now
        uint256 decayedRedeemed = _mulQ128(account.lastSurvivalAccumulator, survivalRatio);
        // What was added to the survival accumulator in the current sync window
        uint256 survivalDiff = _survivalAccumulator > decayedRedeemed ? _survivalAccumulator - decayedRedeemed : 0;

        // Unwind accumulated earmarked at last sync
        uint256 unredeemedRatio = _divQ128(survivalDiff, earmarkSurvival);
        // Portion of earmark that remains after applying the redemption. Scaled back from 128.128
        uint256 earmarkedUnredeemed = _mulQ128(userExposure, unredeemedRatio);
        if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;

        // Old earmarks that survived redemptions in the current sync window
        uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);
        // What was redeemed from the newly earmark between last sync and now
        uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
        // Total overall earmarked to adjust user debt
        uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

        account.earmarked = exposureSurvival + earmarkedUnredeemed;
        account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

...
```

The real issue is, `cumulativeEarmarked` compounds with each `liquidation` due to which `redemptions` may fail or return wrong amounts which breaks the whole internal accounting of the protocol, creates `bad debt` (We can redeem more amount than actual total earmarked as it is inflated) and make the protocol insolvent. Protocol accounting becomes `inconsistent` with each `liquidation`.

## References

The above code snippets can be verified here:

* \_forceRepay(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782>
* repay(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L525-L526>
* \_earmark(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1098-L1132>
* redeem(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L589-L641>
* \_sync(): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1042-L1095>

## Proof of Concept

## Proof of Concept

```solidity
function test_PocCumulativeEarmarked() external {    
    uint256 depositAmount = 1000e18;
    uint256 debtAmount = 900e18;
    
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    //Create position and borrow
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, debtAmount, address(0xbeef));
    vm.stopPrank();

    // Create healthy position for global collateralization
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
    alchemist.deposit(depositAmount * 2, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Create redemption to earmark debt
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debtAmount);
    transmuterLogic.createRedemption(debtAmount);
    vm.stopPrank();

    // Wait for full earmarking
    vm.roll(block.number + 5_256_000);

    // Check state before liquidation
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
    (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);

    // Create undercollateralization via price drop
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    // 10% price drop
    uint256 modifiedVaultSupply = (initialVaultSupply * 1000 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Liquidate (triggers _forceRepay internally)
    vm.startPrank(externalUser);
    alchemist.liquidate(tokenId);
    vm.stopPrank();

    // Check state after liquidation
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();
    (uint256 collateralAfter, uint256 debtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenId);

    // Calculate what was repaid
    uint256 debtRepaid = debtBefore - debtAfter;
    uint256 earmarkedRepaid = earmarkedBefore - earmarkedAfter;
    uint256 cumulativeEarmarkedChange = cumulativeEarmarkedBefore - cumulativeEarmarkedAfter;
    
    assertTrue(earmarkedRepaid > 0, "Earmarked should have been repaid");
    assertEq(cumulativeEarmarkedChange, 0, "Bug: cumulativeEarmarked not updated");
}
```

Paste the above test in `AlchemistV3.t.sol` contract and set up `$MAINNET_RPC_URL` then run it using:

```solidity
FOUNDRY_PROFILE=default forge test --fork-url $MAINNET_RPC_URL --match-path src/test/AlchemistV3.t.sol --match-test test_PocCumulativeEarmarked -vv --evm-version cancun
```
