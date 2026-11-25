# \#98063 [SC-Low] Publish-order check in `PositionProtectionBook._freshProtectionObservation()` blocks TP/SL changes after a historical fill until the next live refresh

**Submitted on Nov 4th 2026 at 18:09:41 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98063
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Since PR #124, `PositionProtectionBook._freshProtectionObservation()` calls `PletherOracle.getLatestPrice(PriceMode.MarkRefresh)`, and that getter reverts when the accepted live basket is older than the Engine mark. A historical order fill creates that state whenever its Pyth tick is newer than the last live refresh: the fill moves the Engine mark to its tick and leaves the accepted live basket where the last live refresh put it. From then until the next live refresh (the reference oracle worker sends one on a configurable 30-second cadence), `createPositionProtection()`, `replacePositionProtection()` and `commitOpenOrderWithProtection()` revert with `PletherOracle__PriceOutOfOrder` for every trader. Both marks are within the 60-second freshness limit, and the Book's own check (`_acceptedMarkInstalled()`) treats that state as valid. No value is lost. Selected impact: **Contract fails to deliver promised returns, but doesn't lose value** (Low).

This finding was identified and documented collaboratively by georgibuilds and EagleEye.

## Vulnerability Details

**Root cause**

1. A historical fill moves the Engine mark but not the accepted live basket. The fill's oracle step resolves the fill price in `PletherOracle._resolveHistoricalOrderBasket()` ([PletherOracle.sol#L596-L641](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L641)), which never writes `_acceptedBasket`. Only the live path writes it: `_acceptedBasket = basket;` in `_resolveCurrentBasket()` ([PletherOracle.sol#L798](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L798)). The sidecar then installs the fill's publish time as the Engine mark whenever it is newer ([OrderRouterExecutionSidecar.sol#L979](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L979), [OrderRouterExecutionSidecar.sol#L1053-L1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061)). After a fill at tick `t1`, the Engine mark is at `t1` and the accepted basket stays at the last live refresh `t0 < t1`. The docs describe this state as normal: "historical execution may advance the Engine without advancing the accepted live basket" ([ORACLE_SYNCHRONIZATION.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md), lines 150-151).
2. The protection observation reverts in that state. `_create()`, `_replace()` and `_stageAttached()` call `_freshProtectionObservation()` ([PositionProtectionBook.sol#L229](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L229), [#L295](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L295), [#L463](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L463)). After checking that the Engine mark is fresh, that helper reads the oracle ([PositionProtectionBook.sol#L880-L885](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L880-L885)):

```solidity
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
```

   The getter loads the accepted basket with `_acceptedLiveBasket()` and passes it to `_snapshotFromBasket(basket, policy, true)` ([PletherOracle.sol#L577-L578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L577-L578)), which reverts with `PletherOracle__PriceOutOfOrder` when the basket is older than the Engine mark ([PletherOracle.sol#L893-L895](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L893-L895)):

```solidity
uint64 markTime = engine.lastMarkTime();
if (snapshot.publishTime < markTime) {
    revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
}
```

3. The check meant to judge this state never runs. `_acceptedMarkInstalled()` returns `true` when the accepted basket is older than the Engine mark ([PositionProtectionBook.sol#L888-L895](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L888-L895)):

```solidity
return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

   The getter one line earlier reverts on that same state, so the `acceptedTime < markTime` branch cannot be reached from create, replace or stage. The same predicate still arms an attached protection after its parent fill ([PositionProtectionBook.sol#L598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L598)).
4. This is a regression. In #120 (`81919218`), the protection path ran `_acceptedMarkInstalled()` alone after the Engine freshness check, so an accepted basket older than the mark passed. #124 (`eadc5c58`, the MED-28 fix) added the `getLatestPrice(MarkRefresh)` call in front of it. The sponsor's MED-28 disposition in the Cecuro report says the guard "rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark". An accepted basket behind the mark is neither. The interface promises that creation "validates trigger geometry against the accepted fresh basket" ([IPositionProtectionActions.sol#L12](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L12)). In the PoC that basket is 3 s old.

**Pre-conditions**
1. A queued order is filled at a historical tick `t1` (within `(commitTime, commitTime + 15]`) newer than the Engine mark `t0` set by the last live refresh. Only then does the fill move the mark ([OrderRouterExecutionSidecar.sol#L1058-L1059](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1058-L1059)). `executeOrder` and `executeOrderBatch` both update the mark this way ([OrderRouterExecutionSidecar.sol#L979](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L979), [OrderRouterExecutionSidecar.sol#L1036](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1036)). Only `updateMarkPrice`, liquidations, protection triggers and LP settlement write the accepted basket.
2. No live refresh lands between the fill and the trader's protection call. Plether's reference oracle worker "submits `updateMarkPrice` on a configurable 30-second cadence" on Arbitrum Sepolia ([README.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/README.md), lines 136-137). That worker is operator-run and out of scope. With that cadence, the state lasts up to about 30 s after each such fill.
3. The Engine mark and every accepted component are within the 60 s limit (deployed `orderExecutionStalenessLimit`): the Book checks the mark with `OrderRouter__ProtectionMarkTooStale` ([PositionProtectionBook.sol#L871-L878](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L871-L878)), and the getter checks each component with `PletherOracle__StalePrice` ([PletherOracle.sol#L720-L721](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L720-L721)). Without the bug, a refresh is needed only after 60 s; the bug adds the revert inside that window.
4. The market is live (not oracle-frozen).

No attacker is needed: any ordinary order execution creates the state. The reference client sends `createPositionProtection` and `replacePositionProtection` as single calls with no refresh in front (`packages/perps-aa-client/src/protection.ts`, lines 49 and 55).

PoC environment (all out of scope):
- Pyth's code is replaced by the repo's `MockPyth` (a stored price is kept only if newer), seeded with the real stored prices and fee (`singleUpdateFeeInWei() = 0`). The scope allows mocking "Pyth price updates and USDG"; USDG goes to the traders via `deal`.
- `ArbSys.arbBlockNumber()` is mocked to follow `vm.roll`. Forge 1.8.7 pins it to the fork block; on chain it advances every L2 block.
- No in-scope contract is modified or mocked. The PoC forks the deployed contracts instead of reusing `BasePerpTest`, as Immunefi requires. `keeper` and `refresher` are plain EOAs.

## Impact Details

**Selected impact:** Contract fails to deliver promised returns, but doesn't lose value

Severity: **Low**. Documented protection functions fail; no value is lost.

From the PoC (fork block 326037869): the accepted basket is at 1793813120, Bob's fill moves the Engine mark to 1793813121, and at the protection calls they are 3 s and 2 s old.

| Call after the fill | Result |
|---|---|
| Alice `replacePositionProtection` (tighten stop-loss 101598303 → 100630701) | reverts `PletherOracle__PriceOutOfOrder(1793813120, 1793813121)`; stop-loss stays 101598303; 246,896 gas spent on the failed call |
| Bob `createPositionProtection` on the position that just filled | same revert; no protection |
| Carol `commitOpenOrderWithProtection` | same revert; the open is not queued (0 orders) |
| Recovery: `OrderRouter.updateMarkPrice` with a live tick ≥ the fill tick | 303,550 gas, Pyth fee 0 wei on this deployment; afterwards all three calls succeed |

Effects:
- After each such fill, no trader can add, tighten or attach take-profit/stop-loss until the next live refresh lands: up to about 30 s with the reference worker's cadence, longer if that worker is slower or not running. That includes the trader whose open just filled, although the accepted basket stays fresh the whole time.
- The shipped client sends these calls without a refresh in front, so the trader's transaction fails and must be retried after the next refresh. A trader who does not want to wait has to fetch a signed Pyth update and send a separate `updateMarkPrice` transaction first (303,550 gas in the PoC, plus the Pyth fee).

Not higher: no funds are lost or frozen. A failed call is atomic, so existing protection stays armed and no order is queued. Cancellation, `cancelPositionProtectionAndCommitClose()` and ordinary closes keep working. The state ends at the next live refresh, which anyone can send.

## Recommendation

When the accepted live basket is older than the Engine mark, the already freshness-checked Engine mark is the newest signed observation. Use it as the protection observation and its time as the arming baseline, as an attached protection does after its parent fill, `_arm(parentProtection, ENGINE.lastMarkTime())` ([PositionProtectionBook.sol#L598-L602](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L598-L602)). Keep `getLatestPrice()` and `_acceptedMarkInstalled()` for the other states, so a basket ahead of or inconsistent with the mark is still rejected (MED-28). `PletherOracle` stays unchanged.
```diff
diff --git a/packages/perps/src/PositionProtectionBook.sol b/packages/perps/src/PositionProtectionBook.sol
index 08359c0f..f1055c5d 100644
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -877,6 +877,11 @@ contract PositionProtectionBook is IPositionProtectionBook, IOrderRouterErrors,
         ) {
             revert OrderRouter__ProtectionMarkTooStale();
         }
+        (bool accepted,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+        if (accepted && acceptedTime < markTime) {
+            // A historical fill installed a newer signed mark than the accepted live basket; observe that mark.
+            return (cachedMarkPrice, cachedMarkPrice, markTime);
+        }
         IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
         currentMarkPrice = current.markPrice;
         currentPublishTime = current.publishTime;
```

## References

- Root cause: [PositionProtectionBook.sol#L880](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L880), [PletherOracle.sol#L578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L578), [PletherOracle.sol#L893-L895](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L893-L895)
- Unreachable branch: [PositionProtectionBook.sol#L894](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L894)
- Affected entry points: `createPositionProtection` ([PositionProtectionBook.sol#L207](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L207)), `replacePositionProtection` ([PositionProtectionBook.sol#L257](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L257)), `commitOpenOrderWithProtection` ([PositionProtectionBook.sol#L432](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L432))
- State producer: [OrderRouterExecutionSidecar.sol#L1053-L1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061); only accepted-basket writer [PletherOracle.sol#L798](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L798)
- Introducing change: PR #124, commit `eadc5c58` (MED-28 fix), on top of PR #120, commit `81919218`
- Deployment: `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env` (deployed from `1209ca8a`; `packages/perps/src` is identical at `a82d4fef`)

## Proof of Concept

# Proof of Concept

### Failure path

1. Alice commits an ordinary LONG open (2,000 units, 200 USDG margin) at `t`. A keeper fills it at `t+3` with its first post-commit tick `t+1`. (`// Step 1`)
2. At `t+4`, a refresher (for example the oracle worker) calls `updateMarkPrice` with live ticks at `t+4`. The accepted basket and the Engine mark are both at `t+4`. (`// Step 2`)
3. Alice arms take-profit/stop-loss 5% either side of the mark. It succeeds. (`// Step 3`)
4. Bob commits an ordinary LONG open at `t+4`. At `t+7` a keeper fills it at tick `t+5`. The Engine mark moves to `t+5`; the accepted basket stays at `t+4`. (`// Step 4`)
5. Alice tries to tighten her stop-loss with `replacePositionProtection`. It reverts with `PletherOracle__PriceOutOfOrder(t+4, t+5)`, and her stop-loss is unchanged. (`// Step 5`)
6. Bob calls `createPositionProtection` on his new position. Same revert. (`// Step 6`)
7. Carol calls `commitOpenOrderWithProtection`. Same revert; nothing is queued. (`// Step 7`)
8. A refresher pays for `updateMarkPrice` with a live tick at `t+7`. The same three calls then succeed. (`// Step 8`)

Control (`test_F019_control`): the same flow with a live refresh between Bob's fill and the protection calls. All three calls succeed, with or without the fix.

Environment: forge 1.8.7, this repository at `a82d4fef`, the test at `packages/perps/test/F019_LiveBasketPublishOrderCheckIn.t.sol` (run from `packages/perps`), env var `ARB_SEPOLIA_RPC_URL` (any Arbitrum Sepolia RPC with state at block 326037869). Optional `F019_LOCAL_BOOK=1` runs the test against the `PositionProtectionBook` built from the checkout, to check the fix.

Add the test at the path shown in the command, then run:
```bash
# Local fork of Arbitrum Sepolia (chain id 421614) at block 326037869, Plether Perps v2.0.0 (USDG).
# Read-only RPC; nothing is sent to the public network. Any archive RPC endpoint works.
ARB_SEPOLIA_RPC_URL=<arbitrum-sepolia-archive-rpc-url> forge test --match-path test/F019_LiveBasketPublishOrderCheckIn.t.sol --match-test test_F019_poc -vvv

# Same test against a local Anvil fork:
anvil --fork-url <arbitrum-sepolia-archive-rpc-url> --fork-block-number 326037869 --chain-id 421614 &
ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F019_LiveBasketPublishOrderCheckIn.t.sol --match-test test_F019_poc -vvv
```
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Live-basket publish-order check in `PositionProtectionBook._freshProtectionObservation()` makes take-profit/stop-loss creation and replacement revert after any historical order fill until a paid live refresh
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F019_LiveBasketPublishOrderCheckIn.t.sol --match-test test_F019_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number <FORK_BLOCK> &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F019_LiveBasketPublishOrderCheckIn.t.sol ...
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {ICfdEngineCore} from "@plether/perps/interfaces/ICfdEngineCore.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public Router getters and the permissionless mark refresh (not all on the trader/keeper interfaces).
interface IRouterF019 {
    function nextCommitId() external view returns (uint64);
    function nextExecuteId() external view returns (uint64);
    function pletherOracle() external view returns (address);
    function positionProtectionBook() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function pendingOrderCounts(address account) external view returns (uint256);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
}

/// @dev Real Pyth getter, read before the Pyth stand-in is installed.
interface IPythFeeF019 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IOracleFeedsF019 {
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function orderExecutionStalenessLimit() external view returns (uint256);
}

interface IArbSysBlockF019 {
    function arbBlockNumber() external view returns (uint256);
}

contract F019_LiveBasketPublishOrderCheckIn is Test {
    // Pinned Arbitrum Sepolia block (2026-11-04 17:25:16 UTC, a Wednesday: FX market open), after deployment block 323658447.
    uint256 internal constant FORK_BLOCK = 326_037_869;

    // Deployed v2.0.0 addresses (addresses.env). Deployed from 1209ca8a; packages/perps/src is identical at scope a82d4fef.
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant PROTECTION_BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    // Pyth contract the deployed oracle reads (out of scope; replaced below by the repo's own MockPyth).
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    IPerpsTraderActions internal router = IPerpsTraderActions(ROUTER);
    IPerpsKeeper internal keeperRouter = IPerpsKeeper(ROUTER);
    IRouterF019 internal routerViews = IRouterF019(ROUTER);
    IPletherOracle internal oracle = IPletherOracle(ORACLE);
    ICfdEngineCore internal engine = ICfdEngineCore(ENGINE);
    IOrderLifecycleBook internal lifecycle = IOrderLifecycleBook(LIFECYCLE_BOOK);
    IPositionProtectionActions internal protection = IPositionProtectionActions(PROTECTION_BOOK);
    IPositionProtectionViews internal protectionViews = IPositionProtectionViews(PROTECTION_BOOK);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal alice = makeAddr("alice"); // has a position and armed protection; wants to tighten it
    address internal bob = makeAddr("bob"); // ordinary trader whose open is filled, then wants a stop-loss
    address internal carol = makeAddr("carol"); // flat trader opening with attached protection
    address internal keeper = makeAddr("keeper"); // any order executor
    address internal refresher = makeAddr("refresher"); // any live mark refresher (e.g. the oracle worker)

    bytes32[] internal feedIds;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal stalenessLimit;

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");
        assertEq(_arbBlock(), FORK_BLOCK, "ArbSys serves the fork's L2 block");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(routerViews.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(routerViews.positionProtectionBook(), PROTECTION_BOOK, "protection book drifted");
        assertEq(address(oracle.pyth()), PYTH, "oracle Pyth drifted");
        assertFalse(oracle.isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertEq(oracle.orderSettlementWindow(), 15, "orderSettlementWindow drifted");
        stalenessLimit = IOracleFeedsF019(ORACLE).orderExecutionStalenessLimit();
        assertEq(stalenessLimit, 60, "orderExecutionStalenessLimit drifted");
        assertEq(routerViews.nextExecuteId(), 0, "fork queue must be empty");
        (bool accepted,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        assertTrue(accepted, "deployment has an accepted live basket");
        assertEq(acceptedTime, engine.lastMarkTime(), "fork starts with accepted basket and Engine mark in sync");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleFeedsF019(ORACLE).pythFeedIds(i));
        }
        // Default: the DEPLOYED Book bytecode. Fix check only (F019_LOCAL_BOOK=1): swap in the PositionProtectionBook
        // runtime compiled from this checkout, with the deployed immutables (ROUTER, ENGINE) and storage kept.
        if (vm.envOr("F019_LOCAL_BOOK", false)) {
            _installLocalBook();
        }

        // Pyth is out of scope: replace its code with the repo's MockPyth (Pyth semantics: unique historical parse,
        // updatePriceFeeds keeps a feed only if newer). Storage starts from the REAL stored prices at FORK_BLOCK.
        // The real deployment quotes a zero update fee; keep it.
        pythFee = IPythFeeF019(PYTH).singleUpdateFeeInWei();
        for (uint256 i; i < 6; ++i) {
            storedAtFork.push(IPyth(PYTH).getPriceUnsafe(feedIds[i]));
        }
        vm.etch(PYTH, type(MockPyth).runtimeCode);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0)); // MockPyth scalar slots only
        }
        pyth.setFee(pythFee);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            pyth.setPrice(feedIds[i], p.price, p.conf, p.expo, p.publishTime);
            assertLe(p.publishTime, block.timestamp, "real stored Pyth time must not be in the future");
        }

        // Three ordinary traders with margin (USDG is out of scope; deal to users only).
        _fund(alice, 1_000e6);
        _fund(bob, 1_000e6);
        _fund(carol, 1_000e6);
        vm.deal(keeper, 1 ether);
        vm.deal(refresher, 1 ether);
    }

    /// The claim, against live state at FORK_BLOCK. Must FAIL once the mitigation is applied.
    function test_F019_poc() public {
        uint64 t = uint64(block.timestamp);
        uint256 mark = _openSyncAndProtectAlice(t);
        uint64 aliceProtectionId = protectionViews.activePositionProtectionId(alice);
        PositionProtectionTypes.PositionProtectionView memory armed =
            protectionViews.getPositionProtection(aliceProtectionId);

        // Step 4: bob commits an ordinary LONG open; 3 s later a keeper fills it at its first post-commit tick (t+5).
        //         The fill advances the Engine mark to t+5. The accepted live basket stays at the t+4 refresh.
        uint64 bobId = _commit(bob, CfdTypes.Side.LONG, "bob-open");
        _advance(t + 7, 36);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory fill =
            keeperRouter.executeOrder{value: 2 * pythFee}(bobId, _historicalPayload(t + 4, t + 5));
        assertEq(uint256(fill.status), uint256(OrderV3Types.LifecycleStatus.Executed), "bob's open filled");
        (bool accepted,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertTrue(accepted, "accepted live basket still present");
        assertEq(acceptedTime, t + 4, "fill left the accepted basket at the last live refresh");
        assertEq(markTime, t + 5, "fill advanced the Engine mark");
        // vm.getBlockTimestamp(): via-ir may fold block.timestamp reads across vm.warp in one function.
        uint256 nowTs = vm.getBlockTimestamp();
        assertEq(nowTs, t + 7, "clock at the protection calls");
        assertLe(nowTs - acceptedTime, stalenessLimit, "accepted basket is fresh");
        assertLe(nowTs - markTime, stalenessLimit, "Engine mark is fresh");
        console.log("accepted live basket publish time (last live refresh)", uint256(acceptedTime));
        console.log("Engine mark publish time (historical fill)", uint256(markTime));
        console.log("accepted basket age at the protection calls (s)", nowTs - acceptedTime);
        console.log("Engine mark age at the protection calls (s)", nowTs - markTime);
        bytes memory expected =
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime);

        // Step 5: alice tightens her stop-loss. The call reverts inside getLatestPrice(MarkRefresh).
        uint256 tighterStop = armed.stopLossTriggerPrice - mark / 100;
        vm.prank(alice);
        uint256 gasBefore = gasleft();
        (bool ok, bytes memory ret) = PROTECTION_BOOK.call(
            abi.encodeCall(
                IPositionProtectionActions.replacePositionProtection,
                (aliceProtectionId, _params(CfdTypes.Side.LONG, mark, armed.takeProfitTriggerPrice, tighterStop))
            )
        );
        uint256 replaceGas = gasBefore - gasleft();
        assertFalse(ok, "replace reverts although accepted basket and Engine mark are fresh");
        assertEq(keccak256(ret), keccak256(expected), "revert is PriceOutOfOrder(accepted basket time, Engine mark time)");
        assertEq(
            protectionViews.getPositionProtection(aliceProtectionId).stopLossTriggerPrice,
            armed.stopLossTriggerPrice,
            "alice's stop-loss was not tightened"
        );
        console.log("alice stop-loss kept (1e8)", armed.stopLossTriggerPrice);
        console.log("alice stop-loss requested (1e8)", tighterStop);
        console.log("failed replacePositionProtection gas", replaceGas);

        // Step 6: bob, whose open just filled, tries to add take-profit/stop-loss to it. Same revert.
        vm.prank(bob);
        (ok, ret) = PROTECTION_BOOK.call(
            abi.encodeCall(IPositionProtectionActions.createPositionProtection, (_params(CfdTypes.Side.LONG, mark, 0, 0)))
        );
        assertFalse(ok, "create reverts after the historical fill");
        assertEq(keccak256(ret), keccak256(expected), "create revert is PriceOutOfOrder");
        assertEq(protectionViews.activePositionProtectionId(bob), 0, "bob's position has no protection");

        // Step 7: carol (flat) commits an open with attached protection. The whole commit reverts.
        OrderV3Types.OrderRequest memory carolOpen = _request(CfdTypes.Side.LONG, "carol-open");
        vm.prank(carol);
        (ok, ret) = PROTECTION_BOOK.call(
            abi.encodeCall(
                IPositionProtectionActions.commitOpenOrderWithProtection,
                (carolOpen, _params(CfdTypes.Side.LONG, mark, 0, 0))
            )
        );
        assertFalse(ok, "attached open commit reverts after the historical fill");
        assertEq(keccak256(ret), keccak256(expected), "attached commit revert is PriceOutOfOrder");
        assertEq(routerViews.pendingOrderCounts(carol), 0, "carol's open was not queued");

        // Step 8: the only way out is a separate, paid live refresh whose newest tick is >= the fill tick.
        bytes[] memory live = _livePayload(t + 7); // newest tick at the current second (>= fill tick t+5)
        vm.prank(refresher);
        gasBefore = gasleft();
        IRouterF019(ROUTER).updateMarkPrice{value: pythFee}(live);
        uint256 refreshGas = gasBefore - gasleft();
        (, , acceptedTime) = oracle.getAcceptedBasketMark();
        assertEq(acceptedTime, engine.lastMarkTime(), "live refresh re-synchronizes basket and Engine mark");
        console.log("recovery updateMarkPrice gas", refreshGas);
        console.log("recovery updateMarkPrice Pyth fee (wei, deployed singleUpdateFeeInWei)", pythFee);

        vm.prank(alice);
        protection.replacePositionProtection(
            aliceProtectionId, _params(CfdTypes.Side.LONG, mark, armed.takeProfitTriggerPrice, tighterStop)
        );
        assertEq(
            protectionViews.getPositionProtection(aliceProtectionId).stopLossTriggerPrice,
            tighterStop,
            "after the refresh the same replace succeeds"
        );
        vm.prank(bob);
        protection.createPositionProtection(_params(CfdTypes.Side.LONG, mark, 0, 0));
        assertGt(protectionViews.activePositionProtectionId(bob), 0, "after the refresh bob's create succeeds");
        carolOpen = _request(CfdTypes.Side.LONG, "carol-open-2");
        vm.prank(carol);
        protection.commitOpenOrderWithProtection(carolOpen, _params(CfdTypes.Side.LONG, mark, 0, 0));
        assertEq(routerViews.pendingOrderCounts(carol), 1, "after the refresh carol's attached open is queued");
    }

    /// Control: the same three protection calls with no fill after the last live refresh. Must PASS with and without
    /// the fix.
    function test_F019_control() public {
        uint64 t = uint64(block.timestamp);
        uint256 mark = _openSyncAndProtectAlice(t);
        uint64 aliceProtectionId = protectionViews.activePositionProtectionId(alice);
        PositionProtectionTypes.PositionProtectionView memory armed =
            protectionViews.getPositionProtection(aliceProtectionId);

        // bob's open fills and a live refresh follows before anyone touches protection.
        uint64 bobId = _commit(bob, CfdTypes.Side.LONG, "bob-open");
        _advance(t + 7, 36);
        vm.prank(keeper);
        keeperRouter.executeOrder{value: 2 * pythFee}(bobId, _historicalPayload(t + 4, t + 5));
        vm.prank(refresher);
        IRouterF019(ROUTER).updateMarkPrice{value: pythFee}(_livePayload(t + 7));
        (, , uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        assertEq(acceptedTime, engine.lastMarkTime(), "control: basket and Engine mark in sync");

        uint256 tighterStop = armed.stopLossTriggerPrice - mark / 100;
        vm.prank(alice);
        protection.replacePositionProtection(
            aliceProtectionId, _params(CfdTypes.Side.LONG, mark, armed.takeProfitTriggerPrice, tighterStop)
        );
        assertEq(protectionViews.getPositionProtection(aliceProtectionId).stopLossTriggerPrice, tighterStop);
        vm.prank(bob);
        protection.createPositionProtection(_params(CfdTypes.Side.LONG, mark, 0, 0));
        assertGt(protectionViews.activePositionProtectionId(bob), 0);
        OrderV3Types.OrderRequest memory carolOpen = _request(CfdTypes.Side.LONG, "carol-open");
        vm.prank(carol);
        protection.commitOpenOrderWithProtection(carolOpen, _params(CfdTypes.Side.LONG, mark, 0, 0));
        assertEq(routerViews.pendingOrderCounts(carol), 1);
    }

    // ------------------------------------------------------------------ shared steps

    /// @dev Steps 1-3: alice opens LONG (filled at t+1), a live refresh at t+4 syncs the accepted basket and the Engine
    ///      mark, and alice arms take-profit/stop-loss. Returns the neutral mark used for the trigger geometry.
    function _openSyncAndProtectAlice(uint64 t) internal returns (uint256 mark) {
        // Step 1: alice commits an ordinary LONG open at t; a keeper fills it 3 s later at its first tick (t+1).
        uint64 aliceId = _commit(alice, CfdTypes.Side.LONG, "alice-open");
        _advance(t + 3, 12);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r =
            keeperRouter.executeOrder{value: 2 * pythFee}(aliceId, _historicalPayload(t, t + 1));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "alice's open filled");

        // Step 2: any refresher (e.g. the oracle worker) installs a live basket at t+4: basket and Engine in sync.
        _advance(t + 4, 16);
        vm.prank(refresher);
        IRouterF019(ROUTER).updateMarkPrice{value: pythFee}(_livePayload(t + 4));
        (, , uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        assertEq(acceptedTime, t + 4, "live refresh installed");
        assertEq(engine.lastMarkTime(), t + 4, "Engine mark synced with the accepted basket");

        // Step 3: in sync, alice arms protection 5% either side of the mark (LONG: TP below, SL above).
        mark = engine.lastMarkPrice();
        vm.prank(alice);
        protection.createPositionProtection(_params(CfdTypes.Side.LONG, mark, 0, 0));
        assertGt(protectionViews.activePositionProtectionId(alice), 0, "alice armed protection while in sync");
    }

    // ------------------------------------------------------------------ helpers

    function _installLocalBook() internal {
        PositionProtectionBook local = new PositionProtectionBook(ROUTER, ENGINE);
        vm.etch(PROTECTION_BOOK, address(local).code);
        console.log("F019_LOCAL_BOOK: running the locally compiled PositionProtectionBook at the deployed address");
    }

    function _arbBlock() internal view returns (uint256) {
        return IArbSysBlockF019(address(100)).arbBlockNumber();
    }

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge 1.8.7 serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp, uint256 l2Blocks) internal {
        uint256 l2Block = FORK_BLOCK + l2Blocks;
        vm.warp(timestamp);
        vm.roll(l2Block);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF019.arbBlockNumber.selector), abi.encode(l2Block)
        );
        assertEq(_arbBlock(), l2Block, "L2 block advanced");
    }

    function _fund(address trader, uint256 amount) internal {
        deal(USDG, trader, amount);
        vm.startPrank(trader);
        IERC20(USDG).approve(CLEARINGHOUSE, amount);
        IMarginClearinghouse(CLEARINGHOUSE).depositFor(trader, amount);
        vm.stopPrank();
    }

    /// @dev Ordinary open: 2,000 units (~1,940 USDG notional at the fork's ~0.97 basket), 200 USDG margin.
    function _request(CfdTypes.Side side, string memory tag)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        uint256 window = routerViews.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F019", tag)),
            side: side,
            sizeDelta: 2_000e18,
            marginDelta: 200e6,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: lifecycle.currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commit(address trader, CfdTypes.Side side, string memory tag) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(side, tag);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @dev Direction-aware TP/SL 5% either side of `mark` (overridable), unlimited close ceilings.
    function _params(CfdTypes.Side side, uint256 mark, uint256 takeProfit, uint256 stopLoss)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        if (takeProfit == 0) {
            takeProfit = side == CfdTypes.Side.LONG ? mark * 95 / 100 : mark * 105 / 100;
        }
        if (stopLoss == 0) {
            stopLoss = side == CfdTypes.Side.LONG ? mark * 105 / 100 : mark * 95 / 100;
        }
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev Signed-update stand-in: every feed's first tick after `commit` at `tick`, at the real stored price of
    ///      FORK_BLOCK (previous tick `commit`).
    function _historicalPayload(uint64 commit, uint64 tick) internal view returns (bytes[] memory data) {
        data = _payload(tick, commit);
    }

    /// @dev Signed-update stand-in for a live refresh: every feed ticks at `tick`, at the real stored price.
    function _livePayload(uint64 tick) internal view returns (bytes[] memory data) {
        data = _payload(tick, tick - 1);
    }

    function _payload(uint64 tick, uint64 prevTick) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: p.price, conf: p.conf, expo: p.expo, publishTime: tick, prevPublishTime: prevTick
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```
