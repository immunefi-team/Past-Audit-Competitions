# \#98601 [SC-Low] Open/increase bounty reservation checkpoints and transfers accrued carry but reports a bounty-only commitment, bypassing both signed debit ceilings

**Submitted on Nov 8th 2026 at 23:22:49 UTC by @adeolu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98601
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


## Brief/Intro

The open/increase commitment path can collect accrued carry from a trader's position margin and free settlement while reporting only the execution bounty to the lifecycle Book. Consequently, the trader's signed `maxGrossAccountDebitAssets` and `maxActionChargeAssets` limits are checked against incomplete accounting, allowing an unchanged signed order submitted by a relayer to debit the account beyond both limits and remain successfully queued. Exploitation forces premature collection of already-owed carry and reserves order funds contrary to the trader's authorization, causing unauthorized custody movement and **Medium — Griefing**; it does not create new carry debt or transfer the collected carry to the relayer.

## Vulnerability Details

### The bounds are mandatory financial authority

Every V3 request includes `ExecutionBounds`. The source explicitly says that maximum values are inclusive, zero is a real zero allowance, and `maxGrossAccountDebitAssets` covers settlement debit and the reserved bounty. See [`OrderV3Types.sol#L131-L151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L151).

```solidity
/// @notice Mandatory financial authority supplied with every V3 order.
/// @dev Maximum values are inclusive and zero is a real zero allowance.
///      `maxGrossAccountDebitAssets` covers settlement debit, trader-claim
///      consumption, and the reserved bounty;
struct ExecutionBounds {
    // ...
    uint256 maxExecutionBountyAssets;
    uint256 maxExecutionNotionalAssets;
    uint256 maxGrossAccountDebitAssets;
    uint256 maxActionChargeAssets;
    // ...
}
```

Thus, a request with:

- `maxGrossAccountDebitAssets = 200_000` (0.20 USDG), and
- `maxActionChargeAssets = 0`

does not authorize hundreds of USDG of carry to be collected while the order is committed.

### Root cause: the open branch creates an incomplete effects report

For closes, `_reserveExecutionBounty` delegates to the engine and receives a populated `CloseCommitment`. For an open or increase, however, it sets only `bountyFromFreeAssets`, leaving `carryCollectedAssets`, `carryOutstandingAssets`, `settlementBeforeAssets`, `settlementAfterAssets`, and `freeSettlementAfterAssets` at their zero defaults. It then calls `lockReservedSettlement`. See [`OrderReservationAccounting.sol#L108-L128`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128).

```solidity
function _reserveExecutionBounty(
    address account,
    uint256 sizeDelta,
    uint256 bounty,
    bool isClose
) internal returns (CfdEnginePlanTypes.CloseCommitment memory effects) {
    if (isClose) {
        // The close path returns the engine's complete effects report.
        return _reserveCloseExecutionBounty(account, sizeDelta);
    }

    // Open/increase path populates only this field.
    effects.bountyFromFreeAssets = bounty;
    if (bounty == 0) {
        return effects;
    }
    if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
        revert OrderRouter__InsufficientFreeEquity();
    }
    clearinghouse.lockReservedSettlement(account, bounty);
}
```

The missing report would be harmless only if `lockReservedSettlement` merely moved funds between internal buckets. It does not: its first state-changing operation checkpoints carry. See [`MarginClearinghouse.sol#L812-L824`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L812-L824) and [`MarginClearinghouse.sol#L1336-L1355`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1336-L1355).

```solidity
function lockReservedSettlement(address account, uint256 amountAssets)
    external
    onlyReservedSettlementLocker
{
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}

function _checkpointCarryBeforeMarginChange(address account) internal {
    address engine_ = engine;
    if (engine_ == address(0)) return;
    _realizeOrCheckpointCarryBeforeMarginChange(account);
}

function _realizeOrCheckpointCarryBeforeMarginChange(address account) internal {
    // ...
    ICfdEngineCore(engine_).realizeCarryBeforeMarginChange(account);
}
```

The engine consumes the carry from position margin and then free settlement, reduces the trader's settlement custody, transfers the realized USDG to the HousePool, and records the pool inflow. See [`CfdEngine.sol#L1391-L1445`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1391-L1445).

```solidity
(uint256 marginConsumedAssets, uint256 freeSettlementConsumedAssets, uint256 uncoveredAssets) =
    clearinghouse.consumeSettlementLoss(account, carryDueAssets, address(this));
realizedCarryAssets = carryDueAssets - uncoveredAssets;
unsettledCarryAssets[account] = uncoveredAssets;

if (realizedCarryAssets > 0) {
    settlementToken.safeTransfer(address(pool), realizedCarryAssets);
    pool.recordClaimantInflow(
        realizedCarryAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

`lockReservedSettlement` returns no accounting result, so the `effects` object held by the Router remains unchanged even after this transfer.

### The incomplete report makes both checks pass

The commit handler reserves the bounty, forwards the incomplete report to the lifecycle book, then reserves margin and stores the pending queue record. See [`OrderCommitHandler.sol#L17-L36`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L17-L36) and [`OrderRouterBase.sol#L110-L115`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L110-L115).

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
clearinghouse.recordBountyReservation(/* ... */);
_onOrderCommitted(order, effects); // lifecycleBook.recordCommitment(...)
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
_recordCommittedOrder(order);
```

`recordCommitment` computes both limits exclusively from the supplied report. See [`OrderLifecycleBook.sol#L327-L360`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L360).

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(/* ... */);
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(/* ... */);
}

