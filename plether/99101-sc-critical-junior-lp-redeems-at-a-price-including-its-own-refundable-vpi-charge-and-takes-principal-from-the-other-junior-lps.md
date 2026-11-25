# \#99101 [SC-Critical] Junior LP redeems at a price including its own refundable VPI charge and takes principal from the other Junior LPs

**Submitted on Nov 11th 2026 at 21:31:29 UTC by @truthlover for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99101
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

`HousePool` counts a position's positive price-impact charge (VPI) in Junior NAV at the open, although the position reclaims it as a rebate at the close. An unprivileged Junior LP that also trades opens at the skew cap just before an LP epoch boundary, redeems at the boundary at a price including the charge, then closes and reclaims it. Each three-hour cycle takes principal from the remaining Junior LPs, with no price move or privileged action. With 45% of Junior at 1 bp oracle confidence on a flat book, three cycles raise the attacker's wallet by 286.71 USDG and cost the honest Junior LPs 1,597.86 USDG of principal. This is direct theft of user funds (Critical).

## Vulnerability Details

The protocol keeps VPI that a position can still reclaim out of LP hands. The close-side lifetime clamp is commented as one that "Prevents LP sandwich attacks" built from an open, a change in pool depth, and a close [1], and the security notes say it "intentionally zeroes otherwise extractable rebate-only round trips" [2]. On the reserve side, VPI reserve "cannot be extracted by LPs before its account obligation settles" [3]. The specification states an explicit LP timing acceptance only for carry [4]. Positive VPI breaks the same guarantee from the other side: an exiting LP is paid its share of a charge that the still-open position then reclaims from the LPs who stay. The notes say "neither anticipated VPI nor its reserve inflates" symmetric NAV [5]; the collected, still-reclaimable charge does count in NAV, so it sits inside the exit price.

Root cause:

1. A skew-increasing open computes its VPI charge `X` and adds it to the trade cost [6]. The settlement sidecar records the pool's share of that cost as claimant revenue with cash arrived [7], which increases `accountedAssets` [8]. The position stores `vpiAccrued = +X`.
2. `OrderRouter.settleLpEpoch` is permissionless [9]. It calls `HousePool.settleLpEpoch`, which reconciles (counting `X` in Junior equity) and then funds matured Junior redemptions at that Junior price [10]. The funded amount leaves `juniorPrincipal` and `accountedAssets` as a fixed cash claim [11]; the vault burns the shares.
3. The close computes a fresh VPI delta and clamps it only so that lifetime VPI (`vpiAccrued` plus the delta) does not fall below zero [1]. A skew-reducing close therefore receives a rebate of up to `X`, paid from pool cash. Junior NAV carries no provision for that rebate while the position is open.

Attack, one cycle, by one address that is both a Junior LP and a trader:

1. Deposit USDG into the Junior vault; claim the shares when the deposit activates at the next epoch boundary.
2. After the one-hour deposit cooldown, request redemption of all shares before the five-minute cutoff, so that it matures at the next boundary B.
3. About 12 seconds before B, commit a LONG open at the skew cap; it executes in the next block and pays `X` plus the execution fee.
4. At B, `settleLpEpoch` funds the redemption at a Junior price that includes `X`; the attacker claims its USDG, carrying its share `s` of `X`.
5. Commit a full close; it executes in the next block and receives a rebate equal to `X` from the pool.
6. Deposit again and repeat.

The remaining Junior LPs fund the whole rebate after being credited only `(1 - s)` of the charge. The clamp does what its comment says for the trader (lifetime VPI ends at exactly zero, and the rebate never exceeds the charge at any close), but the value leaves through the redemption price, which the clamp does not see. Execution fees and the oracle's adverse-confidence shift on both legs [12], the cooldown, the request cutoff, hourly epochs and one-pass funding all run; they set the break-even share and the three-hour period.

The transfer does not depend on pool depth moving between open and close, which separates it from the AI-scan item on close-rebate amplification through temporary depth reduction [13], whose root is the execution-time depth denominator. When an unrelated LP's deposit restores depth before the close, the attacker's gain is unchanged (+612.15 against +612.52 USDG at 1 bp) and the depth effect is 1.69 USDG of a transfer of about 1,875 USDG.

Another instance of the same root cause: an LP whose deposit activates at the same boundary buys in at the inflated price and pays the rebate (logged at -1,449.18 USDG against its no-trade baseline at 1 bp).

Recommendation: keep reclaimable positive VPI out of LP entry and exit prices. Treat the outstanding positive `vpiAccrued` of open positions as a liability in the reconcile that prices Junior redemptions and deposits, mirroring the negative-VPI reserve [3], and release to LP equity only the part a close does not rebate. This covers both the redeem and the deposit side.

## Impact Details

