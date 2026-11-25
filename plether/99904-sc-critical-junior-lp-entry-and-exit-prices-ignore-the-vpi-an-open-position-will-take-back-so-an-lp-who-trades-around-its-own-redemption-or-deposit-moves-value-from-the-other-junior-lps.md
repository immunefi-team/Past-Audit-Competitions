# \#99904 [SC-Critical] Junior LP entry and exit prices ignore the VPI an open position will take back, so an LP who trades around its own redemption or deposit moves value from the other Junior LPs

**Submitted on Nov 15th 2026 at 17:32:15 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99904
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value". The promise that fails is `SECURITY.md`'s "Exact symmetric LP NAV". The protocol as a whole loses nothing: value moves between Junior LP cohorts. The actor's side is a profitable, repeatable skim, which has the shape of a theft. We grade it Low because of the preconditions below, all stated up front.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`:
- LP pricing: `HousePool._settleLpEpoch` reconciles and then prices Junior requests off `juniorPrincipal` (`HousePool.sol:835-840`);
- the VPI flows that pricing ignores: the open's charge or rebate through pool cash (`CfdEnginePlanLib.sol:331-333`, `:719-735`); the close's lifetime clamp (`CloseAccountingLib.sol:103-118`); the liquidation's rebate clawback (`CfdEnginePlanLib.sol:1554-1571`).

**Relation to prior findings:** a new root. No prior finding, listed limitation or repository document describes a transfer between LP cohorts through outstanding VPI. The nearest is the AI scan's info item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate", which has a different root. The section "Relation to prior findings" explains each one.

## Summary

**Severity self-assessment: LOW.** Anchor: a value transfer from one Junior LP cohort to another, executed at the release risk parameters with public entry points only. Staying Junior LPs lose 698.38 USDC per cycle and the actor gains 318.18 net of every fee (Junior-only pool). In a Senior-heavy pool the figures are 1,872.48 and 872.22. Honest mitigants, which a triager will raise:

- **A dominant Junior share.** The round trip pays the actor only if it holds about 40% of the Junior tranche, or about 25% when another trader's same-side skew is already open. At a third of Junior with no other skew, the actor loses 106.31 while the staying LPs still lose 113.89.
- **Basis points per cycle.** Each position's VPI is at most `vpiFactor × maxSkew² × D / 2`, which is 8 bps of depth. The staying LPs lose about 7-19 bps of their Junior value per cycle. A cycle takes 3-4 hours (a deposit epoch, the 1-hour cooldown, about 2 hours to redemption maturity). It repeats linearly; two cycles were executed.
- **Skew room and a quiet window.** The open needs room under the 40% skew cap. If a third party reduces skew during the window, that trader collects the refund instead of the actor. The staying LPs lose in either case.

Without any attacker, an ordinary depositor who never trades captures part of a rebate that the earlier LPs funded. In our test it captures 398.68 of a 797.77 rebate when the rebate holder is liquidated at the scheduled Friday FAD shoulder.

**The mechanism in one paragraph.** VPI moves through pool cash as soon as an open executes. A skew-increasing open pays a charge `V` into the pool, and it reconciles into Junior principal. A skew-reducing open is paid a rebate `R` out of pool cash into the opener's nonwithdrawable VPI reserve. The two reversals are not symmetric, and this report only relies on what the tests demonstrate:

- **Positive `V` is a *ceiling* on a possible future refund, not a guaranteed refund.** The lifetime clamp caps a *fresh* skew-reducing voluntary close's rebate at the accrued `V` (`CloseAccountingLib.sol:103-118`); it does not create an entitlement. The refund reaches `V` only on a voluntary close whose fresh curve delta is at least `V` (sufficient skew-reduction and depth) with enough free pool cash; a liquidation computes no fresh positive rebate at all (`CfdEnginePlanLib.sol:1554-1571`), and an intervening trade, greater depth or a cash shortfall can leave the refund below `V`. The tests take exactly one such sufficient path and measure it.
- **Negative `R` has a stronger, reserve-backed recovery.** On a voluntary close or a liquidation the clawback consumes the dedicated reserve (`CfdEnginePlanLib.sol:1046-1058`, `:1554-1571`; `SECURITY.md:1436-1437`), so its return — or an equivalent withholding — is reliable.

LP pricing deliberately keeps "anticipated VPI" and the VPI reserve out of terminal `K` and symmetric NAV (`ACCOUNTING_SPEC.md:1064-1066`, `SECURITY.md:871-872`). Between the open and the reversal, Junior NAV is therefore high by `V` (or low by `R`) compared with what the LPs will own once the position is gone and that reversal has occurred. An LP request that settles in that window is mispriced along the tested paths, and the LPs on the other side of the request pay the difference. The owner of the position controls when, and on which path, the reversal happens.

## Relation to prior findings

