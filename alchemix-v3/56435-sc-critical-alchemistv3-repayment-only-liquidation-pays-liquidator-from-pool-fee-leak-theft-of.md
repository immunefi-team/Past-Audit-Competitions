# 56435 sc critical alchemistv3 repayment only liquidation pays liquidator from pool fee leak theft of unclaimed yield

**Submitted on Oct 15th 2025 at 23:06:08 UTC by @humaira45 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56435
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

In AlchemistV3’s liquidation pipeline, the “repayment‑only” branch (i.e., when earmarked debt fully clears a borrower’s debt or restores their ratio above the lower bound) pays a “repayment fee” to the liquidator. Due to a subtle accounting bug, the fee is not limited to what can actually be deducted from the borrower’s collateral. Instead, the contract transfers the full fee target from its own MYT balance (the shared pool), even if the borrower had less collateral than the fee.

In practice, this leaks MYT from the protocol pool to the liquidator. Our Foundry PoC shows a clean 5% leak per repayment‑only liquidation when minimumCollateralization = 110% and repaymentFee = 15%: the borrower can only fund 10% from collateral, yet the liquidator always receives 15%. We also top‑up the pool via a second depositor (not by minting straight to the contract) to remove any argument about artificial balance; the leak still drains pool liquidity and prevents other depositors from withdrawing their credited collateral.

## Vulnerability Details

What the contract does

* Earmark and repayment-first liquidation: When a position is liquidatable, AlchemistV3 first repays earmarked debt via \_forceRepay(), then:
* If debt is fully cleared or the account becomes healthy, the “repayment‑only” branch returns early (no seize), paying a “repayment fee” to the liquidator.
* Otherwise, it proceeds to seize collateral in \_doLiquidation().

Where the leak happens

```
In the repayment‑only branch of _liquidate():
    It calls _resolveRepaymentFee(accountId, repaidAmountInYield) to compute and deduct the fee from the borrower’s collateral.
    It then transfers feeInYield (the returned value) to the liquidator from the contract’s MYT balance.

In _resolveRepaymentFee(accountId, repaidAmountInYield):
    feeTarget = repaidAmountInYield × repaymentFee / BPS
    The function deducts at most the borrower’s remaining collateral: account.collateralBalance -= min(feeTarget, account.collateralBalance)
    BUG: it returns feeTarget (the target) rather than the amount actually deducted (feePaid).
    The caller (_liquidate) trusts that return value and sends the full feeTarget from the protocol pool to the liquidator.
```

Why this leaks funds

* When minimumCollateralization is 110%, a full earmark repay of debt D leaves borrower collateral ≈ 10% × D.
* With repaymentFee set to 15%, feeTarget is 15% × D. The borrower can only fund 10% × D; the 5% difference comes from the contract’s MYT balance (the shared pool), not the borrower.
* This is theft of unclaimed yield: liquidators are overpaid beyond what the borrower’s collateral actually covers.

Root cause (code level)

```
_resolveRepaymentFee() reduces collateral by min(feeTarget, collateralBalance) but returns feeTarget:
    Expected: return fee actually deducted (feePaid).
    Actual: return feeTarget regardless of what was deducted.
_liquidate() uses that return value and transfers feeInYield to msg.sender (the liquidator), paying from the contract’s MYT balance.
```

## Impact Details

Severity: High — Theft of unclaimed yield (explicitly in scope)

* Liquidators are paid more than the borrower can cover from their collateral; the delta is taken from the shared MYT pool.
* This can be farmed across many small accounts when earmarks are present (repayment‑only branch), cumulatively draining pool liquidity.

## References

* In‑scope contract: AlchemistV3.sol
  * Functions: \_liquidate(), \_resolveRepaymentFee()
* Related interfaces used in PoC:
  * IAlchemistV3Position (position NFT)
  * MockTransmuter (queryGraph drives earmark → repayment‑only path)

## Link to Proof of Concept

<https://gist.github.com/humairar301-droid/d5788d36cc5c5efeb36253c69a879f83>

## Proof of Concept

## Proof of Concept

What this PoC proves (end‑to‑end)

* Sets up AlchemistV3 via proxy (ERC1967Proxy) and realistic parameters (minCollat 110%, repaymentFee 15%, protocolFee 0).
* Borrower deposits 110 MYT and mints 100 debt; second depositor (whale) deposits 5 MYT via deposit() to ensure the pool has enough to pay a 15 MYT fee.
* Next block: liquidation triggers repayment‑only (full earmark repay). We assert: Event ForceRepay(..., creditToYield=100e18). Event RepaymentFee(..., fee=15e18). transmuter receives 100e18; liquidator receives 15e18 from the contract. Pool outflow = 115e18. Borrower’s fee actually deducted from collateral is only 10e18 (clamped).
  * The 5e18 difference is a leak from the pool (theft of unclaimed yield).
* The whale depositor then tries to withdraw 5e18; it reverts because the pool has been drained.

How to run

Full PoC (single file) Link Gist: <https://gist.github.com/humairar301-droid/d5788d36cc5c5efeb36253c69a879f83>

* Save the test as: src/test/AlchemistV3\_RepaymentFeeLeak.t.sol Run:
* forge test --match-test test\_RepaymentFeeLeak\_WithWhaleDeposit\_MismatchAndWithdrawFail -vvvv --evm-version cancun

Representative results:

```
Pool before liquidation: 115e18 (110 borrower + 5 whale via deposit()).
Events:
    ForceRepay(accountId: 1, amount: 100e18, creditToYield: 100e18, protocolFeeTotal: 0)
    RepaymentFee(accountId: 1, amount: 100e18, feeReceiver: liquidator, fee: 15e18)
Transfers:
    100e18 MYT → transmuter
    15e18 MYT → liquidator
Borrower fee actually deducted from collateral (computed from CDP deltas): 10e18
Pool outflow: 115e18; pool after: 0
Whale CDP shows 5e18 credited but withdraw(5e18) reverts (pool lacks funds) — clear user impact.
```

Why this is in‑scope and feasible

* In‑scope asset: AlchemistV3.sol (core protocol logic).
* Impact explicitly listed: “Theft of unclaimed yield” (High).
* No exotic assumptions:
  * We drive the normal liquidation path: earmark → \_forceRepay → repayment‑only branch.
  * The pool top‑up comes from a second user via the real deposit() flow (not by minting to the contract), so the leak demonstrably drains real depositor liquidity.
* Replicable: The attack can be repeated across many accounts with earmarks to cumulatively drain the pool.

Suggested remediation

```
Minimal, ABI‑compatible fix: return the fee actually deducted (feePaid) from _resolveRepaymentFee(), and pay the liquidator exactly that amount.
Example:

function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 feePaid) {
Account storage account = _accounts[accountId];
uint256 feeTarget = (repaidAmountInYield * repaymentFee) / BPS;
uint256 deductible = feeTarget > account.collateralBalance ? account.collateralBalance : feeTarget;
account.collateralBalance -= deductible;
emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, deductible);
return deductible; // Pay exactly what was actually deducted from collateral
}

Optional hardening:
    If collateral is insufficient, consider feePaid=0 (or pro‑rata) instead of subsidizing from the pool.
    Ensure event RepaymentFee logs the feePaid value, not the feeTarget.
```

The repayment‑only liquidation path overpays liquidators by charging the protocol pool, not just the borrower. Our PoC shows the mismatch on‑chain (event + transfer deltas) and demonstrates the immediate impact on another depositor. This is a clear High severity theft of unclaimed yield. The suggested one‑line fix (return feePaid, not feeTarget) resolves the issue without breaking external interfaces.
