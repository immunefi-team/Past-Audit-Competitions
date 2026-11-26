# \#97625 [SC-Critical] Refundable positive VPI is counted in LP NAV when collected, so LPs who exit at high skew take value that the remaining Junior LPs later rebate to traders

**Submitted on Nov 2nd 2026 at 14:52:58 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97625
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The positive VPI that traders pay on skew-increasing opens is booked as pool revenue the moment it arrives, and the next reconcile credits it to tranche principal: to Junior, after Senior restoration and the coupon. The same VPI is refundable. A later skew-reducing close receives a rebate of up to the position's whole lifetime VPI, paid from pool cash and charged to Junior first. LP NAV carries no liability for this refundable amount, so an LP whose redemption settles while skew is high leaves with a share of VPI that the remaining LPs later pay back, and an LP who enters at high skew overpays by the same amount. At default parameters the transfer is bounded at about 8 bps of the pool, or about 40 bps of Junior NAV, per full maximum-skew cycle. A worked example on a 10M USDG pool moves about 3.2k USDG between Junior LPs.

## Vulnerability Details

### 1. VPI is the difference of a skew potential, so it is refundable

`CfdMath.tryGetSkewCost` defines C(S) = k·S²/(2D), and VPI charges C(S_post) − C(S_pre):

```solidity
// packages/perps/src/CfdMath.sol:134-139
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;
if (vpiFactorWad != 0 && sqSkewOverDepthWad > type(uint256).max / vpiFactorWad) {
    return (false, 0);
}
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
return (true, costWad / 1e12);
```

At constant depth the charges telescope. Starting from zero skew, the pool's net VPI is C(S_now), and traders get it back as skew returns to zero.

### 2. Collected VPI is recognised as revenue immediately

The open path records the trade cost, net of the protocol fee, as `Revenue` with `CashArrived`:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:454-462
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

`HousePool.recordClaimantInflow` adds it to `accountedAssets` at once (`HousePool.sol:640-648`). It is held back only when both tranche principals are zero.

### 3. NAV has no liability for refundable VPI

```solidity
// packages/perps/src/libraries/HousePoolAccountingLib.sol:63-77
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
```

Distributable assets are physical assets minus trader claims, plus or minus the terminal price delta. VPI is deliberately excluded from the price delta (`ACCOUNTING_SPEC.md:140-142`). `HousePool._reconcile` (`HousePool.sol:1555-1599`) then credits the new revenue to Junior through `distributeRevenue` (`HousePoolWaterfallAccountingLib.sol:140-159`). `_settleLpEpoch` prices redemptions from Junior principal immediately after that reconcile (`HousePool.sol:835-841`).

### 4. The refund reaches the whole lifetime VPI

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:107-118
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp only stops lifetime VPI going negative, so a skew-reducing close can be refunded everything it paid.
- A negative net action amount becomes a rebate (`CfdEnginePlanLib.sol:1041-1061`).
- The rebate is paid from free pool cash (`CfdEnginePlanLib.sol:970-978`) through `payOut` (`CfdEngineSettlementSidecar.sol:621-624`).
- At the next reconcile, `absorbLoss` charges the outflow to Junior first (`HousePoolWaterfallAccountingLib.sol:167-184`).

### 5. Nothing offsets the timing

`_settlementFeeBps` returns 0 whenever the oracle is live (`HousePool.sol:1147-1155`), and there is no skew-based LP gate. Skew is public and persists across epochs, and LP requests are made at least 5 minutes ahead of settlement.

### Documentation

`ACCOUNTING_SPEC.md:731-740` explicitly accepts LP-to-LP timing transfers for carry, as a "collection-time ownership policy". No document states or accepts the same transfer for refundable VPI.

## Impact Details

- **Who gains and who loses.** LPs who exit while skew is high, mainly Junior, take their share of collected VPI that the remaining LPs later rebate to traders. LPs who enter at high skew overpay by the same amount.
- **Size.** The transfer is bounded by C(0.4D) = 0.08·k·D per full maximum-skew cycle.
  - At defaults (k = 0.01, maxSkewRatio = 0.4) that is 8 bps of the pool gross, about 6.4 bps net of the 4 bps close fee that is netted against the rebate.
  - At the 20% Junior floor, it is up to about 40 bps of Junior NAV per cycle.
  - The exiting LP captures its fraction of Junior.
- **Worked example** (10M pool, 6M Senior and 4M Junior, an LP holding half of Junior; full steps in the PoC):
  - the exiting LP receives about 3,197 USDG more than its fair share;
  - the remaining Junior holders lose about 3,203 USDG, roughly 0.16% of their capital.