| Item | What it says | Why this report differs |
|---|---|---|
| AI scan (info), "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" | Cutting depth before a skew-reducing close raises the rebate above the unmanipulated curve, up to the clamp | Here the refund is exactly `V`, which the unmanipulated curve also pays at unchanged depth. Our control runs the same open and close at the same depth with the redemption *after* both, and nothing moves. The deposit-side cases raise depth, and the FAD case has no trade by the entrant, yet the same transfer appears. The cause is LP pricing, not the depth denominator. |
| AI scan, "Deferred trader claims inflate open skew capacity and reduce VPI"; "Temporary pool liquidity can reduce a skew-increasing close charge" | Depth inputs to the VPI curve | Different input. Neither concerns LP entry or exit prices. |
| Pre-audit SC-M3, "Fresh negative vpiAccrued is omitted from post-open IMR" | Trader initial-margin check | Trader margin, not LP pricing. |
| Cecuro VPI items | Preview fields and receipt depth | Previews only. |
| `ACCOUNTING_SPEC.md:731-740` (carry) | "LP ownership follows collection time … This is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new LP." | This accepted limitation is written for **carry**, which is never refunded. No VPI counterpart exists. A VPI charge comes back to its payer, and the payer chooses when. |
| `SECURITY.md:1434-1444` (VPI limitations) | Liquidation computes no fresh VPI; "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" | No line names an LP-cohort transfer. The clamp line covers *rebate-first* round trips by one trader. This report is about the LP layer. |

**Uniqueness caveat (duplicate risk remains).** A captured AI scan already describes an unprivileged LP/trader withdrawing before a skew-reducing close and reclaiming accrued VPI from remaining pool cash (its stated focus is denominator/depth amplification, a different *root*). The withdraw/close sequence it describes overlaps this report's redeem path materially. We frame this as a **potentially distinct root (LP entry/exit pricing omits outstanding VPI, measured at unchanged depth) with residual duplicate risk**, not an adjudicated new root.

**Is it by design?** The documents describe the mechanism, but not this consequence. They state its purpose as the opposite of what happens here:

> anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`; the book is synchronized only to actually installed PnL pledge and same-account claim state after settlement. The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV, (`ACCOUNTING_SPEC.md:1064-1066`)

> new close-time action rebates use only pool cash free after protecting existing trader claims. Unfunded value is waived rather than claim-backed, and neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV, (`SECURITY.md:871-872`)

> new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV, (`SECURITY.md:859`, under "Exact symmetric LP NAV")

The exclusion exists so that VPI does *not* distort symmetric LP NAV. On these paths the exclusion is the distortion: `V` sits in NAV as cash while its refund is not counted, and `R` has left NAV while its certain return is not counted. The close-time waiver ("unfunded value is waived") applies only when free cash is short. It never applied in our runs. The clamp's own comment says it exists to stop "LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes" (`CloseAccountingLib.sol:112-114`). It bounds the trader's refund at `V`. It does not stop the LP layer from paying for that refund.

## Vulnerability details

**Redeem side.** M is a Junior LP and also a trader.

1. M opens a LONG at the skew cap through the Router and pays `V`. The planner computes the flow (`CfdEnginePlanLib.sol:331-333`, projected at `:719-735`) and the actual cash movement into the pool is executed in `CfdEngineSettlementSidecar.sol:442-478`; the next reconcile books it into `juniorPrincipal`.
2. M requests redemption of all its Junior shares. A role-less keeper later calls `Router.settleLpEpoch`. `HousePool._settleLpEpoch` reconciles and prices the Junior phase off that `juniorPrincipal` (`HousePool.sol:835-840`), so M is paid its share `s·V` of the charge.
3. M fully closes. The close is skew-reducing. Its fresh curve delta is negative and is clamped to at most the accrued `V`:

```solidity
// CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

   M's redemption shrank depth, which made the curve's rebate larger than `V`, so the clamp binds and the refund is exactly `V`. At unchanged depth the curve telescopes to `V` anyway (`WHITEPAPER.md:851-861`). The pool pays the refund from cash that now belongs only to the LPs who stayed.

Per cycle the staying LPs lose about `s·V` minus the close fee the planner withholds. M nets `s·V − (open fee + close fee) − carry`. With fees at 4 bps per side against up to 20 bps of notional in VPI at the cap, M profits when `s > ~0.4`. With other same-side skew `x0` already open, the break-even is about `s* = 0.16 / (x0 + x1)` (about 0.23 near the cap).

**Deposit side.** An unrelated trader B holds LONG skew. M opens a skew-reducing SHORT and is paid `R` from pool cash into its VPI reserve, so Junior NAV falls by `R`. M's Junior deposit activates at that lower NAV. M then closes; the close re-adds skew and pays `R` back into the pool. M's new shares take their part of it. M needs no skew of its own, since B's skew supplies the rebate.

**Deposit side with no trade.** A rebate holder pledged below the 3% FAD margin becomes liquidatable at an unchanged price at the Friday FAD shoulder, a time everyone knows. Liquidation claws `R` back from the reserve (`CfdEnginePlanLib.sol:1554-1571`). A Junior deposit that settled in the hourly epoch before the shoulder shares `R`, although the earlier LPs funded it.

**The LP request delay does not help.** Redemptions are priced at settlement (ERC-7540 async), not at the request. The test waits the 1-hour cooldown and about 2 hours to maturity, and settles through the public Router. Nothing removes `V` from NAV before M's own close.

## Impact

- **Who loses.** The Junior LPs on the other side of the actor's LP request, pro rata. That is the stayers for a redemption and the incumbents for a deposit. Senior is untouched, because VPI revenue and its reversal both land in Junior principal.
- **How much (executed, release risk parameters, flat price, net of every fee and carry):**

