# 56672 sc high inconsistent myt share accounting leads to under liquidation and solvency risk

**Submitted on Oct 19th 2025 at 08:15:09 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56672
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency

## Description

## Brief/Intro

`AlchemistV3._forceRepay` transfers MYT (vault shares) out of the Alchemist during repay-only liquidations but does not decrement the internal `_mytSharesDeposited` counter used to compute TVL.

As a result, `getTotalUnderlyingValue()` overstates collateral, the global collateralization ratio is inflated, and the system can skip emergency full-liquidation when it should trigger - creating persistent under-liquidation and elevated insolvency risk.

## Vulnerability Details

* The protocol tracks TVL via `_mytSharesDeposited`:

  ```solidity
  function _getTotalUnderlyingValue() internal view returns (uint256) {
      return convertYieldTokensToUnderlying(_mytSharesDeposited);
  }
  ```
* In `_forceRepay`, when earmarked debt is repaid using a user’s MYT collateral, the Alchemist **sends MYT out** (to the Transmuter and optionally the fee receiver) but **does not** decrease `_mytSharesDeposited`.

Elsewhere in the codebase, when MYT leaves the Alchemist, `_mytSharesDeposited` is lowered to keep TVL honest; this function breaks that invariant.

**Affected snippet (inside `_forceRepay`)**:

```solidity
// user collateral reduced…
account.collateralBalance -= creditToYield;
uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

// …but _mytSharesDeposited is NOT decremented before sending MYT out
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}
if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
}
```

* Because TVL is read from `_mytSharesDeposited`, not from the live MYT balance, every `_forceRepay` outflow **leaves reported TVL stale/too high**.

**Why this matters**

The liquidation controller uses TVL to compute the Alchemist’s **global collateralization** and decide whether to take the **emergency full-liquidation** branch:

```solidity
alchemistCurrentCollateralization =
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    // fully liquidate branch
}
```

If `_getTotalUnderlyingValue()` is overstated, `alchemistCurrentCollateralization` is inflated and the emergency branch can **fail to trigger** when the true (economic) ratio is already below the minimum. Positions that should be fully liquidated are only partially delevered or not delevered at all.

**Scenario PoC**

* Set fees to zero for clarity.
* Deposit MYT, mint debt, create and mature a redemption (to push the repay-only path).
* Force a price drop so `_forceRepay` sends MYT out to the Transmuter.
* Observe:
  * `vault.balanceOf(Alchemist)` **drops** (real MYT left).
  * `getTotalUnderlyingValue()` (derived from `_mytSharesDeposited`) **does not** drop accordingly (remains stale).
  * The computed global collateralization is **too high** vs. reality.

The coded `testForceRepay_MissingMYTOutflow_DeSync()` demonstrates exactly this: real MYT outflow occurs, but reported TVL is unchanged within tight tolerance.

## Impact Details

**Primary impact: Protocol insolvency (systemic).** By overstating TVL, the protocol underestimates risk and **skips emergency full-liquidations** that should occur. Under-liquidation lets unhealthy positions linger, allowing **bad debt to accumulate**, and reduces liquidator compensation (“outsourced fee”), weakening incentives.

**Secondary impacts:**

* **Temporary freezing of funds.** As the true asset base shrinks while TVL appears healthy, later withdrawals/redemptions can **revert** once the real shortfall surfaces.
* **Contract fails to deliver promised safety behavior.** The system’s liquidation guarantees rely on accurate TVL; misreporting breaks those guarantees.

**Concrete loss mechanics (example):**

* A repay-only liquidation transfers **100 MYT shares** out (to Transmuter + fees).
* Real MYT holdings drop by 100 shares; **reported TVL is unchanged**.
* The global collateralization ratio derived from TVL stays **artificially high** and emergency liquidation is **not triggered**.
* Repeating this across multiple undercollateralized accounts can **compound the TVL gap**, increasing bad debt and leading to **withdrawal failures**; in the worst case, overall **insolvency**.

## References

* Affected file: `src/AlchemistV3.sol` — `_forceRepay` outflows without `_mytSharesDeposited` decrement: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L764-L775>
* TVL source: `_getTotalUnderlyingValue()` uses `_mytSharesDeposited`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L1238-L1241>

## Proof of Concept

## Proof of Concept

Paste the following test in `AlchemistV3.t.sol` and run with `forge test --match-test testForceRepay_MissingMYTOutflow_DeSync`:

