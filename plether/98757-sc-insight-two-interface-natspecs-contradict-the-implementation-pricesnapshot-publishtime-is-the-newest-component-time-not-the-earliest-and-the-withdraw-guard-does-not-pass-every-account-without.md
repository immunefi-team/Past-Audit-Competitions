# \#98757 [SC-Insight] Two interface NatSpecs contradict the implementation: `PriceSnapshot.publishTime` is the newest component time, not the earliest, and the withdraw guard does not pass every account without a position

**Submitted on Nov 10th 2026 at 02:25:58 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98757
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Affected contracts (in-scope commit `a82d4fef`; live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release Router configuration and Pyth mocked. No in-scope contract is modified.

| Contract / interface | Arbitrum Sepolia v2.0.0 address | Role in the insight |
|---|---|---|
| `IPletherOracle` (`PriceSnapshot`, `LiquidationBatchSnapshot`) | interface | Documents `publishTime` as the "Earliest component publish time" (lines 31 and 53) |
| `PletherOracle` | `0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2` | Sets `basket.publishTime` to the **newest** component time (lines 874–875) and copies it into every snapshot (line 889) |
| `IWithdrawGuard` | interface | Documents "Accounts without a position pass" (line 8) |
| `CfdEngine` / `CfdEngineSettlementSidecar` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` / `0xd3ede337e042d5aa6ba68e03d0c885b3d4a062dc` | `checkWithdraw` → `validateWithdraw` refuses a flat account's withdrawal below its pending order's settlement floor (sidecar lines 255–271) |
| `MarginClearinghouse` | `0xb5b851f627d6df3cf3f096e23e20a58884d7b89a` | `withdraw` / `withdrawMargin` call the guard |

Release configuration that bounds the publish-time gap:
- historical order-execution baskets are limited by `maxComponentPublishTimeDivergence`, which is 5 s (`PletherOracle.sol` line 90, the `DEPLOYMENT.md` Router values, and the proof of concept's log);
- live baskets parse every component inside `LIVE_TICK_LOOKBACK` = 15 s (line 29), with a divergence limit of `orderExecutionStalenessLimit` = 60 s, or `liquidationStalenessLimit` = 15 s for liquidations (`_maxPublishTimeDivergence`, lines 993–999).

## Brief/Intro

This is an **Insight**: two interface comments that integrators, keepers and monitors rely on describe the opposite of what the code does. The protocol's own checks are not affected.

1. **Publish time.** `IPletherOracle.PriceSnapshot.publishTime` and `LiquidationBatchSnapshot.publishTime` are documented as the "Earliest component publish time". `PletherOracle` returns the **newest** component time, as `ORACLE_SYNCHRONIZATION.md` line 26 says ("Its publish time is the newest component time"). A consumer that reads `publishTime` as a lower bound on every component's age overestimates the freshness of the oldest component. That component can be up to 5 s older on order-execution baskets and up to the live lookback on live baskets. In the proof of concept, the oldest component is at commit + 1 s, the newest at commit + 3 s, and `publishTime` is commit + 3 s, so the oldest component is 2 s older than the reported time.
2. **Withdraw guard.** `IWithdrawGuard.checkWithdraw` says "Accounts without a position pass." `validateWithdraw` refuses a flat account's withdrawal that would take its balance below the settlement floor of its pending open (`minPostSettlementBalanceAssets` plus `maxGrossAccountDebitAssets`). In the proof of concept, a flat account with 10,000 USDG and a pending open (floor 1,000 USDG, maximum debit 1,000 USDG) is refused an 8,500 USDG withdrawal and allowed an 8,000 USDG withdrawal.

## Vulnerability Details

### `publishTime` is documented as the earliest component time

`IPletherOracle.sol` line 31 (in `PriceSnapshot`) and line 53 (in `LiquidationBatchSnapshot`):

```solidity
/// @param publishTime Earliest component publish time as a Unix timestamp.
```

### `PletherOracle` sets it to the newest component time

`PletherOracle.sol` lines 848–875 (`_computeBasketPriceFromFeeds`) track both bounds and store the maximum as the basket time:

```solidity
if (p.publishTime < minPublishTime) {
    minPublishTime = p.publishTime;
}
if (p.publishTime > maxPublishTime) {
    maxPublishTime = p.publishTime;
}
...
if (maxPublishTime > minPublishTime + maxPublishTimeDivergence) {
    revert PletherOracle__PublishTimeDivergence(mode, minPublishTime, maxPublishTime, maxPublishTimeDivergence);
}
...
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

`_snapshotFromBasket` copies `basket.publishTime` into `snapshot.publishTime` (line 889). `oldestPublishTime` is used internally (for example by the batch cache) but is not exposed in `PriceSnapshot` or `LiquidationBatchSnapshot`. `ORACLE_SYNCHRONIZATION.md` line 26 describes the implemented behaviour: "Accept the entire basket together. Its publish time is the newest component time".

### The withdraw guard is documented to pass every account without a position

`IWithdrawGuard.sol` lines 7–11:

```solidity
/// @notice Validates an account after the clearinghouse has provisionally debited a withdrawal.
/// @dev Callable only by the configured clearinghouse. Accounts without a position pass. Open positions require
///      non-degraded mode and a cached mark fresh under the active live or frozen policy; ...
```

### `validateWithdraw` refuses a flat account below its pending order's floor

`CfdEngineSettlementSidecar.sol` lines 255–271:

```solidity
function validateWithdraw(
    address account
) external onlyEngine {
    ...
    CfdTypes.Position memory pos = _loadPosition(engine, account);
    if (pos.size == 0) {
        (uint64 orderId, uint256 minimum, uint256 maxDebit) =
            OrderSettlementFloorLib.load(engine.orderRouter(), account);
        if (minimum != 0) {
            uint256 balance = IMarginClearinghouse(host.clearinghouse()).balanceAssets(account);
            if (balance < minimum || balance - minimum < maxDebit) {
                revert IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal(
                    orderId, balance, minimum, maxDebit
                );
            }
        }
        return;
    }
```

`OrderSettlementFloorLib.load` reads the account's head pending order and returns its `minPostSettlementBalanceAssets` and `maxGrossAccountDebitAssets` bounds. The behaviour is sensible, because it protects the pending order's funding. Only the interface comment is wrong.

### Why it matters, and why there is no in-protocol fund impact

The oracle enforces the component divergence limit itself, and every internal consumer uses the correct semantics. Order reuse uses `oldestPublishTime`, and freshness checks run per component. So neither discrepancy moves funds inside the protocol. The risk is external:
- **Freshness monitors and keepers.** A keeper that skips a basket "older than N seconds" by reading `publishTime` as the earliest component time will accept baskets whose oldest component is older than N.
- **Integrators.** An integrator that bounds oracle age off-chain the same way gets the same wrong answer.
- **Wallets and front ends.** A wallet or front end that relies on "accounts without a position pass" will show a flat account's full free balance as withdrawable and then hit `OrderRouter__SettlementFloorWithdrawal`.

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack with a six-component basket oracle (the repository's oracle-synchronization fixture), installed through the owner's timelocked oracle-config path. It applies the release Router configuration and calls only public functions. `MockPyth` supplies signed prices.

`test_PublishTimeIsNewestComponentNotEarliest`:
1. A historical order-execution basket is resolved through `PletherOracle.updateBatchOrderExecutionPrice` for commit time `c`. Components 1–2 are published at `c + 1`, components 3–4 at `c + 2`, and components 5–6 at `c + 3`.
2. The returned `PriceSnapshot.publishTime` is `c + 3`, the newest component. The oldest component (`c + 1`) is 2 s older than the reported time.
3. The live limits read from the oracle: `maxComponentPublishTimeDivergence` 5 s, `orderExecutionStalenessLimit` 60 s, `liquidationStalenessLimit` 15 s.

`test_FlatAccountWithPendingOpenIsNotPassedByWithdrawGuard`:
1. A trader with 10,000 USDG of settlement and no position commits a 2,000 USDG LONG open with `minPostSettlementBalanceAssets` = 1,000 USDG and `maxGrossAccountDebitAssets` = 1,000 USDG. The order is `Pending`, and the account has no position.
2. `withdrawMargin(8,500 USDG)` reverts `OrderRouter__SettlementFloorWithdrawal(orderId, 1,500, 1,000, 1,000)`.
3. `withdrawMargin(8,000 USDG)` succeeds.

## Impact Details

**Insight.** This report concerns documentation that contradicts the code. It claims no in-protocol loss, freeze or theft: the oracle enforces divergence itself, and the withdraw guard's behaviour protects pending orders. The value is for consumers of the interfaces:
- a consumer that treats `publishTime` as the earliest component time overestimates basket freshness by up to the divergence limit;
- a consumer that treats a flat account as unconditionally withdrawable mis-displays balances and builds transactions that revert.

## Recommendation

1. Correct `IPletherOracle.sol` lines 31 and 53 to "Newest component publish time", matching `ORACLE_SYNCHRONIZATION.md` line 26.
2. Optionally expose the oldest component time, or both times, in `PriceSnapshot` and `LiquidationBatchSnapshot` (the value is already computed as `basket.oldestPublishTime`), so that external consumers can bound the age of every component.
3. Correct `IWithdrawGuard.sol` line 8, for example: "Accounts without a position pass unless the withdrawal would leave the balance below the settlement floor (`minPostSettlementBalanceAssets` plus `maxGrossAccountDebitAssets`) of their pending head order."

## Proof of Concept

Per the program's Insight rules, runnable code is not required, and this section describes when the insight is valuable.
- **Publish time.** It matters to any off-chain or on-chain consumer that computes basket freshness from `PriceSnapshot.publishTime` or `LiquidationBatchSnapshot.publishTime`, whenever the basket's components were not all published in the same second. Order-execution baskets admit up to 5 s of spread, and live baskets up to the 15 s lookback.
- **Withdraw guard.** It matters to any wallet, front end or integrator that shows withdrawable balances or builds withdrawals for a flat account with a pending open whose `minPostSettlementBalanceAssets` is nonzero.

A runnable test is included anyway. Save it as `packages/perps/test/perps/poc/OracleTimeAndWithdrawGuardDocs.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract OracleTimeAndWithdrawGuardDocsPoC -vv
```

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Insight: two interface NatSpecs that do not match the implementation.
///         1. `PriceSnapshot.publishTime` is documented as the "Earliest component publish time"; PletherOracle returns
///            the NEWEST component time, so the oldest component can be older than the reported time.
///         2. `IWithdrawGuard.checkWithdraw` says "Accounts without a position pass"; a flat account with a pending
///            open whose `minPostSettlementBalanceAssets` is nonzero is refused a withdrawal below that floor.
///         Public calls only; MockPyth supplies signed prices; release Router configuration.
contract OracleTimeAndWithdrawGuardDocsPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);
    uint256 internal constant FEE = 1 gwei;
    int64 internal constant PRICE = 100_000_000;
    uint64 internal constant CONF = 50_000;

    uint256 internal orderNonce;

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

    function setUp() public override {
        super.setUp();
        bytes32[] memory ids = _feedIds();
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        for (uint256 i; i < 6; ++i) {
            weights[i] = i == 5 ? 0.5e18 : 0.1e18;
            bases[i] = 1e8;
        }
        baseMockPyth.setAllPrices(ids, PRICE, -8, SETUP_TIMESTAMP);
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids, weights, bases, new bool[](6)
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(pletherOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        _applyReleaseRouterConfig(); // 72 h timelock
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live market
        vm.roll(block.number + 1);
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        baseMockPyth.setFee(FEE);
        vm.deal(KEEPER, 10 ether);
        _fundTrader(ALICE, 10_000e6);
        uint64 m = uint64(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 10 * FEE}(_window(m - 1, m, m));
        _advance(m + 1);
    }

    /// @notice 1. The reported `publishTime` is the newest component time, not the earliest.
    function test_PublishTimeIsNewestComponentNotEarliest() external {
        uint64 c = uint64(vm.getBlockTimestamp());
        _advance(c + 4);
        // Components: feeds 1-2 at c+1 (oldest), feeds 3-4 at c+2, feeds 5-6 at c+3 (newest).
        bytes32[] memory ids = _feedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(PRICE, CONF, -8, c + 1 + uint64(i / 2), c));
        }
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(updates);
        IPletherOracle.BatchOrderPriceCache memory cache;
        vm.prank(KEEPER);
        (bool ok, IPletherOracle.PriceSnapshot memory snap,) = pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(
            KEEPER, data, IPletherOracle.OrderExecutionRequest(c, 0, CfdTypes.Side.LONG, false, false), cache
        );
        assertTrue(ok, "basket not resolved");
        uint256 oldest = c + 1;
        uint256 newest = c + 3;
        emit log_named_uint("oldest component publish time (offset from commit, s)", oldest - c);
        emit log_named_uint("newest component publish time (offset from commit, s)", newest - c);
        emit log_named_uint("PriceSnapshot.publishTime (offset from commit, s)", snap.publishTime - c);
        assertEq(snap.publishTime, newest, "publishTime is not the newest component");
        assertTrue(snap.publishTime != oldest, "publishTime is the earliest component");
        emit log_named_uint("oldest component is older than the reported publishTime by (s)", snap.publishTime - oldest);
        assertEq(snap.publishTime - oldest, 2, "age gap");
        emit log_named_uint("maxComponentPublishTimeDivergence (order execution, s)", pletherOracle.maxComponentPublishTimeDivergence());
        emit log_named_uint("orderExecutionStalenessLimit (s)", pletherOracle.orderExecutionStalenessLimit());
        emit log_named_uint("liquidationStalenessLimit (s)", pletherOracle.liquidationStalenessLimit());
    }

    /// @notice 2. A flat account with a pending open is refused a withdrawal below the order's settlement floor.
    function test_FlatAccountWithPendingOpenIsNotPassedByWithdrawGuard() external {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", ALICE, ++orderNonce));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 2_000e18;
        request.marginDelta = 50e6;
        request.targetPrice = 1;
        request.bounds = _bounds();
        request.bounds.minPostSettlementBalanceAssets = 1_000e6;
        request.bounds.maxGrossAccountDebitAssets = 1_000e6;
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);
        assertEq(_size(ALICE), 0, "account is not flat");
        assertEq(
            uint256(router.lifecycleBook().lifecycleStatus(orderId)), uint256(OrderV3Types.LifecycleStatus.Pending)
        );
        uint256 balance = clearinghouse.balanceAssets(ALICE);
        emit log_named_decimal_uint("flat account settlement balance (USDG)", balance, 6);
        emit log_named_decimal_uint("pending open minPostSettlementBalanceAssets (USDG)", 1_000e6, 6);
        emit log_named_decimal_uint("pending open maxGrossAccountDebitAssets (USDG)", 1_000e6, 6);

        uint256 tooMuch = balance - 1_500e6; // leaves 1,500: 1,500 - 1,000 < 1,000
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal.selector, orderId, 1_500e6, 1_000e6, 1_000e6
            )
        );
        clearinghouse.withdrawMargin(tooMuch);
        emit log_named_decimal_uint("withdrawal refused although the account has no position (USDG)", tooMuch, 6);

        uint256 allowed = balance - 2_000e6; // leaves 2,000: 2,000 - 1,000 >= 1,000
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(allowed);
        emit log_named_decimal_uint("withdrawal above the floor accepted (USDG)", allowed, 6);
    }

    /// @dev One signed tick per feed: feeds 1-5 at `fiveAt`, feed 6 at `sixthAt`, all with `prev` as previous time.
    function _window(
        uint64 prev,
        uint64 fiveAt,
        uint64 sixthAt
    ) internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(_updates(prev, fiveAt, sixthAt));
    }

    function _updates(
        uint64 prev,
        uint64 fiveAt,
        uint64 sixthAt
    ) internal pure returns (MockPyth.FeedUpdate[] memory updates) {
        bytes32[] memory ids = _feedIds();
        updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint64 at = i == 5 ? sixthAt : fiveAt;
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(PRICE, CONF, -8, at, prev));
        }
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = bytes32(i + 1);
        }
    }


    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _advance(
        uint64 t
    ) internal {
        vm.warp(t);
        vm.roll(block.number + 1);
    }



    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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

    /// @dev Live v2.0.0 Router configuration, applied through the owner's 72 h timelocked config path.
    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

}
```

</details>

### Observed balances:

| Measurement | Documented | Observed |
| --- | --- | ---: |
| Oldest component publish time (offset from commit) | — | 1 s |
| Newest component publish time (offset from commit) | — | 3 s |
| `PriceSnapshot.publishTime` (offset from commit) | earliest component: 1 s | **3 s** (newest) |
| Oldest component older than the reported `publishTime` | 0 s | **2 s** |
| `maxComponentPublishTimeDivergence` (order execution) | — | 5 s |
| `orderExecutionStalenessLimit` / `liquidationStalenessLimit` | — | 60 s / 15 s |
| Flat account settlement balance | — | 10,000.000000 USDG |
| Pending open floor / maximum debit | — | 1,000.000000 / 1,000.000000 USDG |
| Withdrawal of 8,500.000000 USDG by the flat account | passes ("Accounts without a position pass") | **reverts** `OrderRouter__SettlementFloorWithdrawal` |
| Withdrawal of 8,000.000000 USDG by the flat account | passes | passes |
