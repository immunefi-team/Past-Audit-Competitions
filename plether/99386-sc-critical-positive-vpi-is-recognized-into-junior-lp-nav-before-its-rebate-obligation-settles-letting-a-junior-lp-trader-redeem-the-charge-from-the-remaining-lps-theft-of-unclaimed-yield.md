# \#99386 [SC-Critical] Positive VPI is recognized into Junior LP NAV before its rebate obligation settles, letting a Junior-LP trader redeem the charge from the remaining LPs (theft of unclaimed yield)

**Submitted on Nov 13th 2026 at 12:57:49 UTC by @chrissavov for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99386
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

### Brief/Intro

When a skew-increasing position opens, its positive VPI (virtual price impact) charge is booked as pool revenue in the same transaction, so it immediately raises the Junior share price. The identical charge is refundable to the trader on close; the only bound is a clamp keeping lifetime VPI non-negative. Between open and close the pool owes a contingent rebate that LP share pricing never reserves for. An actor who is both a Junior LP and a trader can open the position, redeem Junior shares at the inflated NAV while the position is open, then close and collect the VPI rebate out of pool cash. The rebate is funded by the LPs who did not redeem, so the attacker recovers its own VPI charge and takes its pro-rata share of the charge out of the pool with it. Every step is permissionless.

### Vulnerability Details

#### 1. Positive VPI becomes LP revenue immediately

`packages/perps/src/CfdEngineSettlementSidecar.sol:454-463`: the open's trade cost, which includes the positive VPI charge, is pushed into the pool as `Revenue` with `CashArrived`:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

On the next reconcile this revenue flows through the waterfall to Junior, and every subsequent deposit and redemption is priced at the higher NAV.

#### 2. The same charge is refundable on close

`packages/perps/src/libraries/CloseAccountingLib.sol:103-116`: a skew-reducing close produces a negative `vpiDeltaAssets` (a rebate), bounded only by the lifetime clamp:

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

At unchanged depth VPI telescopes (C(S) = k·S²/2D), so a trader who closes the skew it opened recovers exactly what it paid. The whitepaper confirms this directly (§5.1: "The VPI charge is $20,000. Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp."). The clamp's own comment shows the designers considered an LP sandwich that works by manipulating *depth*. This issue needs no depth manipulation; it exploits *when* LPs exit relative to the rebate.

#### 3. Nothing reserves the contingent rebate in LP pricing

While the position is open, `max(vpiAccrued, 0)` is a liability the pool may repay, yet it sits in LP equity and is redeemable. The protocol protects the opposite sign: negative lifetime VPI is held in a dedicated non-withdrawable reserve kept out of LP equity.

- `packages/perps/src/SECURITY.md:866`: "gross negative lifetime VPI is backed by a dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`".
- `ACCOUNTING_SPEC.md:798-799`: `vpiRebateReserveAssets = max(-vpiAccrued, 0)`; `actionReserveAssets >= protectedExecutionBountyAssets + vpiRebateReserveAssets`.

Positive VPI has no equivalent treatment, which also contradicts the stated pricing model:

- `SECURITY.md:863`: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book".
- `SECURITY.md:872`: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV".

In practice, paid VPI raises the price at which Junior redemptions are funded before the matching rebate obligation settles.

#### Why each step is reachable by an untrusted actor

| Step | Entry point | Access |
|---|---|---|
| LP deposit / redeem / claim | `TrancheVault.requestDeposit`, `requestRedeem`, `claimDeposit`, `claimRedeem` | Any LP |
| Open and close | `OrderRouter.commitOrder(OrderRequest)`, `OrderRouter.executeOrder` | Any trader; attacker can be its own keeper |
| Mark refresh | `OrderRouter.updateMarkPrice(bytes[])` | Permissionless |
| Epoch settlement | `OrderRouter.settleLpEpoch(bytes[])` | Permissionless; installs a PoolReconcile mark and settles in one frame |

LP epochs are one hour (`HousePool.LP_EPOCH_DURATION`) and Junior shares have a one-hour deposit cooldown (`TrancheVault.DEPOSIT_COOLDOWN`), so one cycle takes about three hours.

### Impact Details

In scope: **Theft of unclaimed yield.** The VPI charge is LP revenue the remaining Junior LPs would otherwise keep; the attacker recovers its own charge in full and carries its pro-rata share of that revenue out of the pool, leaving the staying LPs with less. Value moves from Junior LPs who stay to the attacker:

```
attacker gain ≈ w · VPI − (open + close execution fees) − carry
```

where `w` is the attacker's redeemable share of Junior. The release skew cap (`maxSkewRatio = 0.4`) bounds the charge at `VPI_max = k·(0.4D)²/2D = 0.0008·D` for `k = 0.01`. The two 4 bps execution fees cost `0.00032·D` at the cap, so the attack is net-profitable once `w` exceeds roughly 40%. Both the charge and the transfer scale linearly with pool depth `D`.

Measured results from the PoC on a $10M pool with a $3.875M notional position at the release parameters (VPI paid on open = **$7,506.31**):