| Case | Staying / incumbent Junior LPs | Actor |
|---|---|---|
| Redeem side, others 1M, M 1.5M Junior (s = 0.6), 950k notional | −698.38 | +318.18 |
| Same, two back-to-back cycles | −1,397.36 | +636.96 |
| Redeem side, Senior 4M added, 2.5M notional (cap on total depth) | −1,872.48 | +872.22 |
| Redeem side, s = 1/3, no other skew | −113.89 | −106.31 |
| Redeem side, s = 1/4, another trader's LONG 400k open | −60.87 | +8.67 |
| Deposit side, B LONG 390k, M SHORT 150k, M deposits 1M | −235.42 | +115.22 |
| Deposit side, no trade: entrant at the FAD liquidation of a 797.77 rebate | −398.68 (funded by earlier LPs) | +398.68 |

- **Controls.** The same open and close, finished *before* the LP request, moves nothing. On the redeem side the staying LPs end +152.61 (they keep the withheld close fee); on the deposit side +0.22. Without a rebate holder, the FAD entrant's value does not jump.
- **Conservation.** The value moves between cohorts and to the treasury (open fees). The pool's total is conserved, which is why we select the Low line.
- **Capital and repetition.** The actor needs Junior capital of about `s/(1−s)` times the other Junior holders, plus trading margin (20 bps initial at release). A cycle takes 3-4 hours. Repeating it is linear (executed twice). We have not executed a sustained multi-day run.


## Recommendation

Price LP entry and exit off the NAV the LPs will own once open positions' VPI has reversed. Two shapes, both sketches that we have not run against the maintainers' suite:

1. **Defer positive VPI and count the rebate receivable.** Keep each open position's positive `vpiAccrued` in a deferred bucket outside tranche principal, and recognize it as Junior revenue only when the position closes or is liquidated. The clamp can refund at most that amount, so recognizing the unrefunded remainder at exit is exact. Count each position's `max(-vpiAccrued, 0)` reserve as a pool receivable in LP pricing; the clamp and the liquidation clawback make its return certain.

```solidity
// sketch: in the terminal snapshot that LP pricing consumes
lpPricingAssets = juniorPrincipal
    - deferredPositiveVpiAssets          // sum over open positions of max(vpiAccrued, 0)
    + reservedNegativeVpiAssets;         // sum over open positions of max(-vpiAccrued, 0)
```

2. **Or restrict the round trip at the account.** Settle an account's own Junior redemption or deposit only while that account has no open position with nonzero `vpiAccrued`. This stops the actor paths, but it does not stop the no-trade FAD case, so shape 1 is the complete fix.

`ACCOUNTING_SPEC.md:1064-1066` and `SECURITY.md:871-872` should then say what LP pricing does with outstanding VPI. The property tests above are regression tests for either shape.


## Link to Proof of Concept

https://gist.github.com/Richter2552/af350526c1e1d74f7d519f39f73afdf8

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `BasePerpTest` fixture.

**Fidelity.** The ten release `RiskParams` (`DEPLOYMENT.md:128-150`: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, maintenance 10 bps, initial 20 bps, FAD 300 bps) and `executionFeeBps 4` are asserted on-chain in `test_Config_ReleaseRiskParamsOnChain`. After `setUp`, every actor has no role and uses public entry points only:
- trades: the production V3 `Router.commitOrder`, then a role-less keeper's `Router.executeOrder`;
- LP requests: the ERC-7540 `TrancheVault` `requestDeposit`/`claimDeposit` and `requestRedeem`/`claimRedeem`;
- every LP epoch settlement and mark refresh: the permissionless `Router.settleLpEpoch` / `Router.updateMarkPrice`;
- the liquidation: `Router.executeLiquidation`.

No protocol contract is impersonated after `setUp`, and no storage is written. Disclosed deviations:
- **Oracle.** The maintainers' `MockPyth` basket supplies prices at 1.00 (the program allows mocking Pyth).
- **Token.** The settlement token is the fixture's mintable mock.
- **Fixture.** It funds the incumbent Junior LP (1,000,000) in `setUp`. Its seed and senior-capacity values are the fixture's. The Senior-heavy case uses the fixture's `_fundSenior` before the snapshot, identically in both worlds.

Each comparison runs two worlds from one snapshot, identical except that M also trades in world X. Price is flat throughout, so trader price PnL is zero and only VPI, fees and carry differ.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card6029S1VpiNavCohortTransfer.t.sol packages/perps/test/perps/spec/lp/
cd packages/perps

forge test --match-contract '^Card6029S1VpiNavCohortTransferTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card6029S1VpiNavCohortTransferTest$' --match-test '^test_(Config|Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Result: 11 tests in all — 8 pass (4 characterizations, 3 controls, 1 config) and the 3 property tests fail (forge 1.8.1, solc 0.8.35, via-IR). Excerpt (USDC amounts in 1e6 units):

