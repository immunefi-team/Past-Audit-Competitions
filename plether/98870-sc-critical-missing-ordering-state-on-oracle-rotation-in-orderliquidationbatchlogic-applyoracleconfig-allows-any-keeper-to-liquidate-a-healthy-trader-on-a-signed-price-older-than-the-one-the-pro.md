# \#98870 [SC-Critical] Missing ordering state on oracle rotation in `OrderLiquidationBatchLogic.applyOracleConfig` allows any keeper to liquidate a healthy trader on a signed price older than the one the protocol already accepted

**Submitted on Nov 10th 2026 at 17:01:48 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98870
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Brief/Intro.** The security model says live baskets enforce that "component times cannot regress" (SECURITY.md:427-428). The oracle enforces this with per-feed and basket ordering checks, but only while `hasAcceptedBasket` is true (PletherOracle.sol:778, :786). A timelocked oracle rotation installs a fresh `PletherOracle` whose `hasAcceptedBasket` is false, and nothing carries the old oracle's accepted times across. On the first update after rotation, any keeper can submit a still-fresh signed print that is OLDER than what the old oracle already accepted. I reproduced the old oracle refusing that exact print with `PletherOracle__FeedTimeOutOfOrder`, then the new oracle accepting it after `finalizeOracleConfig()`, and a permissionless `executeLiquidation` closing a trader that an in-order print after rotation leaves healthy. The trader's balance fell from 960.00 to 605.92 USDG and the keeper received 50.00 USDG.

**Details.** All lines at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.
1. `OrderRouterAdmin.sol:184-191` stages a new oracle behind the 72 h timelock; `:196-201` finalizes it (the timelock only gates this call, `:197`) and calls `router.applyOracleConfig`.
2. `router/OrderLiquidationBatchLogic.sol:518-529` copies the six policy fields (`_oraclePolicy`, `:531-542`) to the candidate and verifies them. It copies no accepted prices or times. `OrderRouter.sol:420` then installs the candidate as `pletherOracle`.
3. The candidate's `hasAcceptedBasket` is false (`PletherOracle.sol:70`; it is set true only at `:799`). In `_resolveCurrentBasket` the per-feed check `if (hasAcceptedBasket && feedTime < _acceptedPrices[i].publishTime)` (`:778`) and the basket checks (`:786`, `:790`) are skipped. Only the staleness check (`:781`) runs, with the window `min(liquidationStalenessLimit, LIVE_TICK_LOOKBACK)` = 15 s (`:762-763`, `:29`, `:81`).
4. `executeLiquidation` (`OrderRouter.sol:493`) and `executeLiquidationBatch` (`:513`) are permissionless and "Available while paused" (`:487`, `:503`). The single path calls `updateLiquidationPrice` (`router/OrderLiquidationBatchLogic.sol:718-719`; `PletherOracle.sol:334`), the batch path `updateLiquidationBatchPrice` (`router/OrderLiquidationBatchLogic.sol:755-756`; `PletherOracle.sol:308`). Both reach `_updateLiquidationSnapshot` (`PletherOracle.sol:532`), which calls `_resolveCurrentBasket` in Liquidation mode (`:539`), and liquidate at the price it accepted.

So for one update after every rotation, the protocol accepts a component time that the previous oracle would reject as out of order. A print in the 15 s window that is adverse to a position, and older than the last accepted time, becomes usable to liquidate it.

**How this differs from the published items.** Known issue 1564, "Known limitation: live basket component selection" (SECURITY.md:687-698), accepts that a caller may choose an older, still-valid component while the oracle checks "per-feed publish-time ordering" (:690); this report is about that ordering floor being skipped after a rotation because `hasAcceptedBasket` resets, and the harm shown is a wrongful liquidation, not component selection. MED-13 covers governance retargeting armed stop-losses through a differently configured replacement oracle; here the replacement carries the same policy and feeds, and the harm is a keeper liquidating a healthy trader, not stop-loss retargeting.

### Impact

Direct theft of a trader's margin held in MarginClearinghouse through a wrongful liquidation. In the single-account test, on the release margin parameters:
- Trader position 100,000e18 -> 0; trader balance 960.000000 -> 605.921497 USDG (-354.078503 USDG, 36.9%).
- Keeper (an unfunded address) receives 50.000000 USDG.
- Twin runs in the same test: the old oracle refuses the same print, and an in-order print after rotation leaves the position and the balance unchanged.

The batch path (`executeLiquidationBatch`) shares the resolver and liquidates the same trader in the second test; that test liquidates one account. By reading `OrderRouter.sol:502` (up to 256 accounts per Pyth snapshot), every account liquidatable at the rewound print is exposed in that first update; the tests do not show more than one. The loss settles in the liquidation transaction. Under the undeployed-code rule I scale amounts to the minimum shown.

Demonstrated attacker gain: keeper +50.000000 USDG from an unfunded address, trader 960.000000 -> 605.921497 USDG, position 100,000e18 -> 0; capital committed: none beyond gas (TriageOracle.log:498-502)