| Scenario | Attacker realized P&L | Remaining Junior LPs realized P&L | Same LPs, honest round trip |
|---|---|---|---|
| Junior-only ($10M), attacker holds 50% of Junior | **+$636.83** | **−$2,186.43** | +$785.69 |
| $6M Senior + $4M Junior, attacker holds 50% of Junior | **+$581.49** | **−$2,294.77** | +$703.39 |
| Depth-neutral (unrelated LP deposits what the attacker redeems) | **+$631.19** | **−$2,181.40** (victim +$785.68, newcomer −$2,967.08) | +$785.69 |

- **Not depth manipulation.** In the depth-neutral run, pool depth at close equals the honest run ($10,009,522.248946 vs $10,009,522.244382) and the attacker's trading leg is identical to within 3 USDG atoms ($296,878.423579 vs $296,878.423582). The entire profit comes from Junior shares being priced with a VPI charge the pool still owes back — not from the size of the rebate. This separates the issue from the known "temporary pool-liquidity reduction can amplify a skew-reducing close rebate" scan finding.
- **Later LPs are also victims.** Anyone depositing Junior while such a position is open buys in at the inflated price; in the depth-neutral run the newcomer loses $2,967.08.
- **Repeatable** every cycle while the attacker can re-enter Junior.
- **Price risk is manageable.** The $3.875M directional position is held flat in the PoC, but a live attacker neutralizes the ~3 hours of basket exposure by hedging the six-currency basket off-venue, or by running the cycle only when already positioned directionally — so the extraction is not contingent on a favorable price move.

### References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (positive VPI booked as Revenue/CashArrived)
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-116` (close rebate, lifetime clamp)
- `packages/perps/src/CfdMath.sol` (`calculateVPI`, C(S)=k·S²/2D)
- `packages/perps/SECURITY.md:863, 866, 872` (VPI outside pre-close price book; negative-VPI reserve; VPI must not inflate NAV)
- `packages/perps/ACCOUNTING_SPEC.md:798-799` (`vpiRebateReserveAssets` floor — asymmetry vs positive VPI)
- `packages/perps/WHITEPAPER.md` §5.1 (telescoping: $20,000 charge → $20,000 rebate at unchanged depth)

## Link to Proof of Concept

https://gist.github.com/chrissavov/b4361ea3815b923ee493b063269f087e

## Proof of Concept

## Proof of Concept

### Step-by-step (Junior-only scenario)

1. The victim and the attacker each deposit $5M of Junior. Deposits settle through the permissionless epoch call.
2. The attacker deposits $300k of trader margin, then commits and executes a $3.875M long, paying **$7,506.31** of VPI. The charge is booked as pool Revenue and Junior NAV rises on the next reconcile.
3. After the one-hour deposit cooldown, the attacker requests the largest Junior redemption allowed. At the next epoch boundary any EOA settles it via `OrderRouter.settleLpEpoch`, and the attacker claims ~$5.0038M while the long is still open.
4. The attacker closes the long at the unchanged $1.25 basket. The close rebate returns the VPI it paid, out of pool cash that now belongs mostly to the victim.
5. Everyone exits. The attacker is up **+$636.83**; the victim is down **−$2,186.43**, where the honest round trip would have left it **+$785.69**. The difference is the attacker's extraction plus the victim's forfeited yield.

A runnable Foundry PoC is included: `packages/perps/test/perps/poc/VpiNavTimingPoC.t.sol`. Run from the repo root at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```bash
forge test --root packages/perps --match-path test/perps/poc/VpiNavTimingPoC.t.sol -vv
```

All three tests pass and log the control vs. attack P&L in the table above.

### Rules compliance
- No in-scope contract is modified; the PoC extends the repo's own `BasePerpTest`.
- No privileged address is impersonated after `setUp`. Every action uses a public entry point, and epochs are settled by an unrelated EOA through `OrderRouter.settleLpEpoch(bytes[])`. Orders use the production `commitOrder(OrderRequest)`, not the harness's scalar shortcut.
- Only Pyth is mocked. The basket is held flat at $1.25 (≠ $1.00), so the measured transfer is VPI-only. Carry (500 bps) and the 4 bps execution fee are live and paid by the attacker.
- Risk parameters are the DEPLOYMENT.md release values; the Senior scenario applies the 80% Senior cap through the Owner timelock (propose → wait 72h → finalize).
- All P&L is realized: every party redeems every share and the attacker withdraws its margin. No preview/estimate view feeds an assertion. Each test runs the honest round trip and the attack from the same snapshot on the same timeline (same carry), so the harm is the difference between two executions of the real code.

### Tests
- `test_JuniorOnly_Attacker50pct_ExtractsFromRemainingLp`
- `test_WithSenior_Attacker50pctOfJunior_ExtractsFromRemainingLp`
- `test_DepthNeutral_SameTradingLeg_ProfitComesFromLpPricing` — holds depth at close equal to the control (within 0.01%) and the trading leg equal (within 1 USDG), proving the profit flows through LP share pricing, not the rebate size.
