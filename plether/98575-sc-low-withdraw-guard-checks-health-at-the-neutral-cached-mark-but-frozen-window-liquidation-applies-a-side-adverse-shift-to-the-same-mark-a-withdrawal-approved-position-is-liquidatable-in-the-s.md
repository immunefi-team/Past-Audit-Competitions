# \#98575 [SC-Low] Withdraw guard checks health at the neutral cached mark, but frozen-window liquidation applies a side-adverse shift to the same mark — a withdrawal-approved position is liquidatable in the same block

**Submitted on Nov 8th 2026 at 19:06:43 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98575
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

During the oracle-frozen weekend window, `validateWithdraw` approves a withdrawal after checking position health at the **neutral** `engine.lastMarkPrice()`. Liquidation reads the **same** cached mark, but `PletherOracle` attaches a deterministic confidence to it and shifts the price **against** the position side (~2–2.5 bp). Both checks use the same threshold: `max(IM, FAD)` = 3% during FAD/frozen windows. Because there is no buffer between them, any position whose equity is within ~2.5 bp of notional above 3% gets a "safe" verdict from the guard and is liquidatable by any keeper in the same block, at the same mark and timestamp.

### Vulnerability Details

**Withdraw guard: neutral price, no shift**

```solidity
// CfdEngineSettlementSidecar.sol:381-385  (_liveMark)
@>  price = engine.lastMarkPrice();

// CfdEngineSettlementSidecar.sol:301-303
    uint256 reachableAssets = clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
    if (_withdrawPositionLiquidatable(engine, pos, account, price, reachableAssets)) revert ...;

// CfdEngineSettlementSidecar.sol:326-331  -> max(initMarginBps, isFadWindow ? fadMarginBps : maintMarginBps)
```

**Liquidation: same mark, then side-adverse shift**

```solidity
// PletherOracle.sol:729-743  (_frozenBasket)
@>  basket.price = engine.lastMarkPrice();                                    // same input
@>  basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;  // 10 bps default (:84)

// PletherOracle.sol:342  (updateLiquidationPrice)
    snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));

// PletherOracle.sol:965, :972
@>  shift = confidence * adverseConfidenceMultiplierBps / 10_000;             // 2000 default (:95) -> 2 bp
    return side == LONG ? price + shift : price - shift;
```

Liquidation then checks health against `fadMarginBps` during FAD (`CfdEnginePlanLib.sol:1450`), using the same collateral `pos.margin + traderClaim` (`:1461`). This is the same threshold and the same collateral as the guard. Only the price differs.

The keeper does not control any of this. In frozen mode `pythUpdateData` is ignored (`PletherOracle.sol:754-755`), and the confidence parameters are protocol config. No trusted-role mistake is required.

The guard's own NatSpec (`CfdEngine.sol:722`) says it "Validates that a clearinghouse withdrawal leaves an open position sufficiently collateralized". `SECURITY.md:425` says the protocol uses "side-adverse pricing for execution, equity checks, and liquidation". The withdraw equity check does not apply this shift.

### Impact Details

- Invariant broken: if a withdrawal succeeds while `lots > 0`, an immediate liquidation at the same mark and timestamp should not be possible. The PoC breaks this.
- PoC numbers (100k-token SHORT, notional ≈ 99.6k USDG, release risk params): the guard approves a **6,500 USDG** withdrawal, and the position is then liquidated in the same block at the unchanged mark `0.99649444`. The keeper bounty is **49.81 USDG**.
- Affected band: equity in `(3%, 3% + ~2–2.5 bp]` of notional, every frozen window (every weekend).
- **Scope limitation (measured):** free settlement is not counted by either check, and withdraw cannot touch locked position margin (`MarginClearinghouse.sol:470-472`, `:1726`). The withdrawal therefore does not cause or worsen the liquidation: the same position was liquidatable before the withdrawal. No LP bad debt arises, because equity is still ≈ 3% at liquidation.
- The concrete harm is that users and integrators who treat `checkWithdraw` success as "position is safe" get a false signal during every frozen window.

### Mitigation

In `_requireWithdrawPositionHealthy`, apply the same side-adverse shift when `isOracleFrozen()`. It can be computed from the oracle's config (`lastMarkPrice * basketMaxConfidenceRatioBps * adverseConfidenceMultiplierBps / 1e8`) without a Pyth call. Alternatively, require equity strictly above `fadMarginBps` plus the maximum adverse shift during FAD/frozen windows.

