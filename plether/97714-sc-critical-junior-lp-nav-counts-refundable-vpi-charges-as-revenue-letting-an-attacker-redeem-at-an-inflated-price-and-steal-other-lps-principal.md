# \#97714 [SC-Critical] Junior LP NAV counts refundable VPI charges as revenue, letting an attacker redeem at an inflated price and steal other LPs' principal

**Submitted on Nov 2nd 2026 at 19:34:49 UTC by @Attaullah_Wazir for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97714
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When an open increases market skew, the trader pays a VPI charge that `HousePool` immediately books as Junior LP revenue, yet the same position can reclaim that charge in full as a rebate when it closes. LP share pricing never reserves this refundable amount. An attacker who is both a trader and a Junior LP (one address is enough) can therefore open a large position just before an LP epoch boundary, have its Junior redemption funded at a NAV inflated by its own VPI charge, and then close the position to take the charge back out of the remaining LPs' capital. With the release parameters and an unchanged market price, the attacker makes a net profit every hourly epoch cycle and passive Junior LPs lose principal (−714.65 USDG per cycle on a 1,000,000 USDG LP in the PoC). The attack is repeatable, needs no privileged role, no oracle manipulation and no keeper role, and scales linearly with pool size.

## Vulnerability Details

### 1. A positive VPI charge is booked as Junior revenue at open

`CfdEngineSettlementSidecar.executeOpen` collects the trade cost (VPI + execution fee) and records everything except the protocol fee as claimant revenue. The next reconcile credits it to Junior principal.

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol  L446-L464
(int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
    delta.account, delta.marginDeltaAssets, delta.tradeCostAssets, host.pool(),
    host.protocolTreasury(), delta.executionFeeAssets
);
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

### 2. The same position can reclaim the whole charge on close

A skew-reducing close produces a negative fresh VPI. The lifetime clamp only prevents the position's lifetime VPI from going below zero, so the position may recover everything it paid:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol  L103-L118
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

### 3. The rebate is paid from pool cash, which belongs to the LPs still in the pool

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol  L1041-L1061 (_planCloseActionSettlement)
int256 actionNetAssets = cs.vpiDeltaAssets + int256(cs.executionFeeAssets + cs.frozenSpreadAssets
    + delta.pendingCarryAssets - delta.realizedCarryAssets);
...
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets);
}

// packages/perps/src/CfdEngineSettlementSidecar.sol  L621-L624 (executeClose)
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

### 4. LP pricing never reserves the refundable charge

`HousePool._settleLpEpoch` reconciles and prices the Junior redemption phase directly from `juniorPrincipal`, which already includes the VPI charge:

```solidity
// packages/perps/src/HousePool.sol  L835-L841 and L878-L889
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();
...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
...
juniorPhase.budget = _juniorSettlementCapacity(freeAssets, _getWaterfallState());
juniorPhase = _fundRedemptionPhase(juniorVault, result.cutoffEpoch, juniorPhase, ...);
```

The withdrawal reserve (`C + L + B`) covers trader claims, the maximum price liability and the settlement buffer, but not the open positions' refundable positive VPI. By design, VPI is kept out of LP pricing ("carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book", SECURITY.md "Exact symmetric LP NAV"). The protocol does protect the opposite direction: negative VPI is held in a nonwithdrawable reserve that "cannot be extracted by LPs before its account obligation settles" (ACCOUNTING_SPEC.md). The positive direction, where the charge is paid first and refunded later, has no equivalent. The total of `max(vpiAccrued, 0)` across open positions is an unreserved liability inside Junior NAV.

### 5. Exploit path (one attacker address, all public functions)

1. The attacker deposits into the Junior tranche (`TrancheVault.requestDeposit` → epoch settlement → `claimDeposit`) and waits for the 1-hour activation cooldown.
2. It calls `TrancheVault.requestRedeem` for all its shares, which targets the next epoch boundary `b`.
3. At `b − 120 s` it commits a large skew-increasing LONG with `OrderRouter.commitOrder`. The order is executed with `OrderRouter.executeOrder` (by any keeper), and the position pays VPI `V` into the pool.
4. After `b`, the epoch is settled with `OrderRouter.settleLpEpoch(bytes[])`, by the attacker or by any keeper. The attacker's redemption is funded at a Junior NAV that includes `V`.
5. The attacker commits a full close, which is executed. The skew-reducing close reclaims `V` as a rebate paid from the remaining LPs' cash.
6. The attacker calls `TrancheVault.claimRedeem`.

