# 57726 sc high alchemistv3 myt tvl accounting drift on liquidation forcerepay blocks deposits via depositcap medium smart contract unable to operate due to lack of token funds&#x20;

**Submitted on Oct 28th 2025 at 13:55:09 UTC by @humaira45 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57726
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

AlchemistV3 tracks “how much MYT the protocol holds” in a private variable \_mytSharesDeposited and derives the protocol TVL from it via getTotalUnderlyingValue(). The deposit flow enforces a hard ceiling depositCap using this same internal counter:

* deposit() requires \_mytSharesDeposited + amount <= depositCap.

However, when MYT actually leaves the contract during liquidation and forceRepay, the code transfers MYT out but does not decrease \_mytSharesDeposited. This creates a persistent accounting drift: the internal TVL stays high (as if funds were still there) while the real MYT balance declined.

Consequence: the protocol believes the cap is “full” and rejects any new deposits, even though the contract’s real MYT balance dropped and there is room to accept more. This is a direct, repeatable DoS of deposits and therefore “Smart contract unable to operate due to lack of token funds” (Medium).

## Vulnerability Details

What the contract does

* TVL accounting:
  * getTotalUnderlyingValue() = convertYieldTokensToUnderlying(\_mytSharesDeposited)
  * deposit() gates on \_mytSharesDeposited + amount <= depositCap
* Outflows that move MYT out of the contract:
  * \_forceRepay(): transfers MYT to the Transmuter (and optionally protocol fee)
  * \_doLiquidation(): transfers MYT to the Transmuter (amountLiquidated - feeInYield) and, conditionally, feeInYield to the liquidator

Where the drift happens

* In both outflow paths, \_mytSharesDeposited is not reduced after MYT is transferred out:
  * \_forceRepay: after TokenUtils.safeTransfer(myt, transmuter, creditToYield) and TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal) there is no \_mytSharesDeposited decrement.
  * \_doLiquidation: after TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield) (and optional fee to the liquidator) there is no \_mytSharesDeposited decrement.

Why this breaks depositCap

* deposit() caps by \_mytSharesDeposited, not by the real ERC20(myt).balanceOf(address(this)).
* After liquidation/forceRepay, the real balance drops but \_mytSharesDeposited stays unchanged.
* Any subsequent deposit sees \_mytSharesDeposited still at the previous (higher) level, and the cap check reverts, effectively disabling deposits.

Root cause (code level)

* AlchemistV3.sol:

  ```
    _forceRepay(): misses _mytSharesDeposited -= creditToYield and (if paid) -= protocolFeeTotal.
   _doLiquidation(): misses _mytSharesDeposited -= (amountLiquidated - feeInYield) and (if paid) -= feeInYield.
   deposit(): relies on _mytSharesDeposited in  _checkState(_mytSharesDeposited + amount <= depositCap).
  ```

## Impact Details

Severity: Medium — Smart contract unable to operate due to lack of token funds

* The deposit entrypoint is permanently gated by a stale internal counter. The protocol cannot accept new MYT even though the real token balance fell, until an admin intervention or a code fix realigns accounting.
* This is not cosmetic: it blocks a core function and can last indefinitely in production.

## References

* In-scope: AlchemistV3.sol
  * \_forceRepay(), \_doLiquidation(), deposit(), getTotalUnderlyingValue()
* Ancillary: TokenUtils.safeTransfer (used to move MYT out), convertYieldTokensToUnderlying/convertToAssets (used in TVL derivation)

## Link to Proof of Concept

<https://gist.github.com/humairar301-droid/c6e8a3c2e80d3f8fc8be83050b9a6440>

## Proof of Concept

## Proof of Concept

What this PoC proves (end-to-end)

* Initializes AlchemistV3 via ERC1967Proxy with:
  * depositCap = 400e18; minimumCollateralization m = 2.0; liquidatorFee = 100% (to make a visible outflow on liquidation).
* Two users deposit 200 MYT each → cap “full”.
* Both mint 100 debt (safe at m=2).
* Liquidator liquidates user A at the boundary (ratio == m):
  * Gross seize amountLiquidated = 200; feeInYield = 100; fee isn’t paid (collateral becomes zero), so the real outflow is 100 MYT transferred to the Transmuter sink address (0xDEAD in the mock).
  * Real contract balance drops from 400 → 300.
  * BUG: \_mytSharesDeposited is unchanged; getTotalUnderlyingValue() still returns 400.
* A third user attempts to deposit 1 wei → deposit() reverts IllegalState because \_mytSharesDeposited + 1 > depositCap, even though real balance is now lower. This demonstrates a standing DoS of deposits due to stale internal accounting.

How to run

* This Gist contains the PoC: <https://gist.github.com/humairar301-droid/c6e8a3c2e80d3f8fc8be83050b9a6440>
* Save the test file as: src/test/AlchemistV3\_TVLDrift\_DepositCapDoS.t.sol
* Run: FOUNDRY\_PROFILE=default forge test --match-test test\_Deposit\_DoS\_DueToLiquidationDrift -vvvv --evm-version cancun

Representative results

* Before liquidation: getTotalUnderlyingValue() = 400e18; real MYT balance = 400e18.
* After liquidation of account 1:
  * amountLiquidated = 200e18; feeInYield = 100e18; fee not paid due to insufficient per-account collateral.
  * Real MYT balance = 300e18; getTotalUnderlyingValue() (driven by \_mytSharesDeposited) remains 400e18 (drift).
* A new deposit(1) reverts with IllegalState(), proving depositCap DoS.

Why this is in-scope and feasible

* In-scope asset: AlchemistV3 core logic.
* Impact mapped to the program taxonomy: Medium — “Smart contract unable to operate due to lack of token funds”.
* No exotic assumptions: this uses normal deposit/mint/liquidate flows; the outflow goes to a sink (Transmuter), as in production. The bug exists regardless of fee settings.

Suggested remediation

Minimal, localized fix (ABI-compatible)

```
Decrement _mytSharesDeposited whenever MYT leaves the Alchemist:
    In _forceRepay:
        If protocol fee is transferred: _mytSharesDeposited -= protocolFeeTotal;
        Always: _mytSharesDeposited -= creditToYield; (when > 0)
    In _doLiquidation:
        Always after transfer to transmuter: _mytSharesDeposited -= (amountLiquidated - feeInYield);
        If feeInYield is actually paid to the liquidator: _mytSharesDeposited -= feeInYield;
```

Notes

* This aligns outflow accounting with existing inflow/outflow updates (e.g., withdraw(), redeem(), burn()/repay() already decrease \_mytSharesDeposited for their outbound transfers).
* After the fix, getTotalUnderlyingValue() mirrors the real MYT balance, and depositCap reflects true headroom; the deposit DoS no longer occurs.

Optional hardening

* Add an invariant check in testing: assertEq(IERC20(myt).balanceOf(address(this)), convertYieldTokensToUnderlying(\_mytSharesDeposited)) within a small tolerance if share:asset is 1:1 (or scaled appropriately if not).
* Consider shifting depositCap gating to derive from real balance when feasible, or carefully prove the equivalence at all times.