- **What shrinks it.** Skew unwinding through liquidations (which compute no fresh VPI), growth in pool depth, and the lifetime clamp. An LP who creates the skew themselves as a trader only profits above roughly 40% Junior ownership, after fees, carry and price risk.
- **Classification.** A bounded value transfer between LPs, in which remaining LPs do not receive the returns their NAV implies. Nothing is frozen, the pool stays solvent, and trader balances are unaffected.

## Recommendation

- Keep collected positive VPI out of the tranche waterfall until it can no longer be refunded.
  - Track an engine-side aggregate of refundable VPI, the sum over live positions of max(vpiAccrued, 0), updated on every open, increase, close and liquidation.
  - Subtract it, or min(aggregate, C(S_current, D_net)), from distributable assets in `HousePoolAccountingLib.buildReconcileSnapshot`, the one snapshot used for both entry and exit pricing.
  - Release it to Junior revenue when the paying lots are fully closed or liquidated, and pay close rebates against the deferred amount first.
- Alternatively, vest positive VPI into principal over time.
- At minimum, document the LP-to-LP timing consequence and its bound next to the carry policy in `ACCOUNTING_SPEC.md:731-740`.

## References

- VPI formula: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L118-L140
- Revenue booking at open: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463
- Immediate recognition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L648
- NAV snapshot without a VPI liability: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
- Reconcile and redemption pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841
- Waterfall revenue and loss: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L184
- Close VPI and lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- Rebate planning and payment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978
- No live settlement fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1147-L1155
- Spec on excluded quantities and on carry timing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L140-L142 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L731-L740
- Release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L434-L446


## Proof of Concept

## Proof of Concept

Release defaults: `vpiFactor` 0.01e18 and `maxSkewRatio` 0.4e18 (`script/DeployPerpsArbitrumSepolia.s.sol:434-446`), `maxSeniorShareBps` 8000, `executionFeeBps` 4, `settlementBufferBps` 25 and `seniorRateBps` 800.

Setup: pool depth D = 10,000,000 USDG, split 6,000,000 Senior and 4,000,000 Junior. LP B holds 50% of the Junior shares. The basket price stays unchanged throughout, so price PnL plays no part.

### Step by step

1. **Skew-increasing open.** Traders open 4,000,000 USDG of net LONG notional from zero skew.
   - The maximum skew at 10M depth is 4M, so the open is accepted.
   - VPI = 0.01 × (4e6)² / (2 × 1e7) = 8,000 USDG. It is recorded as `Revenue` / `CashArrived`, so `accountedAssets` becomes 10,008,000.
   - The 1,600 USDG execution fee goes to the treasury.
2. **B exits while skew is high.** B queued a Junior redemption before the cutoff, and `settleLpEpoch` reconciles first.
   - There is no price move and no trader claim, so distributable = 10,008,000.
   - After the one-hour Senior coupon of about 55 USDG, Junior principal is about 4,007,945.
   - The redemption budget allows full funding: free cash = 10,008,000 − 4,000,000 max liability − 10,000 buffer = 5,998,000, the Junior floor is 6M × 2000/8000 = 1.5M, and no fee applies while live.
   - B is paid about 2,003,972. Remaining Junior is about 2,003,973, and pool depth is about 8,004,028.
3. **Skew-reducing close.** The longs close voluntarily at the same price.
   - Fresh VPI = C(0) − C(4M) at depth 8,004,028 ≈ −9,995. The lifetime clamp limits it to −8,000.
   - The 1,600 close fee is netted against it, so the net action amount is about −6,400.
   - The pool pays a rebate of about 6,400 USDG.
4. **The loss lands on the remaining Junior.** At the next reconcile, `absorbLoss` charges it to Junior, leaving about 1,997,573.

### Counterfactual: B does not exit

Depth stays at 10,008,000. The fresh rebate is about 7,994 (below the clamp), so the net payout is about 6,394. Junior ends at about 4,001,552, which is about 2,000,776 per half.

### Result

- B received about 3,197 USDG more than its fair share (2,003,972 against 2,000,776).
- The remaining Junior holders lost about 3,203 USDG (1,997,573 against 2,000,776), roughly 0.16% of their capital. About 6 USDG of the difference comes from the depth change.
- A Junior depositor who activated at step 2 would pay a price that included the refundable 8,000 and would then absorb its share of the rebate.

### How to reproduce in a test

1. Seed a 6M Senior / 4M Junior pool, with LP B holding half the Junior shares.
2. Open 4M of net LONG notional from zero skew, and assert that pool revenue rises by about 8,000.
3. Settle an LP epoch in which B's full Junior redemption is funded. Record B's payout.
4. Close the longs at the same price, settle the next epoch, and record the remaining Junior principal.
5. Repeat from a snapshot without B's redemption. Compare B's payout and the remaining holders' value against half of the final Junior principal in that run.

