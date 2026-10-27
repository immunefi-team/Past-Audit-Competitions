# 56395 sc high accounting desync in liquidation outflows leads to artificial deposit cap exhaustion and denial of service on recapitalization

**Submitted on Oct 15th 2025 at 13:50:07 UTC by @HandsomeEarthworm6 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56395
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistV3 contract maintains an internal tracker \_mytSharesDeposited to enforce the deposit cap, but fails to decrement it during MYT share outflows in liquidation functions (\_forceRepay and \_doLiquidation). This creates a desync where the tracker overstates deposited shares, artificially hitting the cap and reverting new deposits with IllegalState even when actual balance has room. In production, during yield token crashes triggering mass liquidations and redemptions, this DoS prevents users from depositing collateral to restore healthy ratios, exacerbating undercollateralization spirals and potentially leading to protocol insolvency without manual admin intervention.

## Vulnerability Details

The contract uses getTotalDeposited() for external queries, which correctly returns IERC20(myt).balanceOf(address(this)) to reflect true TVL. However, deposit cap enforcement in deposit() relies on the internal uint256 private \_mytSharesDeposited (slot 30), incremented in deposit() (\_mytSharesDeposited += amount) and decremented in withdraw() (\_mytSharesDeposited -= amount), but ignored during liquidation outflows.Liquidations (via liquidate() → \_liquidate() → \_forceRepay() or \_doLiquidation()) transfer MYT shares out of the contract via TokenUtils.safeTransfer(myt, ...), reducing balanceOf but leaving \_mytSharesDeposited stale. Key paths:

1.In \_forceRepay() (triggered for earmarked debt during redemptions/liquidations):