if (
    effects.settlementBeforeAssets != effects.settlementAfterAssets + effects.carryCollectedAssets
        || effects.bountyFromFreeAssets != pending.executionBountyAssets
) {
    revert OrderLifecycleBook__InvalidCommitmentEffects();
}
```

Because the open/increase report contains:

```text
carryCollectedAssets       = 0
settlementBeforeAssets     = 0
settlementAfterAssets      = 0
bountyFromFreeAssets       = 200,000
```

the Book observes `gross = 0 + 200,000`, accepts a zero action-charge ceiling, and even accepts the settlement identity `0 == 0 + 0`. In reality, carry has already reduced the trader's account and reached the HousePool.

The successful call then reserves the order's margin and writes the order as `Pending`. Because there is no revert, the carry transfer, bounty reservation, committed-margin reservation, lifecycle commitment, and live queue entry all remain durable.

### Relayer reachability

The Router uses `msg.sender` as the trading account for a public commitment. See [`OrderLiquidationBatchLogic.sol#L127-L136`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L127-L136).

A plain EOA relayer therefore cannot call the Router directly and impersonate a trader. The reproducible route is:

1. The owner signs an ERC-4337 `PackedUserOperation` whose exact wallet calldata calls `OrderRouter.commitOrder(increase)`.
2. An arbitrary relayer later calls the public EntryPoint's `handleOps` with the unchanged signed operation.
3. EntryPoint validates the signature and calls the SimpleAccount.
4. The SimpleAccount calls the Router, so the Router correctly sees the smart account—not the relayer—as `msg.sender`.

The PoC uses the live Arbitrum Sepolia EntryPoint v0.8, SimpleAccount factory, and SimpleAccount implementation on a **local fork**, with code-hash assertions. It does not require Plether's bundler, gas sponsor, or any privileged service. The root cause remains entirely within the in-scope `packages/perps/src` contracts.

### Concrete attack path

1. A smart-account trader deposits 100,000 USDG.
2. Through the normal delayed-order lifecycle, the trader opens a 300,000-token LONG at `0.80` with 20,000 USDG margin.
3. Before carry accrues, the owner signs an exact same-side increase for 100,000 tokens and 1,000 USDG margin. It permits a 0.20 USDG bounty, sets the gross-debit cap to the same 0.20 USDG, sets the action-charge cap to zero, and remains submittable for 30 days.
4. The relayer waits until `submitBy`. The unchanged request remains valid according to the production open planner after projected carry.
5. Bounty reservation calls `lockReservedSettlement`, which realizes accrued carry and transfers it to the HousePool.
6. The Book receives a report of zero carry, accepts both signed ceilings, and stores order 2 as `Pending`.

The test dynamically derives projected carry from the production carry indexes and proves that the actual account debit is exactly that projected carry, the HousePool credit is exactly the same amount, and the debit exceeds both signed limits. This avoids hardcoding the exploit result.

### Expected behavior

