# \#99632 [SC-Medium] A trader can self-liquidate during a frozen weekend window and exit at the stale Friday mark

**Submitted on Nov 14th 2026 at 15:09:19 UTC by @Norah for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99632
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
From Friday 17:00 to Sunday 17:00 New York, and on holiday overrides, the oracle is frozen and prices cannot move. Traders may still exit, but only at the stale Friday mark and only through a voluntary close that pays a fixed 50 bps LP-owned spread plus the 4 bps execution fee, about 54 bps in total. That spread is the protocol's stated protection against stale-price exits.

Liquidations, by documented design, never pay the spread, never pay an execution fee, and never compute a fresh VPI delta. The bug is that the trader decides when their own account becomes liquidatable during the freeze. An involuntary mechanism turns into a chosen exit costing about 7.5 bps (10 bps charge, 5 bps back as self-keeper, 2.5 bps adverse shift). The trader ends up holding a free option on a closed FX market: exit at Friday's price if the weekend moved against them, hold otherwise. LPs eat the gap, every weekend and holiday.

Two independent levers put eligibility in the trader's hands.

Lever 1, withdraw free settlement. Carry is collected margin-first (`MarginClearinghouseAccountingLib.planCarryLossConsumption`: active margin before free settlement), so it erodes the pledge over time. The withdrawal guard (`CfdEngineSettlementSidecar.sol:277-305`) checks only instantaneous carry coverage, the VPI reserve, and price health. There is no carry runway. A price-healthy trader withdraws all free settlement; a second later the account has uncovered carry, and `planLiquidation` (`CfdEnginePlanLib.sol:1465-1468`) marks it liquidatable regardless of price health:

```solidity
if (!riskState.liquidatable && delta.pendingCarryAssets == delta.realizedCarryAssets) return;
```

Lever 2, carry crosses the FAD margin. During the freeze the 300 bps FAD margin applies, and pending carry is projected out of the pledge first even when free settlement exists. A position entering the freeze just above the FAD margin crosses into liquidatable territory at a time anyone can compute. `addMargin` is not gated during the freeze (`CfdEngine.sol:497+`), so the trader can postpone the crossing with small top-ups and stop once the weekend news is out.

The exit itself: `executeLiquidation` is permissionless, with no `keeper ≠ account` check (`OrderLiquidationBatchLogic.sol:706-731` passes `msg.sender` as keeper). The frozen basket returns the stale engine mark with a fixed 10 bps confidence (`PletherOracle.sol:729-747`; 2.5 bps adverse at the 2,500 multiplier). An empty Pyth payload costs nothing while frozen (`getUpdateFee` returns 0, `PletherOracle.sol:462-468`). The trader's profit is paid in full, the 10 bps charge comes from the liquidation reserve, and half of it is credited straight back to the self-keeper (`MarginClearinghouse.sol:1264-1269`). No spread, since that is voluntary-close only (`CloseAccountingLib.sol:122-123`). No fee. No fresh VPI.

## Attack path (lever 1, the PoC below)

1. Alice opens LONG 2,000 lots at 1.00 with minimal margin (1,200) in a 500k pool, keeping about 18k free settlement.
2. The basket moves to 0.95. Alice is up 10,000, comfortably above every requirement.
3. Around 150 days pass, accruing about 1,635 of carry at ~40% utilization against the 1,200 pledge. Nobody interacts, so nothing can liquidate her.
4. Friday 16:00 UTC, before the 17:00 New York close, the last live mark (0.95) is published. Saturday noon the market is frozen.
5. Alice withdraws her free settlement in two steps. The first withdrawal realizes the accrued carry margin-first, taking the pledge to zero, and passes the health check on unrealized profit. The second drains the rest.
6. Ten seconds later her carry is uncovered. Alice calls `router.executeLiquidation(alice, [])` herself: no Pyth payload, zero fee, settlement at the stale Friday mark.

## Impact

Same frozen Saturday, same snapshot, same clock:

| Exit | Alice ends with |
|---|---|
| Designed voluntary frozen close (50 bps spread + 4 bps fee) | 27,202.484599 |
| Carry-lever self-liquidation | 27,737.364674 |