```console
[FAIL: PROPERTY: staying Junior LPs must not fund the refund of VPI an exiting LP was paid out: -698379652 < 0] test_Property_RedeemSide_StayingLpsDoNotFundRefundOfVpiHeldInNav()
  REDEEM SIDE, s = 0.6
    VPI V paid by M's opens (sum) 1803557154
    M LP redemption paid: X, Y 1501081685174 1499999983559
    M trader custody: X, Y 399236478022 400000000000
    staying Junior LPs' value: X, Y 1000301591110 1000999970762
    treasury custody X (open + close fees) 380000000
    staying Junior LPs X - Y (neg = loss)
  -698379652
    M total X - Y (pos = gain, net of all fees and carry)
  318179637

[PASS] test_Control_RedeemSide_RoundTripBeforeRedeemTransfersNothing()
    staying Junior LPs X - Y (neg = loss)
  152611680
    M total X - Y (pos = gain, net of all fees and carry)
  -532811697

[PASS] test_Characterize_RedeemSide_TwoCycles()
  -1397362549   (staying LPs)
  636962533     (M)

[PASS] test_Characterize_RedeemSide_SeniorHeavyPool()
    VPI V paid by M's opens (sum) 4806213472
  -1872478094   (staying LPs)
  872217194     (M)

[PASS] test_Characterize_RedeemSide_ThirdShareNoOtherSkew()
  -113889008    (staying LPs)
  -106311007    (M)

[PASS] test_Characterize_RedeemSide_QuarterShareWithOtherSkew()
  -60867075     (staying LPs)
  8667060       (M)

[FAIL: PROPERTY: incumbent Junior LPs must not fund the rebate a depositor returns after entry: -235419842 < 0] test_Property_DepositSide_IncumbentsNotDilutedByRebateHeldOutsideNav()
    rebate R paid to M's SHORT open 471199968
    incumbent Junior LPs X - Y (neg = loss)
  -235419842
    M total X - Y (pos = gain, net of all fees and carry)
  115219839

[PASS] test_Control_DepositSide_RoundTripBeforeDepositTransfersNothing()
    incumbent Junior LPs X - Y (neg = loss)
  221569

[FAIL: PROPERTY: an entrant must not capture a rebate that left pool cash before it entered: 548466259 > 149784416] test_Property_OrganicEntrant_DoesNotCaptureRebateClawedBackAtFad()
    CAROL rebate R 797767524
    Junior NAV jump at the liquidation 1097486082
    NEWLP share of Junior (1e18) 499750219359578841
    NEWLP gain at the liquidation / allowed (ordinary revenue) 548466259 149784415
    NEWLP gain from R 398681844

[PASS] test_Control_OrganicEntrant_NoRebateHolderNoJump()
[PASS] test_Config_ReleaseRiskParamsOnChain()
```

### What the tests show

- **Redeem property (fails).** The staying Junior LPs end 698.38 USDC lower when M holds its own refundable `V` across its exit. M ends 318.18 higher after paying both 4-bps fees (380 each: the open fee goes to the treasury, and the close fee is withheld from the refund and kept by the LPs) and carry. M's redemption is paid 1,081.70 more (≈ 0.6 × `V`), and its trader account gets the full `V` back.
- **Redeem control (passes).** The same round trip done before the redemption request moves nothing: the stayers end +152.61, and M only pays fees.
- **Characterization.** The loss is linear in cycles. It is larger in a Senior-heavy pool (the cap is 40% of total depth, while VPI lands only in Junior). It persists below the actor's break-even. Other same-side skew lowers the break-even to about a quarter of Junior.
- **Deposit property (fails) and control (passes).** A rebate held across the deposit moves 235.42 from incumbents (≈ 0.5 × `R`). Returned before the deposit, it moves nothing.
- **No-trade entrant (fails) and control (passes).** NEWLP holds ~50% of Junior. At the liquidation it gains 548.47. Its share of the liquidation's ordinary revenue (LP half of the charge plus carry) is 149.78. The other 398.68 is its share of `R`.