The open/increase reservation path should report the same complete commitment effects that the state-changing carry checkpoint produced. With accurate effects, `recordCommitment` would calculate `carry + bounty`, see that this exceeds `maxGrossAccountDebitAssets`, see that carry exceeds the zero `maxActionChargeAssets`, and revert with `OrderLifecycleBook__CommitmentBoundExceeded`. EVM atomicity would then roll back the carry realization, HousePool transfer, bounty reservation, margin reservation, and queue write.

A robust fix is to make the carry-checkpointing reservation operation return authoritative before/after balances and collected/outstanding carry, or to explicitly checkpoint through an engine method that returns those values before constructing the lifecycle commitment. The Router must not infer zero effects from a `void` function that can debit the account.

## Impact Details

**Selected in-scope impact: Medium — Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).**

The transaction removes real USDG from the trader's internal settlement custody and transfers it to the HousePool even though the signed financial limits say that only the 0.20 USDG bounty may count toward gross debit and that no action charge is allowed. The realized carry transfer is final for that successful transaction; canceling or later failing the queued increase does not reverse the earlier carry checkpoint. The transaction also leaves the bounty and 1,000 USDG of committed margin reserved while the order remains pending.

The amount bypassing the bounds is not capped by either signed ceiling. It is bounded instead by the carry accrued on the existing position and the settlement/margin available to realize it while still leaving enough free assets for the bounty and margin reservation. Larger positions and longer permitted submission windows increase the amount a relayer can force to be realized at a time chosen by the relayer. In the supplied attack scenario, 199,026,792 USDG atoms (199.026792 USDG) are consumed and transferred while the gross cap is only 200,000 atoms (0.20 USDG) and the action-charge cap is zero.

This should not be described as Critical direct theft in its current form:

- the relayer does not receive the carry; the HousePool does;
- the carry is an already-accrued liability of the position rather than a fabricated fee;
- exploitation requires a still-valid owner-signed wallet operation (or the trader submitting the request directly); and
- the relayer's only receipt is ordinary ERC-4337 gas reimbursement from the smart account's EntryPoint deposit.

The security violation is nevertheless concrete: the signature's explicit debit ceilings fail to constrain a successful state transition, and a third party can select when that unauthorized realization occurs. This is direct user damage without attacker profit, matching the program's Medium griefing category.

## References