The trader saves 534.88 USDG, about 28 bps of the 190k notional, per weekend exit, and the pool keeps correspondingly less. The direction is symmetric and the window recurs every weekend and holiday for every qualifying position. On an adverse weekend gap the saving is larger: the spread plus the part of the gap the stale mark ignores, roughly 44 to 48 bps of notional versus holding on a 50 bps gap at these parameters. The treasury also loses the 4 bps fee. A third-party keeper variant still bypasses the spread; the trader nets 5 bps less.
If the program reads the skipped spread as missed prospective returns rather than an LP loss, the fallback impact is 'Contract fails to deliver promised returns, but doesn't lose value' (Low).

## Why this is not a known issue

The registry documents the design: liquidations pay no frozen spread and no fresh VPI (SECURITY.md L1434-1444, ACCOUNTING_SPEC L1188) to keep liquidation liveness during closures, and the frozen spread applies to "oracle-frozen voluntary closes only" (SECURITY.md:1442). All of that assumes liquidations are involuntary. The root cause reported here, that eligibility is under the trader's control through a withdrawal guard with no carry runway and through pledge-first carry against the FAD margin with freely timed top-ups, appears nowhere in the Cecuro, pre-audit, or scan registries. "Uncovered carry makes an account liquidatable" is a normative rule in ACCOUNTING_SPEC, not a known limitation. Cecuro MED-30 (no automatic frozen stop-loss trigger) and HIGH-1 (clock alignment on live ticks) are different root causes.

## Recommendation

The primary fix covers both levers: assess the frozen-close spread and execution fee on liquidations executed while `oracleFrozen`, or defer freeze-period liquidation of accounts that were healthy at the freeze to the live reopen. Judge FAD price health during the freeze on the pledge as it stood at the freeze, and settle freeze-period carry afterwards. For lever 1, also require withdrawals to leave a minimum carry runway: pledge plus remaining free settlement covering some hours of carry.


## Proof of Concept

The PoC is a single A/B Foundry test on one snapshot of the same frozen Saturday. Branch one is the designed exit: a voluntary close while the oracle is frozen, paying the 50 bps spread and the 4 bps fee. Branch two is the attack: Alice drains her free settlement through the public `clearinghouse.withdraw`, lets ten seconds of carry accrue, and liquidates her own account through the public `router.executeLiquidation` with an empty Pyth payload and zero fee. The position is opened with the repository's Router-prank order helper. No file under `src/` is modified, and there is no fuzzing. The test exercises lever 1 end to end; lever 2 follows the same eligibility rule through the FAD-margin projection at `CfdEnginePlanLib.sol:1441-1465`.

To reproduce:

```bash
# 1. Frozen checkout (Foundry Forge 1.5.1, Solidity 0.8.35, via-IR)
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8

# 2. Save the source at the end of this report as packages/perps/test/poc/Med01_FrozenSelfLiquidation.t.sol

# 3. Run it
forge test --root packages/perps --match-path 'test/poc/Med01_FrozenSelfLiquidation.t.sol' -vv
```

The first compile covers the whole perps package under via-IR and takes several minutes. To compile only the PoC directory instead, prefix the same command with `FOUNDRY_TEST=test/poc` and add `--skip script`, which is how the numbers below were produced.

Expected output, all passing:

```

[PASS] test_PLETHER_Med01_SelfLiquidation_BeatsDesignedFrozenClose()
  Alice wealth, frozen close    : 27202.484599
  Alice wealth, self-liquidation: 27737.364674
  trader saving (attack - control): 534.880075
```