The trade leg is VPI-neutral and costs only two execution fees (4 bps each). The LP leg keeps `f × V`, where `f` is the attacker's share of the Junior tranche being redeemed. The mark price never changes, so no price view is needed, and the position is open for only about 2 minutes. The existing mitigations against VPI "LP sandwich" attacks (the lifetime clamp and the 1-hour LP cooldown) are both active and do not prevent the attack. They target pool-depth manipulation of the VPI amount; here the extraction comes from LP pricing recognising a refundable charge. In the PoC the attacker's redemption shrinks pool depth so much that the fresh close rebate would be about −4,503 USDG; the clamp correctly cuts it to the 1,878.64 USDG actually paid, and the attack still succeeds.

## Impact Details

- **Selected impact:** Critical, *Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.* Passive Junior LPs lose principal: in the PoC the victim's redeemable value drops below the 1,000,000 USDG it deposited and keeps falling every cycle, with the market price unchanged. The attacker ends each cycle with a net profit after every fee and carry payment.
- **Measured losses** (release v2.0.0 economics, competition-page Senior limits, constant mark; victim Junior LP 1,000,000 USDG; attacker Junior LP 1,400,000 USDG; attack position 950,000 USDG notional):

| Scenario | Attacker net PnL (USDG) | Victim Junior LP PnL (USDG) |
|---|---|---|
| One cycle | +335.34 | −714.65 |
| One cycle, a third-party keeper executes and settles everything | +335.14 | −714.65 |
| One cycle, honest trader already holds a 400k LONG; attacker owns ~41% of Junior, 280k LONG | +141.21 | −252.99 |
| Five consecutive cycles | +1,683.22 | −3,579.78 |
| Control: same LP cycle without the trade | −0.01 | −0.02 |
| Control: same trade without redeeming | −539.08 | +158.89 |

- **Scaling:** profit per cycle is approximately `f·k·S·(2S₀+S)/(2D) − 2·feeBps·S`, where `S` is the attack notional, `S₀` is existing same-direction skew, `D` is pool depth and `k` is the VPI factor. At release parameters (`k = 0.01`, fee 4 bps, maximum skew 40% of depth) the attack is profitable once the attacker redeems more than about 40% of the Junior tranche with no prior skew. Existing honest skew lowers that threshold (about 23% when honest skew is 30% of depth). All amounts scale linearly with pool depth.
- **Repeatability:** one LP position completes a cycle every ~3 hours (deposit epoch, 1-hour cooldown, redemption epoch); two staggered LP positions allow one extraction per hourly epoch. Each cycle removes at least `f × V` from the remaining LPs. The attacker's capital is returned every cycle, and market exposure lasts only about 2 minutes.
- **Preconditions:** live market hours, the attacker holds a large enough Junior share (capital is temporary and fully returned), and there is enough free cash to fund the redemption (true in the PoC). No privileged role, no keeper role, no oracle manipulation and no second account are required.

## References