The remaining Junior LPs lose principal; part of it reaches the attacker as cash and the rest is paid as execution fees on the attacker's trades. The pool holds an illustrative 1M USDG Senior and 1M USDG of honest Junior split across two LPs (the release seeds each tranche with 1 USDG [14]), with an unrelated trader's position open throughout. Three cycles over 8.99 hours at 1 bp oracle confidence on a flat book give, against the same LP round trips without the trade:

| attacker share of Junior | attacker wallet change, attack / control | honest Junior LPs' loss on full redemption |
|---|---|---|
| 45% (843,182.63 USDG tied up) | +286.71 / -24.65 USDG (gain over control +311.37) | -1,597.86 USDG |
| 43% | +151.13 / -23.56 USDG (gain over control +174.69) | -1,430.78 USDG |

Senior is not the payer (it redeems 1,000,091.32 USDG in attack and control alike). Moving the exit off the boundary removes the harm: in single-cycle controls at a 66.66% share the staying Junior LPs end +231.66 USDG with the trade but no exit at the boundary, and +222.52 USDG with the exit one epoch after the close. A 1 bp price move between each open and close, in either direction, keeps both effects (on average +311.50 USDG to the attacker and -1,598.02 USDG to the honest Junior LPs at a 45% share).

At the oracle's 10 bps ceiling the attacker loses money at a 45% share, and break-even needs 58.42% of Junior.

Prerequisites, each created by the attacker or ordinary for the market:

- Junior capital: profit at 1 bp needs roughly 43% of Junior or more (single-cycle break-even 42.55%), and the capital comes back in full at every boundary.
- Skew headroom below the 40% cap [14]: a flat book provides it.
- Keeper actions: LP epoch settlement and order execution are permissionless [9][15], as are mark updates; in the proof a separate keeper address performs every one of them.
- Oracle confidence: 1 bp is above the basket confidence of all nine signed six-feed updates in the repository's fixtures (`test/fixtures/oracle-sync`, 0.21 to 0.93 bp, decoded by `fixture_basket_confidence.py` in the gist).