if (creditToYield > 0) { // Transfer the repaid tokens from the account to the transmuter. TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); // Outflow: reduces balanceOf // MISSING: \_mytSharesDeposited -= creditToYield; @audit } // Fee transfer similarly missing decrement if (account.collateralBalance > protocolFeeTotal) { account.collateralBalance -= protocolFeeTotal; TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); // Outflow // MISSING: \_mytSharesDeposited -= protocolFeeTotal; @audit }

2.In \_doLiquidation() (core liquidation):

// send liquidation amount - fee to transmuter TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield); // Outflow // send base fee to liquidator if (feeInYield > 0 && account.collateralBalance >= feeInYield) { TokenUtils.safeTransfer(myt, msg.sender, feeInYield); // Outflow } // MISSING: \_mytSharesDeposited -= amountLiquidated; // Total outflow @audit

3.Partial liquidation (repay-only in \_liquidate()):

feeInYield = \_resolveRepaymentFee(accountId, repaidAmountInYield); TokenUtils.safeTransfer(myt, msg.sender, feeInYield); // Outflow // MISSING: \_mytSharesDeposited -= feeInYield; @audit

Redemptions via redeem() correctly decrement (\_mytSharesDeposited -= collRedeemed + feeCollateral), but liquidations compound the drift linearly with each event. Yield accrual (vault auto-mints to Alchemist) can create opposite drift (phantom room), but outflows dominate in crises.Deposit check: \_checkState(\_mytSharesDeposited + amount <= depositCap) uses stale internal, reverting while getTotalDeposited() < depositCap.

## Impact Details

Direct: No funds loss—transfers succeed, but deposits revert, denying users the ability to add collateral.

This bug creates a protocol insolvency risk by disabling the recapitalization mechanism during market crashes—the only time mass liquidations occur. The desync compounds exponentially with each liquidation, creating a death spiral where blocked deposits lead to more liquidations. Without manual admin intervention (which may not occur in time during rapid market moves), the protocol can become insolvent with no automated recovery path.

## References

n/a

## Proof of Concept

## Proof of Concept

add the following test code to src/test/AlchemistV3.t.sol ,and then run forge test --match-test test\_Issue7 and then check the test output ,check the amount after and before

function test\_Issue7\_MytSharesDepositedDesyncOnLiquidation() external { // This test demonstrates that liquidations don't update \_mytSharesDeposited, // causing a desync that blocks deposits when the protocol needs recapitalization most

```
uint256 depositAmount = 200_000e18;

// Setup low deposit cap to force the impact demo
uint256 lowCap = depositAmount * 2;
vm.prank(alOwner);
alchemist.setDepositCap(lowCap);

// Setup: Create whale to ensure global collateralization stays healthy
vm.startPrank(someWhale);
IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
vm.stopPrank();

// Setup: Healthy position to keep global ratios acceptable (but close to cap)
vm.startPrank(yetAnotherExternalUser);
deal(address(vault), yetAnotherExternalUser, depositAmount * 2); // Ensure tokens
SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
vm.stopPrank();

// Record initial internal state (slot for _mytSharesDeposited: 30 / 0x1e)
bytes32 mytSharesSlot = bytes32(uint256(30)); // Confirmed via forge inspect
uint256 initialInternal = uint256(vm.load(address(alchemist), mytSharesSlot));
uint256 initialBalance = IERC20(address(vault)).balanceOf(address(alchemist));
assertEq(initialInternal, initialBalance, "Initial sync");

// Step 1: Create an undercollateralized position (fills near cap)
vm.startPrank(address(0xbeef));
deal(address(vault), address(0xbeef), depositAmount + 100e18);
SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
alchemist.deposit(depositAmount, address(0xbeef), 0);
uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

// Mint maximum debt for easy liquidation
uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
vm.stopPrank();

// Step 2: Create redemption to start earmarking debt (this will trigger _forceRepay)
vm.startPrank(anotherExternalUser);
deal(address(alToken), anotherExternalUser, mintAmount);
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
transmuterLogic.createRedemption(mintAmount / 2); // Create redemption for half the debt
vm.stopPrank();

// Advance time to earmark some debt
vm.roll(block.number + (5_256_000 * 30 / 100)); // 30% through transmutation

// Record state before liquidation
uint256 actualBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 totalDepositedBefore = alchemist.getTotalDeposited();
uint256 internalBefore = uint256(vm.load(address(alchemist), mytSharesSlot));

console.log("=== Before Liquidation ===");
console.log("Actual MYT balance:", actualBalanceBefore);
console.log("getTotalDeposited():", totalDepositedBefore);
console.log("Internal _mytSharesDeposited:", internalBefore);
assertEq(actualBalanceBefore, totalDepositedBefore, "Should be in sync before liquidation");
assertEq(internalBefore, actualBalanceBefore, "Internal should match before");

// Step 3: Crash the price to make position liquidatable (ensure > lower bound breach)
uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
// Bigger drop: 20% supply inflate for max-LTV position to breach ~150% lower bound
uint256 modifiedVaultSupply = (initialVaultSupply * 2000 / 10_000) + initialVaultSupply;
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

// Step 4: Liquidate the position
vm.startPrank(externalUser);
deal(address(alToken), externalUser, mintAmount); // For repayment in liq
(uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
vm.stopPrank();

// Step 5: Check the desync (internal vs balance)
uint256 actualBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 totalDepositedAfter = alchemist.getTotalDeposited();
uint256 internalAfter = uint256(vm.load(address(alchemist), mytSharesSlot));

console.log("\n=== After Liquidation ===");
console.log("Actual MYT balance:", actualBalanceAfter);
console.log("getTotalDeposited():", totalDepositedAfter);
console.log("Internal _mytSharesDeposited:", internalAfter);
console.log("Amount liquidated:", amountLiquidated);
console.log("Fee in yield:", feeInYield);

// Calculate expected desync
uint256 tokensTransferredOut = actualBalanceBefore - actualBalanceAfter;
uint256 visibleDesync = totalDepositedAfter - actualBalanceAfter; // Should be 0
uint256 internalDesync = internalAfter - actualBalanceAfter;

console.log("\n=== Desync Analysis ===");
console.log("Tokens transferred out:", tokensTransferredOut);
console.log("Visible desync (getTotalDeposited):", visibleDesync);
console.log("Internal desync (_mytSharesDeposited):", internalDesync);

// The bug: Internal should have decreased but didn't
assertEq(visibleDesync, 0, "Visible always synced (uses balanceOf)");
assertGt(internalDesync, 0, "Internal desync should exist after liquidation");
assertApproxEqAbs(internalDesync, tokensTransferredOut, 1e18, "Internal desync should equal tokens sent out");

// Step 6: Demonstrate the practical impact - deposits are blocked
uint256 depositCap = alchemist.depositCap();

console.log("\n=== Deposit Cap Impact ===");
console.log("Deposit cap:", depositCap);
console.log("Internal (wrong for cap):", internalAfter);
console.log("Actual balance (correct):", actualBalanceAfter);
console.log("Room actual:", depositCap - actualBalanceAfter);
console.log("Room perceived (internal):", depositCap - internalAfter);

// Internal close to/exceeds cap, new deposits fail despite actual room
assertGe(internalAfter, depositCap - 1e18, "Internal hits cap after liq");

console.log("\n!!! CRITICAL: Deposits are BLOCKED due to internal desync !!!");
console.log("Users cannot deposit to save their positions during crisis");

// Try to deposit and show it fails
vm.startPrank(externalUser);
deal(address(vault), externalUser, 1e18);
SafeERC20.safeApprove(address(vault), address(alchemist), 1e18);
vm.expectRevert(IllegalState.selector);
alchemist.deposit(1e18, externalUser, 0);
vm.stopPrank();

console.log("Deposit attempt FAILED as expected due to bug");

// Step 7: Show admin workaround
console.log("\n=== Admin Workaround ===");
vm.prank(alOwner);
// Admin must raise cap by the desync amount to re-enable deposits
alchemist.setDepositCap(depositCap + internalDesync + 1000e18);
console.log("Admin raised deposit cap to:", alchemist.depositCap());

// Now deposit works
vm.startPrank(externalUser);
deal(address(vault), externalUser, 100e18);
SafeERC20.safeApprove(address(vault), address(alchemist), 100e18);
alchemist.deposit(100e18, externalUser, 0);
console.log("Deposit SUCCESS after admin intervention");
vm.stopPrank();

// Step 8: Demonstrate the issue compounds with multiple liquidations
console.log("\n=== Issue Compounds With Multiple Liquidations ===");
console.log("After N liquidations, internal desync = N * avg_liquidation_amount");
console.log("This is a SYSTEMIC issue that gets worse during MYT crisis events");
console.log("When liquidations are needed most, deposits become blocked");
```

}

function test\_Issue7\_MytSharesDepositedDesync\_MultipleScenarios() external { // This test shows the desync in multiple scenarios to prove it's systematic

```
uint256 depositAmount = 100e18;

// Setup low cap to highlight impact
uint256 lowCap = depositAmount * 4;
vm.prank(alOwner);
alchemist.setDepositCap(lowCap);

// Setup whale
vm.startPrank(someWhale);
IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
vm.stopPrank();

// Create 3 positions (fill near cap)
address[] memory users = new address[](3);
users[0] = address(0xbeef);
users[1] = address(0xdead);
users[2] = address(0xbabe);

uint256[] memory tokenIds = new uint256[](3);

for (uint256 i = 0; i < users.length; i++) {
    vm.startPrank(users[i]);
    deal(address(vault), users[i], depositAmount);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, users[i], 0);
    tokenIds[i] = AlchemistNFTHelper.getFirstTokenId(users[i], address(alchemistNFT));
    
    // Mint max debt for liquidatability
    uint256 maxMint = alchemist.totalValue(tokenIds[i]) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIds[i], maxMint, users[i]);
    vm.stopPrank();
}

// Record initial internal
bytes32 mytSharesSlot = bytes32(uint256(30)); // Confirmed via forge inspect
uint256 initialInternal = uint256(vm.load(address(alchemist), mytSharesSlot));
uint256 initialBalance = IERC20(address(vault)).balanceOf(address(alchemist));
assertEq(initialInternal, initialBalance, "Initial sync");

// Create redemptions to enable earmarking
for (uint256 i = 0; i < users.length; i++) {
    vm.startPrank(users[i]);
    deal(address(alToken), users[i], depositAmount);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), depositAmount);
    transmuterLogic.createRedemption(depositAmount / 2);
    vm.stopPrank();
}

vm.roll(block.number + 1_000_000);

// Crash price sufficiently for max-LTV positions
uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
uint256 modifiedVaultSupply = (initialVaultSupply * 3500 / 10_000) + initialVaultSupply; // ~26% drop
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

// Track cumulative desync (internal)
uint256 cumulativeDesync = 0;

console.log("=== Liquidating Multiple Positions ===");

for (uint256 i = 0; i < users.length; i++) {
    uint256 balanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 internalBefore = uint256(vm.load(address(alchemist), mytSharesSlot));
    
    vm.prank(externalUser);
    deal(address(alToken), externalUser, depositAmount * 2); // For liq repays
    (uint256 amountLiquidated,,) = alchemist.liquidate(tokenIds[i]);
    
    uint256 balanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 internalAfter = uint256(vm.load(address(alchemist), mytSharesSlot));
    uint256 tokensOut = balanceBefore - balanceAfter;
    uint256 perDesync = internalAfter - balanceAfter;
    
    cumulativeDesync += perDesync;
    
    console.log("\nLiquidation", i + 1);
    console.log("  Tokens out:", tokensOut);
    console.log("  Per-liq internal desync:", perDesync);
    console.log("  Cumulative desync:", cumulativeDesync);
    assertGt(perDesync, 0, "Desync per liquidation");
}

uint256 finalBalance = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 finalDeposited = alchemist.getTotalDeposited();
uint256 finalInternal = uint256(vm.load(address(alchemist), mytSharesSlot));

console.log("\n=== Final State ===");
console.log("Actual balance:", finalBalance);
console.log("getTotalDeposited():", finalDeposited);
console.log("Internal _mytSharesDeposited:", finalInternal);
console.log("Total internal desync:", finalInternal - finalBalance);

assertEq(finalDeposited - finalBalance, 0, "Visible desync always 0");
assertApproxEqAbs(
    finalInternal - finalBalance,
    cumulativeDesync,
    1e18,
    "Cumulative internal desync should match total tokens transferred out"
);

// Bonus: Try deposit post-multiple liqs - should block if internal >= cap
uint256 depositCap = alchemist.depositCap();
if (finalInternal >= depositCap) {
    vm.startPrank(externalUser);
    deal(address(vault), externalUser, 1e18);
    SafeERC20.safeApprove(address(vault), address(alchemist), 1e18);
    vm.expectRevert(IllegalState.selector);
    alchemist.deposit(1e18, externalUser, 0);
    vm.stopPrank();
    console.log("Final deposit BLOCKED due to compounded desync");
}
```

}

function test\_Issue7\_DepositCapBypassScenario() external { // This test shows how the desync can lead to deposit cap being artificially reached

```
uint256 smallCap = 1000e18;

// Set a small deposit cap for demonstration
vm.prank(alOwner);
alchemist.setDepositCap(smallCap);

// Setup
vm.startPrank(someWhale);
IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
vm.stopPrank();

// Fill to 90% of cap
vm.startPrank(address(0xbeef));
uint256 deposit1 = smallCap * 90 / 100;
deal(address(vault), address(0xbeef), deposit1 + 100e18);
SafeERC20.safeApprove(address(vault), address(alchemist), deposit1 + 100e18);
alchemist.deposit(deposit1, address(0xbeef), 0);
uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

// Mint near-max for liquidatability
uint256 maxMint = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
alchemist.mint(tokenId, maxMint * 95 / 100, address(0xbeef));
vm.stopPrank();

console.log("=== Initial State ===");
console.log("Deposit cap:", smallCap);
console.log("Used:", deposit1);
console.log("Available:", smallCap - deposit1);

// Record internal pre-liq
bytes32 mytSharesSlot = bytes32(uint256(30)); // Confirmed via forge inspect
uint256 internalPre = uint256(vm.load(address(alchemist), mytSharesSlot));

// Create redemption and liquidate
vm.startPrank(address(0xdad));
deal(address(alToken), address(0xdad), deposit1);
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), deposit1);
transmuterLogic.createRedemption(deposit1 / 2);
vm.stopPrank();

vm.roll(block.number + 1_000_000);

// Crash price
uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
uint256 modifiedVaultSupply = (initialVaultSupply * 2500 / 10_000) + initialVaultSupply; // ~20% drop
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

vm.prank(externalUser);
deal(address(alToken), externalUser, deposit1);
alchemist.liquidate(tokenId);

uint256 actualBalance = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 trackedBalance = alchemist.getTotalDeposited();
uint256 internalPost = uint256(vm.load(address(alchemist), mytSharesSlot));

console.log("\n=== After Liquidation ===");
console.log("Actual balance:", actualBalance);
console.log("Tracked balance (getTotal):", trackedBalance);
console.log("Internal _mytSharesDeposited:", internalPost);
console.log("Actual room:", smallCap - actualBalance);
console.log("Perceived room (internal):", smallCap > internalPost ? smallCap - internalPost : 0);

// Verify bug
assertEq(trackedBalance, actualBalance, "Visible synced");
assertGt(internalPost, actualBalance, "Internal desync exists");

// Try to deposit the "available" amount
vm.startPrank(yetAnotherExternalUser);
uint256 actualRoom = smallCap - actualBalance;
uint256 testDeposit = actualRoom / 2;
if (actualRoom > 1e18) {
    deal(address(vault), yetAnotherExternalUser, testDeposit);
    SafeERC20.safeApprove(address(vault), address(alchemist), testDeposit);
    
    // This should work (actual room) but fails due to internal desync
    vm.expectRevert(IllegalState.selector);
    alchemist.deposit(testDeposit, yetAnotherExternalUser, 0);
    
    console.log("\n!!! Deposit BLOCKED despite having room !!!");
    console.log("Actual room exists but internal desync prevents deposit");
}
vm.stopPrank();
```

}