- In-scope code (commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L878-L889
- Documentation:
  - SECURITY.md "Exact symmetric LP NAV": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L876
  - ACCOUNTING_SPEC.md negative-VPI reserve rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L821
  - WHITEPAPER.md §5.1 Virtual price impact: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L816-L904
- Release parameters used in the PoC (v2.0.0 manifest "economics"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json#L165-L195

## Recommendation

Treat outstanding positive lifetime VPI as refundable (deferred) revenue for LP pricing. In reconcile and in the withdrawal snapshot, deduct the total of `max(vpiAccrued, 0)` across open positions (the most the lifetime clamp lets them reclaim) from distributable equity, or hold positive VPI in a dedicated pool bucket that is excluded from tranche principal. Release it to the waterfall only when the position's VPI is realised at close or liquidation. This mirrors the existing protection for negative VPI. Alternatively, recognise VPI as revenue only at close.

## Link to Proof of Concept

https://gist.github.com/Attaullahwazir/c0167f890e46648057bf1059b6d6c11b

## Proof of Concept

### Environment

- Repository `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The PoC runs in the repository's own Foundry test suite; no in-scope contract is modified.
- The PoC test file `packages/perps/test/perps/audit/C004_VpiRedemptionExtraction.t.sol` (full source in the secret Gist) extends the repository's `BasePerpTest`.
- Only Pyth (`MockPyth`, constant price `1e8`) and USDG (mock settlement token) are mocks, as the competition rules allow.
- Engine risk parameters are the release v2.0.0 manifest "economics": `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, base carry 500 bps, execution fee 4 bps, settlement buffer 25 bps, frozen close spread 50 bps. The competition-page Senior limits (40,000,000 USDG and 80%) are applied by the owner through the normal 72-hour timelock.
- The market is live (weekday). The keeper is permissionless; one test uses a neutral third-party keeper to show the attacker does not need that role.

### Step-by-step

1. **Setup (trusted deployment steps from `BasePerpTest`):** the tranches are seeded and trading is activated. The owner proposes and finalises the pool config `maxSeniorExposureAssets = 40,000,000e6` and `maxSeniorShareBps = 8000` after the 72-hour timelock.
2. **Victim LP:** `VICTIM_LP` calls `juniorVault.requestDeposit(1_000_000e6, VICTIM_LP, VICTIM_LP)`. The epoch is settled through `router.settleLpEpoch(pythData)`, and `VICTIM_LP` calls `juniorVault.claimDeposit(...)`. The victim's Junior value is ≈1,000,000 USDG (minus sub-cent Senior-coupon dust).
3. **Attacker becomes an LP:** the single address `ATTACKER` calls `juniorVault.requestDeposit(1_400_000e6, ATTACKER, ATTACKER)`. After `router.settleLpEpoch`, it calls `claimDeposit` and waits the 1-hour activation cooldown. It also holds 50,000 USDG of free margin in `MarginClearinghouse`, deposited with `deposit`.
4. **Redemption request:** `ATTACKER` calls `juniorVault.requestRedeem(allShares, ATTACKER, ATTACKER)`. This targets the next epoch boundary `b` (requested more than 5 minutes before `b`).
5. **Skew-increasing open:** at `b − 120 s`, `ATTACKER` calls `router.commitOrder` for a LONG of 9,500 lots (950,000 USDG notional) with 9,500 USDG margin. Any keeper calls `router.executeOrder(orderId, pythData)`, and the order executes at the unchanged price.
   - VPI charged and stored in `vpiAccrued`: **1,878.642797 USDG**.
   - Execution fee to the treasury: 380 USDG.
   - Junior NAV (`juniorVault.totalAssets()`): **2,400,999.31 → 2,402,877.95**, up by exactly the VPI charge.
6. **Epoch settlement:** after `b`, any keeper (or the attacker) calls `router.settleLpEpoch(pythData)`. The attacker's redemption is fully funded: `claimableRedeemAssets` = **1,401,095.408191 USDG** for a 1,400,000 USDG deposit.
7. **Skew-reducing close:** `ATTACKER` commits a full close and a keeper executes it. The close reclaims the VPI. The fresh rebate of about −4,503 USDG is clamped to the 1,878.64 USDG accrued, and the net rebate after the close fee is paid from pool cash. Junior NAV, now owned almost entirely by the victim: **1,001,782.54 → 1,000,283.97**.
8. **Claim:** `ATTACKER` calls `juniorVault.claimRedeem(...)`. Its Junior share balance is 0.
9. **Result:**
   - `ATTACKER` total value (wallet + clearinghouse custody + Junior shares at vault NAV): **+335.336312 USDG**.
   - `VICTIM_LP` Junior value: **−714.649555 USDG**, below its 1,000,000 USDG deposit (asserted).
   - The mark never moved.
10. **Repeat:** five consecutive cycles give the attacker **+1,683.222007 USDG** and the victim **−3,579.781695 USDG**.

### Controls (same file)

- Same LP cycle without the trade: attacker −0.010658 USDG, so the redemption alone is not profitable.
- Same trade without the redemption: attacker −539.077748 USDG, so the trade alone only costs fees.
- Neutral third-party keeper: attacker +335.136312 USDG, so no keeper role is needed.
- Honest pre-existing 400k LONG skew, attacker LP 700k (~41% of Junior), 280k attack: attacker +141.212578 USDG, victim −252.987171 USDG.

### Run

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/audit
# save the Gist file C004_VpiRedemptionExtraction.t.sol into packages/perps/test/perps/audit/
forge test --root packages/perps --match-path test/perps/audit/C004_VpiRedemptionExtraction.t.sol -vv
```

### Expected output

```
[PASS] test_Control_SameLpCycleWithoutTradeIsNotProfitable()
[PASS] test_Control_SameTradeWithoutRedeemingIsNotProfitable()
[PASS] test_PoC_ExtractionRepeatsEveryCycle()            attacker PnL 1683222007, victim -3579781695
[PASS] test_PoC_HonestSkewLowersRequiredLpShare()        attacker PnL 141212578,  victim -252987171
[PASS] test_PoC_RedeemingLpCapturesSelfRebatedVpi()      attacker PnL 335336312,  victim -714649555
[PASS] test_PoC_WorksWithNeutralProtocolKeeper()         attacker PnL 335136312,  victim -714649555
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

(Amounts are 6-decimal USDG atoms. Compilation uses via-IR and can take several minutes on first run.)