Reading the output: both exits start from the same state on the same frozen Saturday, and the self-liquidation leaves Alice 534.88 USDG richer, about 28 bps of her 190k notional, with the pool keeping correspondingly less. The test asserts the lever itself, not just the totals: after the two withdrawals, free settlement is zero and the pledge is zero (the first withdrawal's carry realization consumed it margin-first), the engine reports the oracle frozen, and `executeLiquidation` succeeds with `msg.sender` equal to the liquidated account and an empty oracle payload.

### PoC source (`packages/perps/test/poc/Med01_FrozenSelfLiquidation.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Med 01: during an oracle-frozen weekend window, a trader can make their own account
///         liquidatable on demand (withdraw all free settlement, or let accrued carry erode the
///         pledge) and exit at the stale Friday mark through permissionless self-liquidation.
///         Liquidations never pay the 50 bps frozen-close spread, the 4 bps execution fee, or a
///         fresh VPI delta, and pay the 10 bps liquidation charge's keeper half back to the
///         self-keeper - so the designed ~54 bps exit cost collapses to ~7.5 bps and LPs keep
///         bearing stale-weekend risk the spread was designed to price.
contract PocMed01FrozenSelfLiquidationTest is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);

    // Saturday 2024-08-03 12:00 UTC: inside the Fri 17:00 -> Sun 17:00 New York freeze (EDT).
    uint256 internal constant FRIDAY_MARK = 1_722_614_400; // Friday 2024-08-02 16:00 UTC, live
    uint256 internal constant SATURDAY = 1_722_686_400;
    uint256 internal constant LONG_SIZE = 200_000e18; // 2,000 lots

    // Release parameters.
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 500_000e6;
    }

    /// @dev A/B on the same frozen Saturday: the designed voluntary close (50 bps LP spread
    ///      + 4 bps fee) versus the carry-lever self-liquidation (no spread, no fee, 5 bps
    ///      self-keeper rebate on the 10 bps charge).
    function test_PLETHER_Med01_SelfLiquidation_BeatsDesignedFrozenClose() public {
        _weekendState();
        uint256 snap = vm.snapshotState();

        // --- Designed exit: voluntary close while frozen. ---
        _close(ALICE, CfdTypes.Side.LONG, LONG_SIZE, 0.95e8);
        uint256 controlWealth = _aliceWealth();
        emit log_named_decimal_uint("Alice wealth, frozen close    ", controlWealth, 6);

        // --- Attack: drain free settlement, become carry-liquidatable, self-liquidate. ---
        vm.revertToState(snap);

        uint256 w1 = _freeSettlementAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, w1 / 2);
        uint256 w2 = _freeSettlementAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, w2);
        assertEq(_freeSettlementAssets(ALICE), 0, "free settlement drained");
        assertEq(
            clearinghouse.getAccountAssetBuckets(ALICE).activePositionMarginAssets,
            0,
            "carry realization must have consumed the pledge"
        );

        // Accrual makes carry uncovered -> liquidatable regardless of price health.
        vm.warp(vm.getBlockTimestamp() + 10);
        vm.prank(ALICE);
        router.executeLiquidation{value: 0}(ALICE, new bytes[](0));

        uint256 attackWealth = _aliceWealth();
        emit log_named_decimal_uint("Alice wealth, self-liquidation", attackWealth, 6);
        emit log_named_decimal_uint("trader saving (attack - control)", attackWealth - controlWealth, 6);

        assertGt(attackWealth - controlWealth, 500e6, "self-liquidation must beat the spread+fee exit");
    }

    /// @dev Monday setup, LONG 2,000 lots at 1.00 with minimal margin, 120 days of carry
    ///      accrual (paid margin-first), then the Friday 0.95 mark before the weekend freeze.
    function _weekendState() private {
        _fundTrader(ALICE, 20_000e6);
        _open(ALICE, CfdTypes.Side.LONG, LONG_SIZE, 1_200e6, 1e8);

        // 150 days of carry accrual at ~40% utilization (~1,635 USDG) erodes the 1,200 pledge.
        vm.warp(block.timestamp + 150 days);

        // Friday 2024-08-02 16:00 UTC, still live: publish the last live mark 0.95.
        vm.warp(FRIDAY_MARK);
        vm.prank(address(router));
        engine.updateMarkPrice(0.95e8, uint64(vm.getBlockTimestamp()));

        vm.warp(SATURDAY);
    }

    function _aliceWealth() private view returns (uint256) {
        return settlementToken.balanceOf(ALICE) + clearinghouse.balanceAssets(ALICE)
            + engine.traderClaimBalanceAssets(ALICE);
    }
}
```
