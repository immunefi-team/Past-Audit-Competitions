# 58287 sc high mytsharesdeposited is not updated on some token transfer

**Submitted on Oct 31st 2025 at 23:58:15 UTC by @a16 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58287
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The *\_mytSharesDeposited* variable that is meant to track the amount of myt shares deposited is sometimes not updated, leading to potential DoS of deposits due to caps.

## Vulnerability Details

In previous versions of the code \[1], the cap was checked again the actual yieldToken balance of the AlchemistV3 contract.

```
    _checkState(IERC20(yieldToken).balanceOf(address(this)) + amount <= depositCap);
```

The code was later changed to support internal accounting using the *\_mytSharesDeposited* variable, that is updated whenever myt are deposited/withdrawn.

However, in some parts of the code myt are transferred without a corresponding update to the *\_mytSharesDeposited* variable, leading to an inherent mismatch between the two quantities and incorrect accounting.

For example, inside \_forceRepay() myt are transferred, but *\_mytSharesDeposited* is never decreased to reflect that.

```
    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    return creditToYield;
```

## Impact Details

Since the protocol always increases *\_mytSharesDeposited* when myt is transferred to the AlchemistV3 contract (only through deposit) but sometime does not decrease *\_mytSharesDeposited* when tokens are sent out of the contract, it is expected that \_mytSharesDeposited will grow over time (even if the net myt balance remains about the same). This could lead to deposit DoS due to the depositCap, which is compared to the sum of *\_mytSharesDeposited* and the deposited amount.

## References

\[1] Previous commit- <https://github.com/alchemix-finance/v3-poc/blob/audit-contest/src/AlchemistV3.sol#L361C1-L361C89>

## Proof of Concept

function test\_ForceRepay\_ShowsSharesMismatch\_BlocksDeposits() external { address trueAdmin = alchemist.admin(); address myt = address(vault); uint256 extra = 5e18;

```
// Zero fees so ONLY creditToYield moves MYT out.
vm.startPrank(trueAdmin);
alchemist.setDepositCap(type(uint256).max);
alchemist.setRepaymentFee(0);
alchemist.setProtocolFee(0);
alchemist.setLiquidatorFee(0);
vm.stopPrank();

// Seed strategy MYT
vm.startPrank(someWhale);
IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
vm.stopPrank();

// Keep global TVL healthy
vm.startPrank(yetAnotherExternalUser);
SafeERC20.safeApprove(myt, address(alchemist), 0);
SafeERC20.safeApprove(myt, address(alchemist), depositAmount * 2);
alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
vm.stopPrank();

// Victim opens a position and borrows at the bound
vm.startPrank(address(0xbeef));
SafeERC20.safeApprove(myt, address(alchemist), 0);
SafeERC20.safeApprove(myt, address(alchemist), depositAmount + 100e18);
alchemist.deposit(depositAmount, address(0xbeef), 0);
uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
uint256 tv = alchemist.totalValue(tokenId);
uint256 debt = (tv * FIXED_POINT_SCALAR) / minimumCollateralization;
alchemist.mint(tokenId, debt, address(0xbeef));
vm.stopPrank();

// Full redemption
vm.startPrank(anotherExternalUser);
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 0);
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt);
transmuterLogic.createRedemption(debt);
vm.stopPrank();
vm.roll(block.number + 5_256_000);

// Deflate yield token price so liquidations are allowed
uint256 s0 = IERC20(address(mockStrategyYieldToken)).totalSupply();
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s0);
uint256 s1 = (s0 * 3000 / 10_000) + s0; // +30%
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s1);

// --- BEFORE logs
uint256 balBefore = IERC20(myt).balanceOf(address(alchemist));
uint256 underlyingBefore = alchemist.getTotalUnderlyingValue(); // valuation (not raw MYT)
uint256 depositedBefore = alchemist.getTotalDeposited();        // internal deposited counter (used by cap)
console.log("=== BEFORE forceRepay ===");
console.log("MYT balance (balBefore):"); console.logUint(balBefore);
console.log("_getTotalUnderlyingValue (underlyingBefore):"); console.logUint(underlyingBefore);
console.log("getTotalDeposited() (depositedBefore):"); console.logUint(depositedBefore);

// Trigger repay-only via liquidate 
vm.startPrank(externalUser);
(uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
vm.stopPrank();
(, uint256 debtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenId);
require(debtAfter == 0, "forceRepay did not clear all debt");
require(earmarkedAfter == 0, "earmarked not cleared");
require(feeInYield == 0 && feeInUnderlying == 0, "fees must be zero");

// --- AFTER logs
uint256 balAfter = IERC20(myt).balanceOf(address(alchemist));
uint256 underlyingAfter = alchemist.getTotalUnderlyingValue();
uint256 depositedAfter = alchemist.getTotalDeposited();
console.log("=== AFTER forceRepay ===");
console.log("forceRepay assets (paid to transmuter):"); console.logUint(assets);
console.log("MYT balance (balAfter):"); console.logUint(balAfter);
console.log("_getTotalUnderlyingValue (underlyingAfter):"); console.logUint(underlyingAfter);
console.log("getTotalDeposited() (depositedAfter):"); console.logUint(depositedAfter);

// Show that Alchemist actually spent MYT.
require(balAfter < balBefore, "MYT balance did not decrease");
// Underlying may not track balance due to price changes; no assert here.

// Set cap to the *post-balance view* + extra. deposit() must revert if counter wasn't decremented
vm.startPrank(trueAdmin);
alchemist.setDepositCap(balAfter + extra);
vm.stopPrank();

vm.startPrank(someWhale);
SafeERC20.safeApprove(myt, address(alchemist), 0);
SafeERC20.safeApprove(myt, address(alchemist), extra);
vm.expectRevert(); // _checkState(_mytSharesDeposited + amount <= depositCap)
alchemist.deposit(extra, someWhale, 0);
vm.stopPrank();
```

}
