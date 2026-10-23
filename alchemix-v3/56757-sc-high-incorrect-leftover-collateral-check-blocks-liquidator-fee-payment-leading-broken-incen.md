# 56757 sc high incorrect leftover collateral check blocks liquidator fee payment leading broken incentives delayed deleveraging

**Submitted on Oct 20th 2025 at 12:42:31 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56757
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

In `AlchemistV3._doLiquidation`, the protocol debits the **gross** seized collateral (including the liquidator’s base fee) from the victim, then always forwards the **net** amount to the Transmuter-but only pays the **base fee** to the liquidator if the account still has “leftover” collateral after that gross debit.

When the seizure consumes all collateral (common), the fee is withheld, making liquidations uneconomic. Reduced liquidator participation leads to stalled deleveraging, which can **temporarily freeze user funds** that depend on liquidations to proceed.

## Vulnerability Details

`src/AlchemistV3.sol`, liquidation path inside `_doLiquidation(...)`.

**What happens :**

```solidity
// inside _doLiquidation(...)
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, /* ... */) = calculateLiquidation(/*...*/);

uint256 amountLiquidated = convertDebtTokensToYield(liquidationAmount); // gross in MYT shares
uint256 feeInYield      = convertDebtTokensToYield(baseFee);           // base fee in MYT shares

// 1) Debit victim by the *gross* seizure (net + base fee)
account.collateralBalance = account.collateralBalance > amountLiquidated
  ? account.collateralBalance - amountLiquidated
  : 0;

// 2) Always send the *net* to the Transmuter
TokenUtils.safeTransfer(myt, address(transmuter), amountLiquidated - feeInYield);

// 3) Pay the base fee to liquidator *only if leftover collateral remains*
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

**Why this is a bug**

* The base fee is *part of the gross seizure already removed from the victim*. Payment to the liquidator should **not** depend on any post-seizure “leftover collateral” in the account.
* If the seizure consumes the account (common on marginal positions), `account.collateralBalance == 0` and the conditional check fails—**no fee is paid**. Value stays pooled in the contract (not stolen), but the liquidator isn’t compensated as designed.

**Concrete example**

* Victim has `X` MYT shares; liquidation computes `amountLiquidated = X` and `feeInYield = f` (`0<f<X`).
* `_doLiquidation` debits `X` (victim balance → `0`), sends `X-f` to Transmuter, then checks if `0 >= f` to pay the fee → **false** → fee **not paid**.
* Liquidator executed the work but received **no base fee** even though it was included in the gross seizure.
* Liquidations rely on external actors (searchers/keepers). If fees are frequently withheld in “full wipe” cases, the expected payout is inconsistent or negative (after gas + risk). Rational liquidators stop participating.
* With fewer liquidations, unhealthy positions linger; vault rebalances and user withdrawals that depend on deleveraging can **revert or be delayed** until market conditions or buffers change.

## Impact Details

* **Temporary freezing of funds for at least 1 hour.** When liquidations become uneconomic, deleveraging stalls. Withdrawals/rebalances that require collateral to be freed or debt to be burned can revert, temporarily freezing user funds until incentives are fixed or conditions change.
* Any liquidation that fully (or nearly fully) consumes a victim’s collateral can exhibit the issue.
* Repeated occurrences reduce keeper participation across the system, compounding deleveraging delays during stress—precisely when timely liquidations are most needed.

## References

* **Contract:** `src/AlchemistV3.sol`
* **Function:** `_doLiquidation(...)` (liquidation path where fee is gated by `account.collateralBalance >= feeInYield` **after** debiting the gross `amountLiquidated`): <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L878-L880>
* **Related types:** `calculateLiquidation(...)` (provides `liquidationAmount` and `baseFee`), `convertDebtTokensToYield(...)`, and calls to `TokenUtils.safeTransfer(...)` for Transmuter and liquidator payouts. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L1244-L1291>

## Proof of Concept

## Proof of Concept

Paste the following in `AlchemistV3.t.sol` and run with `forge test --match-test test_LiquidationBaseFee_Withheld_When_NoLeftoverCollateral`:

```solidity
// ------------------- PoC: Liquidation base fee withheld -------------------

contract LiqBaseFeeBugHarness {
    IERC20 public immutable myt;
    address public immutable transmuter;

    // Minimal replica of the account collateral mapping the real code uses
    mapping(address => uint256) public collateralBalance;

    constructor(address _myt, address _transmuter) {
        myt = IERC20(_myt);
        transmuter = _transmuter;
    }

    function seedCollateral(address victim, uint256 amt) external {
        collateralBalance[victim] = amt;
    }

    /// Mirrors the buggy payout logic:
    /// - Remove gross from victim first
    /// - Always send net to transmuter
    /// - Send base fee to liquidator ONLY IF leftover collateral >= fee
    function doLiquidation(address victim, uint256 amountLiquidated, uint256 feeInYield, address liquidator) external {
        uint256 bal = collateralBalance[victim];

        // 1) Gross debit
        collateralBalance[victim] = bal > amountLiquidated ? bal - amountLiquidated : 0;

        // 2) Always forward net to transmuter
        require(myt.transfer(transmuter, amountLiquidated - feeInYield), "net xfer failed");

        // 3) Fee gated on "leftover collateral" (bug)
        if (feeInYield > 0 && collateralBalance[victim] >= feeInYield) {
            require(myt.transfer(liquidator, feeInYield), "fee xfer failed");
        }
    }
}

contract AlchemistV3_LiquidationBaseFee_PoC is Test {
    function test_LiquidationBaseFee_Withheld_When_NoLeftoverCollateral() public {
        // Actors
        address victim       = address(0xCAFE);
        address transmuter   = address(0xBEEF);
        address liquidator   = address(this); // test contract plays liquidator

        // Token + harness
        TestERC20 myt = new TestERC20(1_000_000e18, 18); // mints to this test
        LiqBaseFeeBugHarness h = new LiqBaseFeeBugHarness(address(myt), transmuter);

        // Parameters (yield units)
        uint256 X = 100e18; // gross seized
        uint256 f = 2e18;   // base fee (0 < f < X)

        // Fund the harness with exactly the seized gross amount
        myt.transfer(address(h), X);
        // Victim's collateral equals the gross seizure (so leftover becomes 0 after debit)
        h.seedCollateral(victim, X);

        // Baselines
        uint256 transBefore = myt.balanceOf(transmuter);
        uint256 liqBefore   = myt.balanceOf(liquidator);

        // Act: run liquidation with the buggy payout rule
        h.doLiquidation(victim, X, f, liquidator);

        // Assert: collateral fully consumed
        assertEq(h.collateralBalance(victim), 0, "gross should be debited fully");

        // Net always forwarded to transmuter
        assertEq(myt.balanceOf(transmuter) - transBefore, X - f, "net must go to transmuter");

        // Base fee is withheld (not paid to liquidator) because leftover == 0
        assertEq(myt.balanceOf(liquidator) - liqBefore, 0, "base fee improperly withheld");

        // The withheld fee remains on the contract (stuck)
        assertEq(myt.balanceOf(address(h)), f, "withheld fee remains on contract");
    }
}
```