### Risk Breakdown

Attacker: any address acting as a keeper; no role. Trigger: the owner's documented, timelocked oracle rotation (SECURITY.md:422; OrderRouterAdmin.sol:184-204). The window opens when the owner calls `finalizeOracleConfig()` (OrderRouterAdmin.sol:196-201) and the router swaps the pointer (OrderRouter.sol:420); activation time only gates that call (:197). The test shows the old oracle still refusing the print after activation and before finalization (TriageOracle.t.sol:2086-2087). Preconditions: (1) the attacker's update is the first one the new oracle accepts after `finalizeOracleConfig()`; any honest refresh first sets `hasAcceptedBasket` (PletherOracle.sol:799) and closes the window; (2) a genuine signed Pyth print exists that is adverse to the target position, older than the last accepted component time, and no more than 15 s old. The PoC builds that print with the mock; real Pyth prices decide whether one exists. No pause stops it: liquidations stay available while paused (OrderRouter.sol:487, :503). No role can reverse a settled liquidation. The code is in-house, not vendored.

### Recommendation

On rotation, seed the candidate's accepted prices, basket and `hasAcceptedBasket` from the active oracle in `applyOracleConfig` (or reject any component time below the previous oracle's accepted time on the candidate's first update), so `PletherOracle.sol:778` and `:786` hold across a rotation.

### References

- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:514-542` (`applyOracleConfig`, `_oraclePolicy`), `:706-730` (`executeLiquidation`), `:738-809` (`executeLiquidationBatch`).
- `packages/perps/src/OrderRouter.sol:415-421` (candidate installed at :420), `:486-520` (permissionless liquidation entries).
- `packages/perps/src/OrderRouterAdmin.sol:184-204` (72 h timelocked rotation).
- `packages/perps/src/PletherOracle.sol:29`, `:70`, `:81`, `:308`, `:334`, `:532-541`, `:749-801` (ordering guards at :778 and :786, flag set at :799), `:995-999`.
- `packages/perps/SECURITY.md:422` (timelocked rotation), `:427-428` ("component times cannot regress"), `:687-698` (known issue 1564).
- Prior audit MED-7 (Acknowledged) accepts per-feed selection of an older valid price only above the accepted floor, and records that no wrongful liquidation was established. This report shows the floor reset on rotation and the wrongful liquidation. MED-13 is a different mechanism (see Bug Description).

## Link to Proof of Concept

https://gist.github.com/tdev00/92291454f3611fd134661b3f716c498c

## Proof of Concept

Foundry tests in the repo's own `BasePerpTest` harness at the frozen commit. The harness deploys the real CfdEngine, HousePool, MarginClearinghouse, PletherOracle, OrderRouterAdmin and the liquidation sidecar; the router is `LegacyOrderRouterHarness`, which inherits `OrderRouter` and adds only a test commit adapter (`packages/perps/test/utils/LegacyOrderRouterHarness.sol:11-91`). Only USDG (`MockSettlementToken`) and Pyth (`MockPyth`) are mocked. Two identities: the victim `TRADER = address(0xC7)` and the keeper `address(0xD00D)`. Full test file, as a secret gist (README.md, TriageOracle.t.sol): https://gist.github.com/tdev00/92291454f3611fd134661b3f716c498c (TriageOracle.t.sol: 152,333 bytes, sha256 `2804fb4a006c9358e83a5e21cde7a664de902be44859bd4f5d980bc12bc16b7c`). The file is the receipted triage file as run and also holds other test suites; only `A18RotationTest` (with its base `A18RotationBase`) is this report's PoC.

Harness shortcuts, stated plainly:
- `baseMockPyth` supplies the signed prints. The test helper `_explicit` (`TriageOracle.t.sol:2038-2050`) sets each feed's price and publish time directly, with zero confidence and a zero update fee. It produces feed A = 100,500,000 at `t0 - 1` (`:2074`, `:2080`) after the old oracle accepted feed A = 100,000,000 at `t0` (`:2067`). The mock replaces signature verification only; the ordering checks the attack defeats are the real oracle's.
- The keeper is `address(0xD00D)` via `vm.prank` (`:2099-2100`, `:2119`); it is never funded.
- The trader is opened with the suite helpers `_fundTrader` and `_open` (`:2061-2062`).
- The candidate oracle uses the harness feed set: two feeds at 0.5e18 weight each (`BasePerpTest.sol:275-295`), so a 0.5% move on one feed moves the basket 0.25%.
- Release configuration: risk params are DEPLOYMENT.md:131-141 except `vpiFactor` 0 instead of 0.01e18 (`TriageOracle.t.sol:2011-2023`). The 354.08 USDG loss is the liquidation close at the rewound print plus the liquidation charge (the keeper's 50.000000 at keeperShareBps 5000 implies about 100 USDG of charge) plus carry accrued over the 72 h timelock; the log does not split it. The candidate copies the active oracle's policy (liquidationStalenessLimit 15, PletherOracle.sol:81).

What a real attack needs: a genuine signed Pyth print that is adverse to the position, older than the last accepted component time and inside the 15 s window, submitted as the first update the new oracle accepts after `finalizeOracleConfig()`.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run `forge build` once (via_ir, solc 0.8.35).
2. Download `TriageOracle.t.sol` from the gist into `poc/` at the repository root. It imports `../packages/perps/test/perps/BasePerpTest.sol`, and the root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`.
3. Run from the repository root (the root `foundry.toml` has no `test` key, so set `FOUNDRY_TEST=poc` and skip the repo's own trees):
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageOracle.t.sol --match-contract A18RotationTest -vv` reproduces the "Ran 2 tests" block below as logged.
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageOracle.t.sol --match-contract A18RotationTest --match-test test_Rotation_SingleLiquidationAtReleaseMargin -vv`
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageOracle.t.sol --match-contract A18RotationTest --match-test test_Rotation_BatchLiquidation -vv`

   The two single-test commands print "Ran 1 test" each.

Lead test, `test_Rotation_SingleLiquidationAtReleaseMargin` (`TriageOracle.t.sol:2077-2108`); setup `_stageRotation` (`:2060-2068`):

```solidity
function _stageRotation() internal returns (PletherOracle candidate, uint256 t0) {
    _fundTrader(TRADER, 1000e6);
    _open(TRADER, CfdTypes.Side.LONG, 100_000e18, 400e6, 1e8);
    candidate = _candidate();
    routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(candidate)}));
    vm.warp(routerAdmin.oracleConfigActivationTime());
    t0 = vm.getBlockTimestamp();
    router.updateMarkPrice(_explicit(100_000_000, t0, 100_000_000, t0));
}