Recovery: a funded redemption is a fixed cash claim outside `accountedAssets` [11], and nothing claws it back; the loss becomes cash when the remaining LPs redeem. An LP epoch settlement hold (`HousePool.pauseLpEpochSettlement` [16], or the guardian's `EmergencyPauseCoordinator.triggerLpEpochSettlementHold` [17]) applied after a cycle leaves that cycle's loss unchanged (-532.39 USDG with and without the hold) and only stops later cycles.

Severity: Critical, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". The deciding qualifier is that the staying Junior LPs end below their no-trade principal and an attacker holding less than half of Junior takes part of that principal as cash, the rest being paid as execution fees.

## References

1. `CloseAccountingLib` proportional accrual, close VPI and lifetime clamp with its anti-sandwich purpose, `packages/perps/src/libraries/CloseAccountingLib.sol` L103-L118: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
2. Lifetime clamp intent, `packages/perps/SECURITY.md` L1443: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443
3. VPI reserve not extractable by LPs, `packages/perps/ACCOUNTING_SPEC.md` L821-L822: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L821-L822
4. Accepted LP timing transfer, carry only, `packages/perps/ACCOUNTING_SPEC.md` L737-L740: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L737-L740
5. Anticipated VPI kept out of symmetric NAV, `packages/perps/SECURITY.md` L871-L872: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L871-L872
6. `OpenAccountingLib` VPI charge in trade cost, `packages/perps/src/libraries/OpenAccountingLib.sol` L127-L135: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135
7. `CfdEngineSettlementSidecar` open cost recorded as pool revenue, `packages/perps/src/CfdEngineSettlementSidecar.sol` L446-L463: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L463
8. `HousePool.recordClaimantInflow`, `packages/perps/src/HousePool.sol` L624-L650: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650
9. `OrderRouter.settleLpEpoch` (permissionless), `packages/perps/src/OrderRouter.sol` L474-L484: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484
10. `HousePool.settleLpEpoch` and `_settleLpEpoch` (reconcile, then fund), `packages/perps/src/HousePool.sol` L797-L836: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L836
11. Junior funding leaves `juniorPrincipal` and `accountedAssets`, `packages/perps/src/HousePool.sol` L879-L887: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L879-L887
12. `PletherOracle._adverseOrderPrice`, `packages/perps/src/PletherOracle.sol` L905-L923: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L905-L923
13. AI scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate", `audits/scan-plether-fi-plether-core-d4541b49.md` L1742-L1809: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1809
14. Release parameters (`vpiFactor`, `maxSkewRatio`, `executionFeeBps`, tranche seeds), `packages/perps/DEPLOYMENT.md` L125-L156: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L125-L156
15. `OrderRouter.executeOrder` (permissionless order execution), `packages/perps/src/OrderRouter.sol` L240-L255: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L240-L255
16. `HousePool.pauseLpEpochSettlement` and `unpauseLpEpochSettlement`, `packages/perps/src/HousePool.sol` L398-L416: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L398-L416
17. `EmergencyPauseCoordinator.triggerLpEpochSettlementHold`, `packages/perps/src/EmergencyPauseCoordinator.sol` L142-L150: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/EmergencyPauseCoordinator.sol#L142-L150


## Link to Proof of Concept

https://gist.github.com/torxeon/f27fb0f0097fc8e55c2f9923dd0dc6c0

## Proof of Concept


The test drives the real router, engine, pool and vaults at the release parameters and shows an unprivileged address holding 45% of Junior, which is both LP and trader, gaining over three cycles while the staying Junior LPs lose principal on full redemption.

The full test file, a fixture decoder and a README are in a private gist: https://gist.github.com/torxeon/f27fb0f0097fc8e55c2f9923dd0dc6c0. The test goes to `packages/perps/test/VpiLpExitPoC.t.sol` of a checkout of https://github.com/immunefi-team/audit-comp-plether at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and builds on the project's own fixture (`test/perps/BasePerpTest.sol`). It mocks only out-of-scope components: `MockPyth` holds every feed at 1.00000000 (moved by 1 bp in test 7) with the confidence set by `POC_CONF_E8`, and mock USDG supplies starting capital. `setUp` applies the release parameters of `packages/perps/DEPLOYMENT.md`, including the oracle's 2,500 adverse-confidence multiplier and 1 USDG tranche seeds, through the owner admin paths and asserts them. Actors sign with pranks and wait with time warps; snapshots start each attack and its control from one identical state. No in-scope contract is modified, and no storage is written directly. A separate keeper address sends every mark update, order execution and epoch settlement. The tests assert the honest Junior LPs' loss at every confidence level and the attacker's gain at confidence up to 1 bp.

```bash
POC_CONF_E8=10000 forge test --root packages/perps --match-path test/VpiLpExitPoC.t.sol -vv
```

The attacker's cycle, `_cycle` (lines 403-460, excerpt; in the three-cycle tests the LP and the trader are one address):

```solidity
        // 1. cooldown: one hour after the deposit was claimed
        _tick(1 hours);
        uint256 r;
        uint256 boundary = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        if (exitAtBoundary) {
            r = _lpRequestRedeemAll(attLP);
            assertEq(juniorVault.depositEpochStart(r), boundary, "redemption targets the next epoch");
        ...
        // 2. seconds before the boundary: the open is committed; the honest keeper executes it in the next block
        _warpTo(boundary - 12);
        _keeperMark();
        ...
            uint64 id = _commit(attTrader, CfdTypes.Side.LONG, size, MARGIN_PER_OPEN, false);
            (int256 x,) = _keeperExecute(id);
            assertGt(x, 0, "the open paid a positive VPI charge");
            assertEq(_vpiAccrued(attTrader), x, "the position records the charge as its lifetime VPI");
        ...
        // 3. the boundary: the honest keeper settles the LP epoch
        _warpTo(boundary);
        o.freeAtBoundary = pool.getFreeAssets();
        bool settled = _keeperSettle();
        if (exitAtBoundary) {
            assertTrue(settled, "epoch settled");
            o.fundedInOnePass = juniorVault.claimableRedeemRequest(r, attLP) == lpShares;
            o.redeemed = _lpClaimRedeem(attLP, r);
        ...
        // 4. the close, executed by the honest keeper in the next block
        if (trade) {
            int256 accrued = _vpiAccrued(attTrader);
            o.closeDepth = pool.totalAssets(); // depth the close's VPI curve uses
            uint64 id = _commit(attTrader, CfdTypes.Side.LONG, size, 0, true);
            (int256 v, int256 pnl) = _keeperExecute(id);
            assertLe(v, 0, "the close is skew-reducing: its VPI is a rebate");
            assertGe(accrued + v, 0, "lifetime clamp: the rebate never exceeds the charge");
```

The sub-majority, settlement-hold and price-noise assertions (lines 1015-1025, 1108-1110, 1149-1156):

```solidity
    function _assertMinority(uint256 shareBps) internal {
        (Out memory ctl, Out memory atk, uint256 capital) = _minorityThreeCycles(shareBps);
        _logThree(string.concat("THREE CYCLES, ", vm.toString(shareBps / 100), "% of Junior,"), ctl, atk, capital);
        assertLt(atk.share1e6, 500_000, "the attacker never holds a majority of Junior");
        assertLt(atk.stayers, ctl.stayers, "honest Junior LPs receive less when they redeem");
        if (conf <= 10_000) {
            // attacker profit is asserted up to 1 bp of confidence
            assertGt(atk.lpSeat, 0, "attacker's realised gain over three cycles");
            assertGt(atk.lpSeat, ctl.lpSeat, "attack beats the same LP round trips without the trade");
        }
        assertGe(atk.seniorNav + 1e6, ctl.seniorNav, "Senior is not the payer");
    ...
        assertLt(lossHold, 0, "the honest LPs still lose after the hold");
        assertApproxEqAbs(lossHold, lossFree, 1e6, "the hold changes the honest LPs' loss by under 1 USDG");
        assertEq(atkHold.lpSeat - ctlHold.lpSeat, atkFree.lpSeat - ctlFree.lpSeat, "the hold claws nothing back");
    ...
        assertLt(loss[0], 0, "honest LPs lose with the price moving up");
        assertLt(loss[1], 0, "honest LPs lose with the price moving down");
        assertLt(loss[0] + loss[1], 0, "honest LPs lose on average");
        if (conf <= 10_000) {
            // attacker profit is asserted up to 1 bp of confidence
            assertGt(gain[0], 0, "attacker gains with the price moving up");
            assertGt(gain[1], 0, "attacker gains with the price moving down");
            assertGt(gain[0] + gain[1], 0, "attacker gains on average");
```

Output at 1 bp (`-vv`), excerpt:

```text
[PASS] test_1_controls_isolate_the_root_cause() (gas: 77915968)
...
  (a) trade only, LP seat stays               | trader -1276.90 | LP seat +432.88 | pair -844.02 | staying Junior LPs vs baseline +231.66
...
  (b) LP seat exits between open and close    | trader -1275.21 | LP seat +1887.73 | pair +612.52 | staying Junior LPs vs baseline -1224.88
...
  (c) LP seat exits after the close           | trader -1276.90 | LP seat +432.88 | pair -844.02 | staying Junior LPs vs baseline +222.52
...
  (d) attack + honest refill before the close | trader -1275.57 | LP seat +1887.73 | pair +612.15 | staying Junior LPs vs baseline +224.67
...
        refill LP vs its baseline: (d) -1449.18 | rebate amplification (b)-(a) 1.69 | (d)-(a) +1.32
...
    BREAK-EVEN flat book: attacker Junior 740756.22 = share 42.55% of Junior, net +0.09 | open LONG 1090800.00 | just below: -0.05
...
[PASS] test_5a_three_cycles_with_45pct_of_junior() (gas: 75261673)
...
  THREE CYCLES, 45% of Junior, at basket confidence 1.00 bps, capital tied up 843182.63, largest attacker share of Junior at a boundary 45.02%, elapsed hours 8.99
    attacker wallet change: attack +286.71 | control -24.65 | gain over control +311.37
    honest Junior LPs received on full redemption: attack 998344.21 | control 999942.08 | loss -1597.86
    Senior LP received: attack 1000091.32 | control 1000091.32 | sum of X 6683.31 | sum of rebates 6683.31 | sum of price loss 160.74
...
[PASS] test_5b_three_cycles_with_43pct_of_junior() (gas: 75257356)
...
  THREE CYCLES, 43% of Junior, at basket confidence 1.00 bps, capital tied up 779386.71, largest attacker share of Junior at a boundary 43.02%, elapsed hours 8.99
    attacker wallet change: attack +151.13 | control -23.56 | gain over control +174.69
    honest Junior LPs received on full redemption: attack 998510.20 | control 999940.98 | loss -1430.78
...
[PASS] test_6_settlement_hold_after_one_cycle_leaves_the_loss() (gas: 72376014)
Logs:
  SETTLEMENT HOLD after one cycle at basket confidence 1.00 bps, attacker share of Junior 45.00%, charge X 2228.15
    honest Junior LPs on full redemption: held-then-released attack 999393.24 | control 999925.64 | loss -532.39 || no hold: loss -532.39
    attacker cash after the cycle vs no-trade control: with hold +103.49 | no hold +103.49
...
[PASS] test_7_price_noise_one_bp_each_way() (gas: 143353637)
...
    average of both directions: attacker gain over control +311.50 | honest Junior LPs loss -1598.02

Suite result: ok. 9 passed; 0 failed; 0 skipped; finished in 582.77ms (1.96s CPU time)
```

`test_1` runs the no-trade baseline, trade without exit, exit after the close, the attack, and the attack with depth restored by an unrelated deposit from one snapshot; `test_2a` bisects the flat-book break-even share; `test_5a` and `test_5b` run three cycles at 45% and 43% of Junior against no-trade controls; `test_6` applies an LP epoch settlement hold after one cycle and releases it before everyone redeems; `test_7` moves the price 1 bp up and then down between each open and close.