```solidity
function testForceRepay_MissingMYTOutflow_DeSync() public {
    // ---- actors from your declared globals ----
    address victim     = yetAnotherExternalUser; // 0x520a...
    address liquidator = externalUser;           // 0x69E8...

    // Choose the Transmuter address you actually call through.
    // Most setups use the proxy; fall back to the logic var if present.
    address transmuterAddr = address(proxyTransmuter) != address(0)
        ? address(proxyTransmuter)
        : address(transmuter);

    // ---- isolate effect: zero protocol/repayment fees via the correct governor ----
    address feeGovernor = address(0x000000000000000000000000000000000000dEaD);
    vm.startPrank(feeGovernor);
    alchemist.setProtocolFee(0);
    alchemist.setRepaymentFee(0);
    vm.stopPrank();

    // ---- victim deposits 100 MYT shares ----
    uint256 depositShares = 100e18;
    vm.startPrank(victim);
    vault.approve(address(alchemist), depositShares);
    alchemist.deposit(depositShares, victim, 0);
    vm.stopPrank();

    // position id
    uint256 victimId = alchemistNFT.tokenOfOwnerByIndex(victim, 0);

    // ---- borrow close to edge (99% of max) ----
    uint256 maxBorrow  = alchemist.getMaxBorrowable(victimId);
    uint256 debtToMint = (maxBorrow * 99) / 100;
    vm.prank(victim);
    alchemist.mint(victimId, debtToMint, victim);

    // ---- redemption that will mature (80% of debt) ----
    // Use the Transmuter PROXY address for spender/target if available.
    if (transmuterAddr != address(0)) {
        uint256 redeemAmt = (debtToMint * 80) / 100;
        vm.startPrank(victim);
        alToken.approve(transmuterAddr, redeemAmt);
        Transmuter(transmuterAddr).createRedemption(redeemAmt);
        vm.stopPrank();
    }

    // Let the redemption fully mature (matches horizon used elsewhere)
    vm.roll(block.number + 5_256_001);

    // ---- crash MYT price hard so liquidation uses _forceRepay path ----
    uint256 s0 = MockYieldToken(mockStrategyYieldToken).totalSupply();
    MockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s0 * 3); // ~ -67% price in this model

    // ---- snapshots before ----
    uint256 sharesBefore           = vault.balanceOf(address(alchemist));
    uint256 tvlBefore              = alchemist.getTotalUnderlyingValue();
    uint256 transmuterSharesBefore = transmuterAddr == address(0) ? 0 : vault.balanceOf(transmuterAddr);

    // ---- liquidate: triggers _forceRepay and sends MYT out ----
    vm.prank(liquidator);
    alchemist.liquidate(victimId);

    // ---- snapshots after ----
    uint256 sharesAfter           = vault.balanceOf(address(alchemist));
    uint256 tvlAfter              = alchemist.getTotalUnderlyingValue();
    uint256 transmuterSharesAfter = transmuterAddr == address(0) ? 0 : vault.balanceOf(transmuterAddr);

    // Sanity: MYT actually left the Alchemist
    assertLt(sharesAfter, sharesBefore, "MYT shares should have left Alchemist via _forceRepay");

    // If we had a real Transmuter wired, it should have received shares
    if (transmuterAddr != address(0)) {
        assertGt(transmuterSharesAfter, transmuterSharesBefore, "Transmuter should receive MYT shares");
    }

    // BUG ASSERTION:
    // TVL (derived from _mytSharesDeposited) should have dropped with the outflow,
    // but due to missing decrement in _forceRepay it stays stale (allow tiny rounding slack).
    if (tvlAfter > tvlBefore) {
        fail("TVL increased unexpectedly");
    }
    assertApproxEqAbs(
        tvlAfter,
        tvlBefore,
        1, // rounding tolerance
        "TVL stayed stale because _mytSharesDeposited was not decremented (BUG)"
    );

    uint256 realTVLBefore = alchemist.convertYieldTokensToUnderlying(sharesBefore);
    uint256 realTVLAfter  = alchemist.convertYieldTokensToUnderlying(sharesAfter);
    assertGt(realTVLBefore - realTVLAfter, 0, "real TVL didn't drop");
    assertApproxEqAbs(tvlAfter, tvlBefore, 3, "reported TVL should be stale if _mytSharesDeposited not decremented");

}
```