function test_Rotation_SingleLiquidationAtReleaseMargin() public {
    (, uint256 t0) = _stageRotation();
    vm.warp(t0 + 6);
    bytes[] memory rewound = _explicit(SPIKE, t0 - 1, 100_000_000, t0 + 2);
    bytes[] memory honest = _explicit(100_000_000, t0 + 2, 100_000_000, t0 + 2);
    uint256 owed = clearinghouse.balanceAssets(TRADER);

    // Twin one: the old oracle refuses the rewound print.
    uint256 snap = vm.snapshotState();
    vm.expectPartialRevert(IPletherOracle.PletherOracle__FeedTimeOutOfOrder.selector);
    router.executeLiquidation(TRADER, rewound);
    vm.revertToState(snap);

    routerAdmin.finalizeOracleConfig();

    // Twin two: every print at or after what was accepted leaves the trader healthy.
    snap = vm.snapshotState();
    try router.executeLiquidation(TRADER, honest) {} catch {}
    assertEq(_positionSize(TRADER), 100_000e18, "honest print must not remove the trader");
    assertEq(clearinghouse.balanceAssets(TRADER), owed, "honest print must not charge the trader");
    vm.revertToState(snap);

    address liquidator = address(0xD00D);
    vm.prank(liquidator);
    router.executeLiquidation(TRADER, rewound);
    uint256 held = clearinghouse.balanceAssets(TRADER);
    console2.log("trader owed", owed);
    console2.log("trader held", held);
    console2.log("liquidator credit", clearinghouse.balanceAssets(liquidator));
    assertEq(_positionSize(TRADER), 0, "position survived");
    assertLt(held, owed, "trader lost nothing");
}
```

Batch variant, `test_Rotation_BatchLiquidation` (`:2111-2124`): same staging and rewound print, then `finalizeOracleConfig()` and `executeLiquidationBatch([TRADER], rewound)` from `address(0xD00D)`; asserts the position is 0 and the trader's balance is below `owed`.

Captured output, verbatim (`evidence/TriageOracle.log:496-504`; receipt `TriageOracle.receipt.json`: exit 0, outcome passed):

```
Ran 2 tests for poc/TriageOracle.t.sol:A18RotationTest
[PASS] test_Rotation_BatchLiquidation() (gas: 6926278)
[PASS] test_Rotation_SingleLiquidationAtReleaseMargin() (gas: 7686851)
Logs:
  trader owed 960000000
  trader held 605921497
  liquidator credit 50000000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 30.94ms (12.29ms CPU time)
```

The run that produced the receipt executed the whole file through the project's test runner; the commands above filter it to this suite.

### Before / after state delta (USDG, 6 decimals)

| Account / state | Before | After | Delta |
|---|---|---|---|
| Trader `0xC7` position (single and batch) | 100,000e18 | 0 | closed |
| Trader `0xC7` balance (single) | 960.000000 (log :500) | 605.921497 (log :501) | -354.078503 |
| Keeper `0xD00D` balance (single) | never funded (TriageOracle.t.sol:2099-2101) | 50.000000 (log :502) | +50.000000 |
| Same print, old oracle | - | reverts `PletherOracle__FeedTimeOutOfOrder` | asserted at TriageOracle.t.sol:2086-2087 |
| In-order print, new oracle | position 100,000e18, balance 960.00 | unchanged | asserted at TriageOracle.t.sol:2094-2096 |