### References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:274-284` — `validateWithdraw` open-position path
- `packages/perps/src/CfdEngineSettlementSidecar.sol:301-303, 326-331, 381-385` — guard collateral, margin bps, neutral mark
- `packages/perps/src/PletherOracle.sol:84, 95` — default confidence params
- `packages/perps/src/PletherOracle.sol:334-344, 729-743, 754-755, 943-972` — frozen basket and adverse shift
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1450-1461` — liquidation threshold and collateral
- `packages/perps/src/CfdEngine.sol:722` — guard NatSpec
- `packages/perps/SECURITY.md:425` — documented side-adverse equity checks

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/WithdrawGuardFrozenLiqShift.t.sol`. It uses only the repo's `BasePerpTest` fixtures.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

contract WithdrawGuardFrozenLiqShiftTest is BasePerpTest {

    address internal constant TRADER = address(0xA11CE);
    address internal constant KEEPER = address(0xBEEF);

    // Release parameters (packages/perps/DEPLOYMENT.md): MM 0.1%, IM 0.2%, FAD/frozen margin 3%.
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
        return 5_000_000e6;
    }

    function _pyth(uint256 price) internal view returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function test_withdrawApprovedThenSameBlockLiquidationDuringFrozenWindow() public {
        // Monday: open a 100k-token SHORT with spare free settlement.
        _fundTrader(TRADER, 10_000e6);
        _open(TRADER, CfdTypes.Side.SHORT, 100_000e18, 3_500e6, 1e8);
        (uint256 size, uint256 margin,,,,,) = engine.positions(TRADER);

        // Friday 20:00 UTC (live): mark refreshed so equity is ~3.01% of notional.
        vm.warp(1_709_928_000);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 lots = size / 1e20;
        uint256 cost = engine.positionEntryCostAssetAtoms(TRADER);
        uint256 fridayPrice = ((cost - margin) * 10_000 / (lots * (10_000 - 301))) & ~uint256(1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(int256(fridayPrice)), int32(-8), vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(_pyth(fridayPrice));
        assertFalse(engine.isOracleFrozen());

        // Saturday 12:00 UTC: oracle frozen, cached mark = Friday mark.
        vm.warp(1_709_985_600);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen());
        uint256 markPrice = engine.lastMarkPrice();
        uint64 markTime = engine.lastMarkTime();

        // 1) Withdraw guard approves: position judged NOT liquidatable at the cached mark.
        uint256 freeAmount = clearinghouse.balanceAssets(TRADER) - clearinghouse.pnlPledgeAssets(TRADER)
            - clearinghouse.vpiRebateReserveAssets(TRADER) - clearinghouse.liquidationReserveAssets(TRADER);
        uint256 before = settlementToken.balanceOf(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, freeAmount);
        assertEq(settlementToken.balanceOf(TRADER) - before, freeAmount, "withdraw succeeded");
        (uint256 sizeAfterWithdraw,,,,,,) = engine.positions(TRADER);
        assertGt(sizeAfterWithdraw, 0);

        // 2) Same block, same mark/time: untrusted keeper liquidates.
        assertEq(engine.lastMarkPrice(), markPrice);
        assertEq(engine.lastMarkTime(), markTime);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, _pyth(markPrice));

        (uint256 sizeAfterLiq,,,,,,) = engine.positions(TRADER);
        assertEq(sizeAfterLiq, 0, "liquidated right after an approved withdraw");
        emit log_named_uint("free settlement withdrawn (USDG 6d)", freeAmount);
        emit log_named_uint("engine mark (unchanged)", engine.lastMarkPrice());
        emit log_named_uint("keeper bounty credited (USDG 6d)", clearinghouse.balanceAssets(KEEPER) - keeperBefore);
    }
}
```

Run:

```
forge test --root packages/perps --match-path test/perps/WithdrawGuardFrozenLiqShift.t.sol -vv
```

Result:

```
[PASS] test_withdrawApprovedThenSameBlockLiquidationDuringFrozenWindow() (gas: 1882152)
Logs:
  free settlement withdrawn (USDG 6d): 6500000000
  engine mark (unchanged): 99649444
  keeper bounty credited (USDG 6d): 49814757
```

The test PASSES when the bug is present: it asserts that the withdrawal succeeded and that the position was liquidated in the same block at the same mark.