### Test file: `packages/perps/test/perps/spec/lp/Card6029S1VpiNavCohortTransfer.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @title Junior LP entry and exit prices omit the VPI that open positions will still reclaim or return, so an account
///        that trades around its own LP request moves value between Junior LP cohorts
/// @notice Authorized audit-competition PoC for plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8.
/// @dev Root. A skew-changing open moves VPI through pool cash at once: a positive charge V is paid INTO the pool and
///      reconciles into Junior principal; a rebate R is paid OUT of the pool into the opener's nonwithdrawable VPI
///      reserve. Both are undone later by the same position: the lifetime clamp refunds up to V on a skew-reducing
///      close, and the close or liquidation claws R back from the reserve. LP pricing deliberately keeps "anticipated
///      VPI" and the VPI reserve out of terminal K and symmetric NAV (ACCOUNTING_SPEC.md:1064-1066, SECURITY.md:871-872).
///      So between the open and the reversal, Junior NAV is high by V (or low by R) relative to what the LPs will own
///      once the position is gone. An LP request that settles inside that window is priced off the wrong NAV, and the
///      LP cohort on the other side of the request pays the difference.
///      Two manifestations, one root, one fix:
///      - REDEEM side: M (a large Junior LP) opens skew-increasing and pays V, redeems Junior at a NAV that includes V,
///        then closes and is refunded V out of the LPs who stayed.
///      - DEPOSIT side: with third-party skew open, M opens skew-reducing and is paid R, deposits Junior at a NAV that
///        is short by R, then closes; the close pays R back into the pool and M's new shares take part of it.
///      Every comparison is two worlds from one snapshot, identical except that M also trades in world X. Price is
///      flat at 1.00 throughout, so trader price PnL is zero and only VPI, fees and carry differ.
///      Fidelity: the ten release RiskParams (DEPLOYMENT.md:128-150), asserted on-chain in test_Config. After setUp,
///      every actor is unprivileged and acts through public entry points only: trades through the production V3
///      Router.commitOrder plus a role-less keeper's Router.executeOrder; LP deposits and redemptions through the
///      ERC-7540 TrancheVault requestDeposit/claimDeposit and requestRedeem/claimRedeem; every LP epoch settlement
///      and mark refresh through the permissionless Router.settleLpEpoch / Router.updateMarkPrice. No protocol
///      contract is impersonated after setUp and no storage is written. Prices come from the maintainers' MockPyth
///      basket (the program allows mocking Pyth); the settlement token is the fixture's mintable mock.
contract Card6029S1VpiNavCohortTransferTest is BasePerpTest {
    address internal constant M = address(0xA77AC4); // a Junior LP that also trades
    address internal constant B = address(0xB0B0B0); // an unrelated trader whose skew is already open
    address internal constant KEEPER = address(0x6EE9E9); // no role
    uint256 internal constant P0 = 1e8;

    uint256 private _nonce;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    // ============================================================================================================
    // public-path helpers
    // ============================================================================================================

    function _commit(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = keccak256(abi.encode("card6029", who, ++_nonce));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        // LONG open / SHORT close accept a price >= target; SHORT open / LONG close accept <= target.
        bool wantsHigh = (side == CfdTypes.Side.LONG) != isClose;
        r.targetPrice = wantsHigh ? 1 : type(uint256).max;
        r.isClose = isClose;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = isClose ? 0 : type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
        vm.warp(vm.getBlockTimestamp() + 2);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(who);
        id = router.commitOrder(r);
    }

    function _trade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        uint64 id = _commit(who, side, size, margin, isClose);
        bytes[] memory data = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1 ether}(id, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed");
    }

    function _closeAll(
        address who,
        CfdTypes.Side side
    ) internal {
        (uint256 sz,) = _pos(who);
        _trade(who, side, sz, 0, true);
        (uint256 left,) = _pos(who);
        assertEq(left, 0, "ghost: fully closed");
    }

    function _publicSettle() internal {
        bytes[] memory data = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: 1 ether}(data);
    }

    function _publicRefresh() internal {
        bytes[] memory data = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 1 ether}(data);
    }

    /// Public ERC-7540 Junior deposit: request, wait for the epoch, permissionless settle, claim.
    function _publicDeposit(
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 reqId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(reqId);
        if (block.timestamp < activation) vm.warp(activation);
        vm.roll(vm.getBlockNumber() + 1);
        _publicSettle();
        uint256 claimable = juniorVault.claimableDepositRequest(reqId, lp);
        assertEq(claimable, amount, "ghost: deposit activated");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(reqId, claimable, lp, lp);
    }

    /// Public ERC-7540 Junior redemption of all of `lp`'s shares: request, wait for maturity, permissionless settle,
    /// claim. Returns the assets paid.
    function _publicRedeemAll(
        address lp
    ) internal returns (uint256 paid, uint256 sharesLeft) {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 reqId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        vm.roll(vm.getBlockNumber() + 1);
        _publicSettle();
        uint256 claimable = juniorVault.claimableRedeemRequest(reqId, lp);
        if (claimable > 0) {
            vm.prank(lp);
            paid = juniorVault.claimRedeem(reqId, claimable, lp, lp);
        }
        sharesLeft = shares - claimable;
    }

    function _pos(
        address a
    ) internal view returns (uint256 size, int256 vpiAccrued) {
        (size,,,,,, vpiAccrued) = engine.positions(a);
    }

    /// Value of the Junior shares held by the fixture's incumbent LP (address(this), 1,000,000 USDG).
    /// TrancheVault.totalAssets() reads HousePool.getPendingTrancheState(), the pending-reconcile preview.
    function _othersValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    // ============================================================================================================
    // REDEEM side: V held in NAV across M's own exit
    // ============================================================================================================

    uint8 internal constant NONE = 0; // M deposits and redeems only
    uint8 internal constant ACROSS = 1; // M opens before its redemption request and closes after the claim
    uint8 internal constant BEFORE = 2; // control: M opens and closes before its redemption request

    struct RedeemWorld {
        uint256 mLpPaid;
        uint256 mSharesLeft;
        uint256 mCustody;
        uint256 othersValue;
        uint256 vpiPaid;
        uint256 treasury;
    }

    function _redeemCycle(
        uint8 mode,
        uint256 mDeposit,
        uint256 mLots
    ) internal returns (RedeemWorld memory w) {
        _publicDeposit(M, mDeposit);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 5); // deposit cooldown before a redemption request
        if (mode != NONE) {
            _trade(M, CfdTypes.Side.LONG, mLots * CfdTypes.SIZE_QUANTUM, 100_000e6, false);
            (, int256 v) = _pos(M);
            assertGt(v, 0, "ghost: M's open paid a positive VPI charge");
            w.vpiPaid = uint256(v);
        }
        if (mode == BEFORE) _closeAll(M, CfdTypes.Side.LONG);
        (w.mLpPaid, w.mSharesLeft) = _publicRedeemAll(M);
        if (mode == ACROSS) _closeAll(M, CfdTypes.Side.LONG);
        w.othersValue = _othersValue();
        w.mCustody = clearinghouse.balanceAssets(M);
        w.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _redeemWorld(
        uint8 mode,
        uint256 mDeposit,
        uint256 mLots,
        uint256 cycles
    ) internal returns (RedeemWorld memory t) {
        for (uint256 i; i < cycles; i++) {
            RedeemWorld memory w = _redeemCycle(mode, mDeposit, mLots);
            t.mLpPaid += w.mLpPaid;
            t.mSharesLeft += w.mSharesLeft;
            t.vpiPaid += w.vpiPaid;
            t.othersValue = w.othersValue;
            t.mCustody = w.mCustody;
            t.treasury = w.treasury;
        }
    }

    /// Shared prefix (both worlds): KEEPER gas, M's trading collateral, and optionally an unrelated same-side LONG.
    function _redeemPrefix(
        uint256 bLots
    ) internal {
        vm.deal(KEEPER, 100 ether);
        _fundTrader(M, 400_000e6);
        if (bLots > 0) {
            _fundTrader(B, 200_000e6);
            _trade(B, CfdTypes.Side.LONG, bLots * CfdTypes.SIZE_QUANTUM, 100_000e6, false);
        }
    }

    function _redeemCompare(
        string memory tag,
        uint8 mode,
        uint256 mDeposit,
        uint256 bLots,
        uint256 mLots,
        uint256 cycles
    ) internal returns (int256 othersDelta, int256 mDelta, RedeemWorld memory x, RedeemWorld memory y) {
        _redeemPrefix(bLots);
        uint256 s = vm.snapshotState();
        y = _redeemWorld(NONE, mDeposit, mLots, cycles);
        vm.revertToState(s);
        x = _redeemWorld(mode, mDeposit, mLots, cycles);
        vm.revertToState(s);
        assertEq(x.mSharesLeft, 0, "ghost: X redemption fully funded");
        assertEq(y.mSharesLeft, 0, "ghost: Y redemption fully funded");
        othersDelta = int256(x.othersValue) - int256(y.othersValue);
        mDelta = int256(x.mLpPaid + x.mCustody) - int256(y.mLpPaid + y.mCustody);
        console2.log(tag);
        console2.log("  M Junior deposit / cycles", mDeposit, cycles);
        console2.log("  B lots (other same-side LONG) / M lots", bLots, mLots);
        console2.log("  VPI V paid by M's opens (sum)", x.vpiPaid);
        console2.log("  M LP redemption paid: X, Y", x.mLpPaid, y.mLpPaid);
        console2.log("  M trader custody: X, Y", x.mCustody, y.mCustody);
        console2.log("  staying Junior LPs' value: X, Y", x.othersValue, y.othersValue);
        console2.log("  treasury custody X (open + close fees)", x.treasury);
        console2.log("  staying Junior LPs X - Y (neg = loss)");
        console2.logInt(othersDelta);
        console2.log("  M total X - Y (pos = gain, net of all fees and carry)");
        console2.logInt(mDelta);
    }

    /// PROPERTY (fails on the pinned code). The LPs who stay through an epoch must not end worse off because another
    /// LP made a trade round trip whose net VPI is zero (flat price, the clamp refunds exactly V). M holds 60% of
    /// Junior and opens 950k LONG (the 40% skew cap on ~2.5M depth).
    function test_Property_RedeemSide_StayingLpsDoNotFundRefundOfVpiHeldInNav() public {
        (int256 othersDelta, int256 mDelta,,) = _redeemCompare("REDEEM SIDE, s = 0.6", ACROSS, 1_500_000e6, 0, 9500, 1);
        assertGt(mDelta, 0, "ghost: M profits net of fees");
        assertGe(othersDelta, 0, "PROPERTY: staying Junior LPs must not fund the refund of VPI an exiting LP was paid out");
    }

    /// CONTROL (passes). The same open and close, finished BEFORE M's redemption request: no VPI is held in NAV across
    /// the exit, and the staying LPs do not lose (they keep the withheld close fee and carry).
    function test_Control_RedeemSide_RoundTripBeforeRedeemTransfersNothing() public {
        (int256 othersDelta, int256 mDelta,,) =
            _redeemCompare("CONTROL: round trip BEFORE the redemption request", BEFORE, 1_500_000e6, 0, 9500, 1);
        assertGe(othersDelta, 0, "control: staying LPs do not lose");
        assertLt(mDelta, 0, "control: M only pays fees");
    }

    /// Repeatable: two back-to-back cycles, each with a fresh public deposit by M. About twice one cycle.
    function test_Characterize_RedeemSide_TwoCycles() public {
        (int256 othersDelta, int256 mDelta,,) =
            _redeemCompare("REDEEM SIDE, two cycles", ACROSS, 1_500_000e6, 0, 9500, 2);
        assertLt(othersDelta, 0, "staying LPs lose across cycles");
        assertGt(mDelta, 0, "M profits across cycles");
    }

    /// Senior-heavy pool: the skew cap is 40% of TOTAL depth while VPI revenue and its refund land only in Junior.
    function test_Characterize_RedeemSide_SeniorHeavyPool() public {
        _fundSenior(address(0x5E5E), 4_000_000e6); // fixture helper, before the snapshot, identical in both worlds
        (int256 othersDelta, int256 mDelta,,) =
            _redeemCompare("REDEEM SIDE, Senior 4M, s = 0.6", ACROSS, 1_500_000e6, 0, 25_000, 1);
        assertLt(othersDelta, 0, "staying Junior LPs lose");
        assertGt(mDelta, 0, "M profits");
    }

    /// Below break-even without other skew (s = 1/3): M loses, and the staying LPs still lose.
    function test_Characterize_RedeemSide_ThirdShareNoOtherSkew() public {
        (int256 othersDelta, int256 mDelta,,) = _redeemCompare("REDEEM SIDE, s = 1/3", ACROSS, 500_000e6, 0, 5500, 1);
        assertLt(othersDelta, 0, "staying LPs still lose");
        assertLt(mDelta, 0, "M is below break-even");
    }

    /// With another trader's same-side skew already open (B LONG 400k), the marginal VPI per notional is higher and
    /// the break-even share falls: M profits at s = 1/4.
    function test_Characterize_RedeemSide_QuarterShareWithOtherSkew() public {
        (int256 othersDelta, int256 mDelta,,) =
            _redeemCompare("REDEEM SIDE, s = 1/4, B LONG 400k open", ACROSS, 333_333e6, 4000, 1300, 1);
        assertLt(othersDelta, 0, "staying LPs lose");
        assertGt(mDelta, 0, "M profits at s = 1/4 with existing same-side skew");
    }

    // ============================================================================================================
    // DEPOSIT side: R held outside NAV across M's own entry
    // ============================================================================================================

    struct DepositWorld {
        uint256 mLpValue;
        uint256 mCustody;
        uint256 othersValue;
        uint256 rebate;
    }

    function _depositWorld(
        uint8 mode,
        uint256 mDeposit,
        uint256 mLots
    ) internal returns (DepositWorld memory w) {
        if (mode != NONE) {
            _trade(M, CfdTypes.Side.SHORT, mLots * CfdTypes.SIZE_QUANTUM, 50_000e6, false);
            (, int256 v) = _pos(M);
            assertLt(v, 0, "ghost: M's skew-reducing open received a rebate");
            w.rebate = uint256(-v);
        }
        if (mode == BEFORE) _closeAll(M, CfdTypes.Side.SHORT);
        _publicDeposit(M, mDeposit);
        if (mode == ACROSS) {
            _closeAll(M, CfdTypes.Side.SHORT);
        } else {
            // same clock as the trade world
            vm.warp(vm.getBlockTimestamp() + 2);
            vm.roll(vm.getBlockNumber() + 1);
        }
        _publicRefresh();
        w.mLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(M));
        w.mCustody = clearinghouse.balanceAssets(M);
        w.othersValue = _othersValue();
    }

    function _depositCompare(
        string memory tag,
        uint8 mode
    ) internal returns (int256 othersDelta, int256 mDelta, DepositWorld memory x) {
        vm.deal(KEEPER, 100 ether);
        // B: an unrelated LONG near the 40% skew cap of ~1M depth (both worlds).
        _fundTrader(B, 100_000e6);
        _trade(B, CfdTypes.Side.LONG, 3900 * CfdTypes.SIZE_QUANTUM, 50_000e6, false);
        _fundTrader(M, 200_000e6);
        uint256 s = vm.snapshotState();
        DepositWorld memory y = _depositWorld(NONE, 1_000_000e6, 1500);
        vm.revertToState(s);
        x = _depositWorld(mode, 1_000_000e6, 1500);
        vm.revertToState(s);
        othersDelta = int256(x.othersValue) - int256(y.othersValue);
        mDelta = int256(x.mLpValue + x.mCustody) - int256(y.mLpValue + y.mCustody);
        console2.log(tag);
        console2.log("  rebate R paid to M's SHORT open", x.rebate);
        console2.log("  incumbent Junior LPs' value: X, Y", x.othersValue, y.othersValue);
        console2.log("  M Junior value: X, Y", x.mLpValue, y.mLpValue);
        console2.log("  M trader custody: X, Y", x.mCustody, y.mCustody);
        console2.log("  incumbent Junior LPs X - Y (neg = loss)");
        console2.logInt(othersDelta);
        console2.log("  M total X - Y (pos = gain, net of all fees and carry)");
        console2.logInt(mDelta);
    }

    /// PROPERTY (fails on the pinned code). Incumbent Junior LPs must not lose because a depositor held a VPI rebate
    /// across its deposit activation and returned it after (flat price, net-zero lifetime VPI).
    function test_Property_DepositSide_IncumbentsNotDilutedByRebateHeldOutsideNav() public {
        (int256 othersDelta, int256 mDelta,) = _depositCompare("DEPOSIT SIDE, B LONG 390k open, M SHORT 150k", ACROSS);
        assertGt(mDelta, 0, "ghost: M profits net of fees");
        assertGe(othersDelta, 0, "PROPERTY: incumbent Junior LPs must not fund the rebate a depositor returns after entry");
    }

    /// CONTROL (passes). The same SHORT open and close, finished BEFORE M's deposit request: no dilution.
    function test_Control_DepositSide_RoundTripBeforeDepositTransfersNothing() public {
        (int256 othersDelta, int256 mDelta,) =
            _depositCompare("CONTROL: rebate round trip BEFORE the deposit", BEFORE);
        assertGe(othersDelta, 0, "control: incumbents do not lose");
        assertLt(mDelta, 0, "control: M only pays fees");
    }

    // ============================================================================================================
    // DEPOSIT side, no trade by the entrant: an organic rebate R returns at the Friday FAD liquidation
    // ============================================================================================================

    address internal constant ALICE = address(0xA11CE); // LONG skew at the cap, 5% pledge (survives FAD)
    address internal constant CAROL = address(0xCA201); // skew-reducing SHORT, ~0.5% pledge (below the 3% FAD margin)
    address internal constant NEWLP = address(0x4E3); // an ordinary Junior depositor that never trades
    /// 2024-03-08 21:30:00 UTC, a Friday: the FAD live close-only shoulder begins.
    uint256 internal constant FAD_START = 1_709_933_400;
    uint256 internal constant REQUEST_TS = FAD_START - 90 minutes; // 20:00 UTC, targets the 21:00 epoch

    struct FadWorld {
        uint256 carolRebate;
        uint256 newShares;
        uint256 newValueAtEntry;
        uint256 newValueAfterFad;
        uint256 juniorNavBeforeLiq;
        uint256 juniorNavAfterLiq;
    }

    function _fadWorld(
        bool withCarol
    ) internal returns (FadWorld memory w) {
        vm.deal(KEEPER, 100 ether);
        _fundTrader(ALICE, 30_000e6);
        _fundTrader(CAROL, 3_000e6);
        assertFalse(engine.isFadWindow(), "ghost: the book is built outside FAD");
        _trade(ALICE, CfdTypes.Side.LONG, 4000 * CfdTypes.SIZE_QUANTUM, 20_000e6, false);
        if (withCarol) {
            _trade(CAROL, CfdTypes.Side.SHORT, 4000 * CfdTypes.SIZE_QUANTUM, 2_000e6, false);
            (, int256 v) = _pos(CAROL);
            assertLt(v, 0, "ghost: CAROL's skew-reducing open received a rebate");
            w.carolRebate = uint256(-v);
        }
        // Friday 20:00 UTC: NEWLP requests a Junior deposit; it settles at the 21:00 epoch, before the FAD shoulder.
        vm.warp(REQUEST_TS);
        vm.roll(vm.getBlockNumber() + 1);
        _publicRefresh();
        settlementToken.mint(NEWLP, 1_000_000e6);
        vm.startPrank(NEWLP);
        settlementToken.approve(address(juniorVault), 1_000_000e6);
        uint256 reqId = juniorVault.requestDeposit(1_000_000e6, NEWLP, NEWLP);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(reqId);
        assertLt(activation, FAD_START, "ghost: the deposit epoch settles before the FAD shoulder");
        vm.warp(activation + 10);
        vm.roll(vm.getBlockNumber() + 1);
        _publicSettle();
        uint256 claimable = juniorVault.claimableDepositRequest(reqId, NEWLP);
        assertEq(claimable, 1_000_000e6, "ghost: NEWLP deposit activated");
        vm.prank(NEWLP);
        w.newShares = juniorVault.claimDeposit(reqId, claimable, NEWLP, NEWLP);
        w.newValueAtEntry = juniorVault.convertToAssets(w.newShares);
        // Friday 21:35 UTC: a role-less keeper liquidates CAROL at the unchanged price (FAD margin 300 bps).
        vm.warp(FAD_START + 5 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        _publicRefresh();
        assertTrue(engine.isFadWindow(), "ghost: FAD window");
        w.juniorNavBeforeLiq = juniorVault.totalAssets();
        if (withCarol) {
            bytes[] memory data = _mockPythUpdateData(P0);
            vm.prank(KEEPER);
            router.executeLiquidation{value: 1 ether}(CAROL, data);
            (uint256 carolSize,) = _pos(CAROL);
            assertEq(carolSize, 0, "ghost: CAROL liquidated");
        }
        w.juniorNavAfterLiq = juniorVault.totalAssets();
        w.newValueAfterFad = juniorVault.convertToAssets(w.newShares);
    }

    /// PROPERTY (fails on the pinned code). A Junior entrant that never trades may share the liquidation's ordinary
    /// revenue (the LP half of the charge, carry), but not the rebate R that left pool cash before it entered.
    function test_Property_OrganicEntrant_DoesNotCaptureRebateClawedBackAtFad() public {
        FadWorld memory w = _fadWorld(true);
        uint256 navJump = w.juniorNavAfterLiq - w.juniorNavBeforeLiq;
        uint256 gain = w.newValueAfterFad - w.newValueAtEntry;
        uint256 share1e18 = (w.newValueAfterFad * 1e18) / w.juniorNavAfterLiq;
        uint256 allowed = navJump > w.carolRebate ? ((navJump - w.carolRebate) * share1e18) / 1e18 : 0;
        console2.log("ORGANIC ENTRANT, FAD liquidation of a rebate holder");
        console2.log("  CAROL rebate R", w.carolRebate);
        console2.log("  Junior NAV jump at the liquidation", navJump);
        console2.log("  NEWLP share of Junior (1e18)", share1e18);
        console2.log("  NEWLP gain at the liquidation / allowed (ordinary revenue)", gain, allowed);
        console2.log("  NEWLP gain from R", gain > allowed ? gain - allowed : 0);
        assertGe(navJump, w.carolRebate, "ghost: the clawed-back rebate re-enters Junior NAV");
        assertLe(gain, allowed + 1, "PROPERTY: an entrant must not capture a rebate that left pool cash before it entered");
    }

    /// CONTROL (passes). No rebate holder: the entrant's value does not jump at the FAD shoulder.
    function test_Control_OrganicEntrant_NoRebateHolderNoJump() public {
        FadWorld memory w = _fadWorld(false);
        console2.log("CONTROL: no rebate holder; NEWLP value at entry / after FAD", w.newValueAtEntry, w.newValueAfterFad);
        assertLe(w.newValueAfterFad, w.newValueAtEntry + 1, "control: no NAV jump without a rebate holder");
    }

    // ============================================================================================================
    // CONFIG
    // ============================================================================================================

    function test_Config_ReleaseRiskParamsOnChain() public view {
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
    }
}
```