- [Plether competition scope and in-scope impacts](https://immunefi.com/audit-competition/plether/scope/)
- [Plether competition information](https://immunefi.com/audit-competition/plether/information/)
- [Pinned in-scope repository commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [`OrderReservationAccounting.sol#L108-L128`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128)
- [`MarginClearinghouse.sol#L812-L824`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L812-L824)
- [`MarginClearinghouse.sol#L1336-L1355`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1336-L1355)
- [`CfdEngine.sol#L1391-L1445`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1391-L1445)
- [`OrderCommitHandler.sol#L17-L36`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L17-L36)
- [`OrderLifecycleBook.sol#L327-L360`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L360)
- [ERC-4337 specification](https://eips.ethereum.org/EIPS/eip-4337)



## Proof of Concept


## Proof of Concept

### What the PoC proves

The PoC:

- runs at the exact in-scope commit;
- deploys the production perps contracts through the repository's existing test fixture;
- uses the existing mock settlement token and Pyth fixture only for the external USDG/oracle dependencies permitted by the scope rules;
- creates no new mocks and writes no protocol storage directly;
- opens and executes the initial position through actual deposit, commit, and execute functions;
- uses the fixture's normal `baseCarryBps = 500` configuration (carry is not disabled or manually injected);
- uses a real deployed EntryPoint v0.8, SimpleAccount factory, and SimpleAccount implementation on a local Arbitrum Sepolia fork;
- derives the counterfactual account salt from the current fork head so the PoC does not depend on a permanently undeployed published address;
- has the wallet owner sign the exact operation before carry accrues and has a distinct relayer submit it unchanged after 30 days;
- verifies the production planner still considers the increase valid;
- verifies the smart account's balance debit equals projected carry and the HousePool receives the same amount;
- verifies the debit exceeds both signed ceilings;
- verifies the Book permanently records zero collected carry; and
- verifies order 2 is pending in the live queue under the smart account, not the owner EOA or relayer.

No transaction is broadcast to a public network. The RPC is used only as read-only backing for Foundry's local fork.

### Fresh-clone reproduction

The repository documents Foundry v1.5.1 and Solidity 0.8.35 for CI. The clean-clone verification described below was also successfully run with `forge 1.0.0-stable` and Solc 0.8.35.

1. Clone the audit repository with submodules and check out the pinned commit:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
git rev-parse HEAD
```

Expected revision:

```text
a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

2. Create `test/fork/OpenIncreaseCarryCommitmentRelayerFork.t.sol` with the complete content below. This is the only file added.

<details>
<summary>test/fork/OpenIncreaseCarryCommitmentRelayerFork.t.sol</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../../packages/perps/test/utils/LegacyOrderRouterHarness.sol";
import {IEntryPoint, PackedUserOperation} from "@openzeppelin/contracts/interfaces/draft-IERC4337.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {Vm} from "forge-std/Vm.sol";

interface ICarryBypassAccountFactory {

    function createAccount(address owner, uint256 salt) external returns (address);

    function getAddress(address owner, uint256 salt) external view returns (address);

}

interface ICarryBypassAccount {

    struct Call {
        address target;
        uint256 value;
        bytes data;
    }

    function owner() external view returns (address);

    function execute(address target, uint256 value, bytes calldata data) external;

    function executeBatch(
        Call[] calldata calls
    ) external;

}

interface IEntryPointV08Hash {

    function getUserOpHash(
        PackedUserOperation calldata userOp
    ) external view returns (bytes32);

}

/// @notice End-to-end regression reproducer through the pinned SimpleAccount and EntryPoint v0.8 profile.
/// @dev The owner signs the exact increase before carry accrues. A distinct relayer submits that unchanged,
///      unsponsored UserOperation thirty days later. All perps state is reached through production entrypoints.
contract OpenIncreaseCarryCommitmentRelayerForkTest is BasePerpTest {

    address private constant ENTRY_POINT = 0x4337084D9E255Ff0702461CF8895CE9E3b5Ff108;
    address private constant ACCOUNT_FACTORY = 0x13E9ed32155810FDbd067D4522C492D6f68E5944;
    address private constant ACCOUNT_IMPLEMENTATION = 0x28426d752372D68d34340bd94390950DcE3C9ec3;
    bytes32 private constant ACCOUNT_CODE_HASH = 0x41ee894da413cc99e8dec0a1784470eceb736845ad1591e06ff0ecdf0aca26c9;
    bytes32 private constant USER_OPERATION_EVENT =
        keccak256("UserOperationEvent(bytes32,address,address,uint256,bool,uint256,uint256)");

    uint256 private constant OWNER_KEY = 0xA11CE4337;
    address private constant RELAYER = address(0xB0B4337);
    address private constant KEEPER = address(0xBEEF);

    uint256 private constant PRICE = 80_000_000;
    uint256 private constant DEPOSIT = 100_000e6;
    uint256 private constant INITIAL_SIZE = 300_000e18;
    uint256 private constant INITIAL_MARGIN = 20_000e6;
    uint256 private constant INCREASE_SIZE = 100_000e18;
    uint256 private constant INCREASE_MARGIN = 1000e6;

    uint128 private constant MAX_FEE_PER_GAS = 10 gwei;
    uint128 private constant MAX_PRIORITY_FEE_PER_GAS = 1 gwei;

    IEntryPoint private entryPoint;
    address private account;
    address private accountOwner;
    uint256 private accountSalt;

    function setUp() public override {
        vm.createSelectFork(vm.envString("ARBITRUM_SEPOLIA_RPC_URL"));
        assertEq(block.chainid, 421_614);
        assertEq(ENTRY_POINT.codehash, 0xe3f30f78ae55058acdefea00952c8e44f2263215cf720fe1b27b6f148add0278);
        assertEq(ACCOUNT_FACTORY.codehash, 0xa2e635152a61e180383c7afc045620b7461ef6f43ba27d592262513106b991b7);
        assertEq(ACCOUNT_IMPLEMENTATION.codehash, 0x689a90eff03926a12aedad2fc6d4fdbcbdd9ffac86e7d0d70ce6355961305c74);
        ArbitrumForkSupport.install();

        super.setUp();

        entryPoint = IEntryPoint(ENTRY_POINT);
        accountOwner = vm.addr(OWNER_KEY);
        accountSalt = uint256(
            keccak256(
                abi.encodePacked(
                    "OpenIncreaseCarryCommitmentRelayerForkTest",
                    block.chainid,
                    block.number,
                    blockhash(block.number - 1)
                )
            )
        );
        account = ICarryBypassAccountFactory(ACCOUNT_FACTORY).getAddress(accountOwner, accountSalt);
        assertEq(account.code.length, 0, "account must start counterfactual");

        settlementToken.mint(account, DEPOSIT);
        vm.deal(address(this), 2 ether);
        entryPoint.depositTo{value: 1 ether}(account);
        assertEq(accountOwner.balance, 0, "owner must not submit or fund the operation");
        assertEq(RELAYER.balance, 0, "relayer starts unfunded");
    }

    /// @dev Retains the shared economic fixture while using the exact production V3 Router implementation.
    function _deployLegacyOrderRouter(
        address engine_,
        address lens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predicted = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook book = new OrderLifecycleBook(predicted, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predicted);
        OrderRouter productionRouter = new OrderRouter(
            engine_,
            lens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(book)
        );
        assertEq(address(productionRouter), predicted);
        deployedRouter = LegacyOrderRouterHarness(address(productionRouter));
    }

    function testFork_RelayerSubmitsSignedIncreaseThatDebitsCarryBeyondBothBounds() public {
        OrderV3Types.OrderRequest memory initialOpen =
            _openRequest(bytes32("aa-initial-open"), INITIAL_SIZE, INITIAL_MARGIN, uint64(block.timestamp + 60));
        _submit(_unsignedOperation(_depositAndCommit(initialOpen)), RELAYER);

        assertEq(ICarryBypassAccount(account).owner(), accountOwner);
        assertEq(router.lifecycleBook().pendingIntent(1).account, account);
        assertEq(router.pendingOrderCounts(accountOwner), 0, "owner EOA must never become the trading account");

        bytes[] memory initialPriceUpdate = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory initialResult = router.executeOrder(1, initialPriceUpdate);
        assertEq(uint8(initialResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        (uint256 liveSize,,,, CfdTypes.Side liveSide,,) = engine.positions(account);
        assertEq(liveSize, INITIAL_SIZE);
        assertEq(uint8(liveSide), uint8(CfdTypes.Side.LONG));

        uint64 submitBy = uint64(block.timestamp + 30 days);
        OrderV3Types.OrderRequest memory increase =
            _openRequest(bytes32("aa-bounded-increase"), INCREASE_SIZE, INCREASE_MARGIN, submitBy);
        uint256 quotedBounty = _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE);
        assertEq(quotedBounty, router.maxOpenOrderExecutionBountyAssets());
        increase.bounds.maxExecutionBountyAssets = quotedBounty;
        increase.bounds.maxGrossAccountDebitAssets = quotedBounty;
        increase.bounds.maxActionChargeAssets = 0;
        emit log_named_uint("signedGrossDebitCapAssets", increase.bounds.maxGrossAccountDebitAssets);
        emit log_named_uint("signedActionChargeCapAssets", increase.bounds.maxActionChargeAssets);

        bytes memory exactWalletCall = abi.encodeCall(
            ICarryBypassAccount.execute, (address(router), 0, abi.encodeCall(OrderRouter.commitOrder, (increase)))
        );
        PackedUserOperation memory signedOperation = _signOperation(_unsignedOperation(exactWalletCall));
        bytes32 signedOperationHash = _userOpHash(signedOperation);

        vm.warp(submitBy);
        assertEq(_userOpHash(signedOperation), signedOperationHash, "relayer must submit exact owner-signed operation");

        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(account, CfdTypes.Side.LONG, INCREASE_SIZE, INCREASE_MARGIN, PRICE);
        assertTrue(preview.valid, "increase must be planner-valid after projected carry");

        uint256 projectedCarry = _expectedIndexedCarryAssets(account);
        emit log_named_uint("projectedCarryAssets", projectedCarry);
        assertGt(projectedCarry, increase.bounds.maxGrossAccountDebitAssets, "carry must exceed signed gross cap");

        uint256 accountBalanceBefore = clearinghouse.balanceAssets(account);
        uint256 poolAssetsBefore = pool.totalAssets();
        uint256 ownerNonceBefore = entryPoint.getNonce(account, 0);

        _submit(signedOperation, RELAYER);

        assertEq(entryPoint.getNonce(account, 0), ownerNonceBefore + 1, "relayer consumed signed wallet nonce");
        assertEq(router.nextCommitId(), 3, "signed increase must consume exactly one order id");

        uint256 accountDebit = accountBalanceBefore - clearinghouse.balanceAssets(account);
        uint256 poolCredit = pool.totalAssets() - poolAssetsBefore;
        emit log_named_uint("actualAccountDebitAssets", accountDebit);
        emit log_named_uint("housePoolCreditAssets", poolCredit);
        assertEq(accountDebit, projectedCarry, "reservation must checkpoint carry from smart account");
        assertEq(poolCredit, accountDebit, "checkpointed carry must reach HousePool");
        assertGt(accountDebit, increase.bounds.maxGrossAccountDebitAssets, "actual debit exceeds signed gross cap");
        assertGt(accountDebit, increase.bounds.maxActionChargeAssets, "actual charge exceeds signed action cap");

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        OrderV3Types.PendingIntent memory stored = book.pendingIntent(2);
        emit log_named_uint("storedCarryCollectedAssets", stored.commitment.carryCollectedAssets);
        assertEq(stored.account, account);
        assertEq(stored.executionBountyAssets, quotedBounty);
        assertEq(stored.commitment.carryCollectedAssets, 0, "book omits carry already taken from smart account");
        assertEq(stored.commitment.bountyFromFreeAssets, quotedBounty);
        assertEq(uint8(book.lifecycleStatus(2)), uint8(OrderV3Types.LifecycleStatus.Pending));

        (IOrderRouterAccounting.PendingOrderView memory queued,) = router.getPendingOrderView(2);
        assertEq(queued.orderId, 2);
        assertEq(queued.committedMarginAssets, INCREASE_MARGIN);
        assertEq(queued.executionBountyAssets, quotedBounty);
        assertEq(router.pendingOrderCounts(account), 1);
        assertEq(router.pendingOrderCounts(accountOwner), 0);
        assertEq(router.pendingOrderCounts(RELAYER), 0);
    }

    function _depositAndCommit(
        OrderV3Types.OrderRequest memory request
    ) private view returns (bytes memory) {
        ICarryBypassAccount.Call[] memory calls = new ICarryBypassAccount.Call[](3);
        calls[0] = ICarryBypassAccount.Call(
            address(settlementToken), 0, abi.encodeCall(IERC20.approve, (address(clearinghouse), DEPOSIT))
        );
        calls[1] = ICarryBypassAccount.Call(
            address(clearinghouse), 0, abi.encodeCall(MarginClearinghouse.deposit, (account, DEPOSIT))
        );
        calls[2] = ICarryBypassAccount.Call(address(router), 0, abi.encodeCall(OrderRouter.commitOrder, (request)));
        return abi.encodeCall(ICarryBypassAccount.executeBatch, (calls));
    }

    function _unsignedOperation(
        bytes memory callData
    ) private view returns (PackedUserOperation memory operation) {
        operation = PackedUserOperation({
            sender: account,
            nonce: entryPoint.getNonce(account, 0),
            initCode: account.code.length == 0
                ? abi.encodePacked(
                    ACCOUNT_FACTORY, abi.encodeCall(ICarryBypassAccountFactory.createAccount, (accountOwner, accountSalt))
                )
                : bytes(""),
            callData: callData,
            accountGasLimits: bytes32((uint256(450_000) << 128) | uint256(3_000_000)),
            preVerificationGas: 80_000,
            gasFees: bytes32((uint256(MAX_PRIORITY_FEE_PER_GAS) << 128) | uint256(MAX_FEE_PER_GAS)),
            paymasterAndData: bytes(""),
            signature: bytes("")
        });
    }

    function _signOperation(
        PackedUserOperation memory operation
    ) private returns (PackedUserOperation memory signedOperation) {
        signedOperation = operation;
        bytes32 operationHash = _userOpHash(signedOperation);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(OWNER_KEY, operationHash);
        signedOperation.signature = abi.encodePacked(r, s, v);
    }

    function _submit(PackedUserOperation memory operation, address relayer) private returns (Vm.Log[] memory logs) {
        if (operation.signature.length == 0) {
            operation = _signOperation(operation);
        }
        bytes32 operationHash = _userOpHash(operation);
        PackedUserOperation[] memory operations = new PackedUserOperation[](1);
        operations[0] = operation;

        uint256 depositBefore = entryPoint.balanceOf(account);
        uint256 beneficiaryBefore = relayer.balance;
        vm.recordLogs();
        vm.prank(relayer);
        entryPoint.handleOps(operations, payable(relayer));
        logs = vm.getRecordedLogs();

        uint256 operationCost = _assertSuccessfulOperation(logs, operationHash, operation.nonce);
        assertEq(depositBefore - entryPoint.balanceOf(account), operationCost, "smart account pays EntryPoint gas");
        assertEq(relayer.balance - beneficiaryBefore, operationCost, "relayer receives EntryPoint reimbursement");
        assertEq(account.codehash, ACCOUNT_CODE_HASH);
        assertEq(ICarryBypassAccount(account).owner(), accountOwner);
    }

    function _assertSuccessfulOperation(
        Vm.Log[] memory logs,
        bytes32 operationHash,
        uint256 nonce
    ) private view returns (uint256 operationCost) {
        uint256 matches;
        for (uint256 i; i < logs.length; ++i) {
            Vm.Log memory item = logs[i];
            if (
                item.emitter == ENTRY_POINT && item.topics.length == 4 && item.topics[0] == USER_OPERATION_EVENT
                    && item.topics[1] == operationHash && item.topics[2] == bytes32(uint256(uint160(account)))
                    && item.topics[3] == bytes32(0)
            ) {
                (uint256 recordedNonce, bool success, uint256 gasCost, uint256 gasUsed) =
                    abi.decode(item.data, (uint256, bool, uint256, uint256));
                assertEq(recordedNonce, nonce);
                assertTrue(success, "owner-signed wallet execution must succeed");
                assertGt(gasCost, 0);
                assertGt(gasUsed, 0);
                operationCost = gasCost;
                ++matches;
            }
        }
        assertEq(matches, 1, "exactly one successful UserOperationEvent");
    }

    function _userOpHash(
        PackedUserOperation memory operation
    ) private view returns (bytes32) {
        return IEntryPointV08Hash(ENTRY_POINT).getUserOpHash(operation);
    }

    function _openRequest(
        bytes32 clientOrderId,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint64 submitBy
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = PRICE;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: submitBy,
            executionWindowSeconds: 60,
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

}
```

</details>

3. Run the fork test from the repository root:

```bash
ARBITRUM_SEPOLIA_RPC_URL=https://arbitrum-sepolia-rpc.publicnode.com \
  forge test \
  --match-path test/fork/OpenIncreaseCarryCommitmentRelayerFork.t.sol \
  -vvv
```

The other supplied RPC URLs can be substituted if they expose the current Arbitrum Sepolia state:

```text
https://sepolia-rollup.arbitrum.io/rpc
https://arbitrum-sepolia.drpc.org
```

The test intentionally forks the provider's current head, derives a fresh counterfactual-account salt from that fork head, and asserts the deployed AA contracts' code hashes. The derived salt prevents a later public deployment of a previously published counterfactual address from breaking reproduction. Using the current head also avoids dependence on historical state that public RPC providers may prune, while the code-hash checks fail closed if the external account-abstraction code changes.

### Verified result

The command above was run from a fresh recursive clone at the pinned commit and produced:

```text
Ran 1 test for test/fork/OpenIncreaseCarryCommitmentRelayerFork.t.sol:OpenIncreaseCarryCommitmentRelayerForkTest
[PASS] testFork_RelayerSubmitsSignedIncreaseThatDebitsCarryBeyondBothBounds() (gas: 4623676)
Logs:
  signedGrossDebitCapAssets: 200000
  signedActionChargeCapAssets: 0
  projectedCarryAssets: 199026792
  actualAccountDebitAssets: 199026792
  housePoolCreditAssets: 199026792
  storedCarryCollectedAssets: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped

Ran 1 test suite: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

