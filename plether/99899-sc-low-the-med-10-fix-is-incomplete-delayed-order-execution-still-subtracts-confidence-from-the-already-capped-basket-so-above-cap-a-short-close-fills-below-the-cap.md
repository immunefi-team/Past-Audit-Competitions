# \#99899 [SC-Low] The MED-10 fix is incomplete: delayed-order execution still subtracts confidence from the already-capped basket, so above CAP a SHORT close fills below the cap

**Submitted on Nov 15th 2026 at 17:19:30 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99899
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value". If the judge reads it as informational, it is the Insight.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/PletherOracle.sol`: `_updateOrderExecutionPrice` (lines 557-558), `updateBatchOrderExecutionPrice` (lines 283-284), `_snapshotFromBasket` (line 887) and `_orderExecutionPrice` (line 940).

**Relation to prior findings:** the fix for Cecuro **MED-10** is incomplete. The maintainers' MED-10 comment says: "Single-account and batch liquidation pricing now apply the directional confidence adjustment to the raw, uncapped basket, then clamp the resulting liquidation price to CAP. … This removes the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark." Delayed-order execution, single and batch, still subtracts confidence from the already capped basket. The program's known-issues rule makes a report on a fixed root valid when it shows the fix "is incomplete". The section "Relation to prior findings" also covers the order-path NatSpec and pre-audit SC-M6.

## Summary

Cecuro MED-10 found that liquidation pricing capped the neutral basket at `CAP_PRICE` first and then subtracted the adverse confidence shift, so above the cap a SHORT was priced at `CAP − shift` instead of `min(raw − shift, CAP)`. The maintainers fixed it for liquidation only (`updateLiquidationPrice`, `updateLiquidationBatchPrice`, `PletherOracle.sol:308-343`). Their comment calls the old result an "artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark".

Delayed-order execution has the same ordering. Both order resolvers build the snapshot with `_snapshotFromBasket`, which caps the basket (`PletherOracle.sol:887`), and then pass that **capped** price to `_orderExecutionPrice`, which subtracts the shift and clamps again (`PletherOracle.sol:940`). For an adverse-down order (a SHORT close, or a LONG open) with the raw basket above the cap, the fill is `CAP − shift`. With the MED-10 ordering it would be `min(raw − shift, CAP)`.

The PoC runs through public entry points only. A trader opens a SHORT at 1.00 and the basket moves to 2.50 (CAP is 2.00). With per-feed confidence at the oracle's accepted ceiling (10 bps), the shift is 0.0005. The close fills at **1.9995** instead of 2.00, although `raw − shift` = 2.4995 is far above the cap. The trader ends **4.998 USDC** worse off on a 10,000-unit SHORT than the same close with zero confidence, which fills at the cap. The batch executor gives the identical result. At MED-10's own example input (raw 2.0002, one basis point above the cap, confidence 0.002), the order path fills at 1.9996, while the MED-10 ordering gives 1.9998.

The amount is small: at most `min(raw − CAP, shift)` per unit, and the shift is at most 2 bps of the raw basket at the default parameters. The value is not lost from the system; it goes to the LP side as trade PnL. The Impact section states the limits, and the strongest objection is answered under "Relation to prior findings".

## Relation to prior findings

**Cecuro MED-10** (Acknowledged, "Addressed"), "Liquidator can liquidate short traders at prices below the configured cap-adjusted mark". The finding names `updateLiquidationPrice` and `updateLiquidationBatchPrice` and describes the root: they "first clamp the neutral basket inside `_snapshotFromBasket`, then apply the adverse confidence shift to that already-capped number". The remediation asks to "apply the liquidation confidence shift to the pre-cap basket value and clamp only once afterwards", and to "avoid feeding `snapshot.price` back into the … shift helpers". The fix (`eadc5c58`, #124 in this repository; Cecuro's comment links plether-core `006c818d`) added `_updateLiquidationSnapshot`, which returns the raw basket, and changed only the two liquidation entry points. The order resolvers still feed `snapshot.price` (capped) into `_orderExecutionPrice` (`PletherOracle.sol:284`, `:558`). It is the same root, at call sites the finding did not name and the fix did not touch.

After the fix, liquidation and order execution of the same SHORT at the same tick are priced differently: liquidation at `min(raw − shift, CAP)` (`PletherOracle.sol:317`, `:342`), and a voluntary close at `CAP − shift` (`PletherOracle.sol:940`). `SECURITY.md:425` describes one rule for both: "side-adverse pricing for execution, equity checks, and liquidation".

**The strongest objection: the order-path ordering is documented.** The NatSpec of `updateOrderExecutionPrice` says "The base basket is capped and then shifted against the requested side using aggregate confidence" (`PletherOracle.sol:209`). `SECURITY.md:653-654` speaks of "a downward adjustment that exhausts the capped execution base". Before the MED-10 fix, the liquidation NatSpec already said the opposite ("shifts the price … by the configured fraction of aggregate confidence, then caps it at `CAP_PRICE`"), so MED-10 was code against its own documentation, and the fix did not change the order-path text. A judge may therefore read the order-path ordering as intended. Our answer: the maintainers' reason for the fix is not about documentation. It is about the price: subtracting confidence from an already capped mark produces an "artificial below-cap" SHORT price. That holds for a SHORT close exactly as for a SHORT liquidation. When `raw − shift` is above the cap, the side-adverse price is itself above the cap, so the cap-bounded adverse price is the cap: shifting first and capping second leaves nothing to charge. The NatSpec describes the order the code runs in. It does not give a reason for charging a below-cap price in that case. If the judge holds that the documented order-path ordering is intended, this report is the Insight: the MED-10 rationale applies to delayed orders, and the fix left them on the old ordering.

**Pre-audit SC-M6** ("If Price Cap is Reached Protocol Becomes Unusable", Acknowledged) is the regime this report needs (the basket at or above `CAP_PRICE` = 2e8), not its root. SC-M6 is about `CAP_PRICE` being immutable and payoffs being bounded at the cap. It says nothing about the order of the cap and the confidence shift. MED-10 was accepted and fixed in the same regime, with an example one basis point above the cap. The PoC runs that exact input on the order path.

**The frozen path is not part of this report.** While the oracle is frozen, voluntary closes are left unshifted (`PletherOracle.sol:937-939`), and the frozen basket is the already capped Engine mark, so there is no raw basket to shift.

The Cantina and collaborative audit reports cover the spot package and contain nothing on this path. The AI scan has no CAP/confidence-ordering item.

## Vulnerability details

Both order resolvers cap, then shift. The single resolver (`PletherOracle.sol:545-563`):

```solidity
        snapshot = _snapshotFromBasket(basket, policy, false);
        snapshot.price = _orderExecutionPrice(request, snapshot.price, basket.confidence, policy.oracleFrozen);
```

The batch resolver has the same two lines (`PletherOracle.sol:283-284`). `_snapshotFromBasket` caps the price (`PletherOracle.sol:887`):

```solidity
        snapshot.price = _clampToCap(basket.price);
```

`_orderExecutionPrice` then subtracts the shift from that capped value and clamps again (`PletherOracle.sol:925-941`, via `_adverseOrderPrice`, `:905-923`; `shift = confidence × adverseConfidenceMultiplierBps / 10_000`, and the direction is down for a SHORT close and a LONG open):

```solidity
        if (oracleFrozen && request.isClose) {
            return price;
        }
        return _clampToCap(_adverseOrderPrice(request, price, confidence));
```

The liquidation entry points after the MED-10 fix shift the raw basket first (`PletherOracle.sol:340-342`):

```solidity
        (snapshot, rawPrice, confidence) = _updateLiquidationSnapshot(pythUpdateData);
        snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));
```

With `raw > CAP`, the order path returns `CAP − shift` for an adverse-down order. The MED-10 ordering returns `min(raw − shift, CAP)`. The difference per unit is `min(raw − CAP, shift)`. The router executes the order at the resolver's price: in the PoC the trader's settlement balance after the keeper's `executeOrder` (and `executeOrderBatch`) differs from the zero-confidence run by the shift times the size, less the matching change in notional-based charges.

## Impact

**Who:** any trader holding a SHORT who closes it (or any trader opening a LONG) while the raw basket is above `CAP_PRICE`. No role is needed and no third party is involved. The trader's own order pays.

**What the contract fails to deliver:** a SHORT's price PnL is bounded by "the price interval from zero to `CAP_PRICE`" (`SECURITY.md:341`), and the adverse confidence shift is meant to price uncertainty in the basket. When `raw − shift` is above the cap, the adverse price is still the cap, which is the return the MED-10 ordering pays for the same tick. The order path pays `CAP − shift` instead, which the maintainers' own MED-10 comment calls an "artificial below-cap SHORT … price". In the PoC the trader receives 4.998 USDC less on a 10,000-unit SHORT (about $20,000 of notional at the cap). Nothing is lost from the system: the difference stays with the LP side as trade PnL. This matches the program's Low impact "Contract fails to deliver promised returns, but doesn't lose value".

**Limits, stated plainly:**
- It needs the raw basket at or above `CAP_PRICE` (2e8, that is $2.00, twice a 1e8 basket; `CAP_PRICE` is immutable). That is the regime of the acknowledged SC-M6, and the one MED-10 was accepted and fixed in.
- The amount per unit is `min(raw − CAP, shift)`. At the default parameters (`basketMaxConfidenceRatioBps` 10, `PletherOracle.sol:84`, enforced at `:869`; `adverseConfidenceMultiplierBps` 2000, `:95`) the shift is at most 2 bps of the raw basket. Measured: 4.998 USDC per 10,000 units at raw 2.50 (2.5 bps of the notional at the cap), and 3.998 USDC per 10,000 units at MED-10's input, of which about 2.0 USDC is beyond what the MED-10 ordering would charge.
- Each order pays only for itself. Nobody else can trigger it or profit from it, and it does not change liquidation eligibility.
- A trader can avoid it by closing while the oracle is frozen (voluntary closes are unshifted, `PletherOracle.sol:937-939`). That is avoidance, not recovery: an executed spread is not returned.
- By source, a LONG open above the cap takes the same downward branch. The PoC executes only the SHORT close.

**If the judge reads this as informational**, it is the Insight: the MED-10 rationale applies to delayed-order execution, and the fix left those resolvers on the old ordering.


## Recommendation

Apply MED-10's ordering in both order resolvers: pass the raw basket, not the capped snapshot price, to `_orderExecutionPrice`, which already clamps once at the end (`PletherOracle.sol:940`).

```solidity
// _updateOrderExecutionPrice (:557-558) and updateBatchOrderExecutionPrice (:283-284)
snapshot = _snapshotFromBasket(basket, policy, false); // neutral mark stays capped
snapshot.price = _orderExecutionPrice(request, basket.price, basket.confidence, policy.oracleFrozen);

// _orderExecutionPrice: keep the frozen-close early return capped as well
if (oracleFrozen && request.isClose) {
    return _clampToCap(price);
}
return _clampToCap(_adverseOrderPrice(request, price, confidence));
```

While frozen, `basket.price` is the Engine mark, which is already capped, so frozen pricing does not change. Update the NatSpec at `PletherOracle.sol:208-210` to "shifted against the requested side and then capped". A regression test like `test_Property_ShortCloseAboveCapSettlesAtCap` above covers it. This is a sketch; it has not been run against the maintainers' suite.


## Link to Proof of Concept

https://gist.github.com/Richter2552/d8477246d05451e2d6e6c6073b66898d

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `PositionProtectionTest` fixture. After the fixture's `setUp` it uses only public entry points from unprivileged accounts: production `OrderRouter.commitOrder(OrderRequest)` from the trader's own account, `router.executeOrder` and `router.executeOrderBatch` from an address with no role, the permissionless `router.updateMarkPrice`, `HousePool.settleLpEpoch`, and `TrancheVault` request/claim. To read the execution price it calls the oracle's unguarded `updateOrderExecutionPrice` from an unrelated address inside a state snapshot that is reverted before the keeper executes. Pyth is the maintainers' `MockPyth` (Pyth is outside the scope). No protocol contract is pranked and no storage is written. The fixture's junior deposit helper is switched off (`_initialJuniorDeposit() = 0`) because it impersonates the router. The junior LP deposits through the public vault flow instead. Because of that override, the tests inherited from `PositionProtectionTest` are not meant to run in this contract, so run only the tests named below.

The **property** test states the behaviour the code should have and **fails** on the pinned code. The **control** and **characterize** tests pass and pin the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card9850S1CapThenShift.t.sol packages/perps/test/perps/spec/trader/
cd packages/perps

forge test --match-contract '^Card9850S1CapThenShiftTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card9850S1CapThenShiftTest$' --match-test '^test_(Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35, via-IR, the repository's remappings). Result: the control and the three characterize tests pass, and the property test fails. Excerpt (forge `-vv` logs; prices in 8 decimals, USDC amounts in 1e6 units):

```console
[PASS] test_Characterize_BatchExecutionSameOrdering() (gas: 23329013)
Logs:
  CAP / raw basket / per-feed conf 200000000 250000000 250000
  adverseConfidenceMultiplierBps / shift (8dp) 2000 50000
  SHORT close price conf=0 / conf>0 200000000 199950000
  MED-10 ordering min(raw - shift, CAP) 200000000
  ALICE balance conf=0 / conf>0 29987799999 29982801999
  ALICE worse off vs conf=0 (USDC 6dp) 4998000
  ALICE balance single / batch (conf>0) 29982801999 29982801999

[PASS] test_Characterize_Med10ExampleInputOnOrderPath() (gas: 15541076)
Logs:
  CAP / raw basket / per-feed conf 200000000 200020000 200000
  adverseConfidenceMultiplierBps / shift (8dp) 2000 40000
  SHORT close price conf=0 / conf>0 200000000 199960000
  MED-10 ordering min(raw - shift, CAP) 199980000
  ALICE balance conf=0 / conf>0 29987799999 29983801599
  ALICE worse off vs conf=0 (USDC 6dp) 3998400

[PASS] test_Characterize_ShortCloseAboveCapPricedAtCapMinusShift() (gas: 15543447)
Logs:
  CAP / raw basket / per-feed conf 200000000 250000000 250000
  adverseConfidenceMultiplierBps / shift (8dp) 2000 50000
  SHORT close price conf=0 / conf>0 200000000 199950000
  MED-10 ordering min(raw - shift, CAP) 200000000
  ALICE balance conf=0 / conf>0 29987799999 29982801999
  ALICE worse off vs conf=0 (USDC 6dp) 4998000
  shift x size (USDC 6dp) 5000000

[PASS] test_Control_ZeroConfidenceShortCloseAboveCapSettlesAtCap() (gas: 7739546)
Logs:
  SHORT close price conf=0 / CAP 200000000 200000000
  ALICE balance after close (conf=0) 29987799999

[FAIL: SHORT close priced below CAP although raw - shift > CAP: 199950000 != 200000000] test_Property_ShortCloseAboveCapSettlesAtCap() (gas: 16630504)
Logs:
  CAP / raw basket / per-feed conf 200000000 250000000 250000
  adverseConfidenceMultiplierBps / shift (8dp) 2000 50000
  SHORT close price conf=0 / conf>0 200000000 199950000
  MED-10 ordering min(raw - shift, CAP) 200000000
  ALICE balance conf=0 / conf>0 29987799999 29982801999
  ALICE worse off vs conf=0 (USDC 6dp) 4998000

Suite result: FAILED. 4 passed; 1 failed; 0 skipped; finished in 33.70ms (85.02ms CPU time)
```

Expected output on a fixed contract: the property test passes. `test_Characterize_ShortCloseAboveCapPricedAtCapMinusShift` and `test_Characterize_Med10ExampleInputOnOrderPath` then fail, because they pin the `CAP − shift` price. The control and the batch test still pass.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| control: ALICE opens a SHORT at 1.00 (through `commitOrder` + keeper `executeOrder`), anyone refreshes the basket to 2.50 (the engine stores CAP 2.00), ALICE closes with zero confidence: the close is priced at CAP | `test_Control_ZeroConfidenceShortCloseAboveCapSettlesAtCap` | passes |
| the same round trip with per-feed confidence at the 10 bps ceiling (shift 0.0005, `raw − shift` = 2.4995 > CAP) should also close at CAP, with the same balance | `test_Property_ShortCloseAboveCapSettlesAtCap` | **FAILS**: priced 199950000, ALICE 4.998000 USDC worse |
| the close is priced at exactly `CAP − shift`, and ALICE's shortfall is above 0 and at most `shift × size` (5.000000 USDC) | `test_Characterize_ShortCloseAboveCapPricedAtCapMinusShift` | passes |
| the batch executor (`executeOrderBatch`, the batch resolver) settles the close exactly as `executeOrder` does, below the zero-confidence result | `test_Characterize_BatchExecutionSameOrdering` | passes |
| MED-10's own example input (raw 2.0002, confidence 0.002): the order path prices the close at 1.9996, below the MED-10 ordering's 1.9998 | `test_Characterize_Med10ExampleInputOnOrderPath` | passes |

### Test file: `packages/perps/test/perps/spec/trader/Card9850S1CapThenShift.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title Order execution shifts the already-capped basket (the MED-10 fix covers liquidation pricing only)
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to PositionProtection.t.sol in
///         packages/perps/test/perps/spec/trader/ and run with the repository's own foundry.toml.
/// @dev The MED-10 fix makes liquidation pricing shift the RAW basket by the adverse confidence and clamp afterwards
///      (`updateLiquidationPrice` / `updateLiquidationBatchPrice`, PletherOracle.sol:308-343). Delayed-order execution
///      still clamps first (`_snapshotFromBasket`, :878-887) and then shifts the capped value
///      (`_orderExecutionPrice` / `_adverseOrderPrice`, :905-941), in the single resolver (:557-558) and the batch
///      resolver (:283-284). With the raw basket above CAP_PRICE and raw - shift still above CAP, a SHORT close fills at
///      CAP - shift instead of min(raw - shift, CAP) = CAP.
///      Public entry points only after the fixture's setUp: V3 `OrderRouter.commitOrder(OrderRequest)` from the
///      trader's own account, `router.executeOrder` / `router.executeOrderBatch` from KEEPER (no role), the
///      permissionless `router.updateMarkPrice`, `HousePool.settleLpEpoch`, `TrancheVault` request/claim, and the
///      oracle's unguarded `updateOrderExecutionPrice` from an unrelated EOA, used only as a priced read inside a state
///      snapshot that is reverted. Pyth is the maintainers' MockPyth. No protocol contract is pranked and no storage
///      is written. `_initialJuniorDeposit()` is 0 because the fixture's helper impersonates the router; the junior LP
///      deposits through the public vault flow instead, so the tests inherited from PositionProtectionTest are not
///      meant to run here.
contract Card9850S1CapThenShiftTest is PositionProtectionTest {
    address internal constant LP = address(0x1B1B);
    address internal constant KEEPER = address(0x5EE5); // any EOA, no role
    address internal constant REFRESHER = address(0x4EF4); // any EOA
    address internal constant READER = address(0x8EAD); // any EOA
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;

    uint256 internal constant ABOVE_CAP_RAW = 250_000_000; // basket 2.50, CAP_PRICE is 2.00
    // Cecuro MED-10's own example: raw basket 200_020_000, aggregate confidence 200_000
    uint256 internal constant MED10_RAW = 200_020_000;
    uint256 internal constant MED10_CONF = 200_000;

    uint256 private _clientNonce;
    uint256 private _sRaw;
    uint256 private _sConf;
    bool private _sBatch;
    uint64 private _sCloseId;

    struct RoundTrip {
        uint256 aliceBalanceAfter; // ALICE's settlement balance after the close (USDC, 6 dp)
        uint256 closePrice; // the close's execution price for the same tick, as the oracle's resolver reports it
    }

    /// No junior deposit in the fixture setUp: its helper pranks the router into `engine.updateMarkPrice`.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // -------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // -------------------------------------------------------------------------------------------------------------

    /// One signed update for every basket feed: price, per-feed confidence, publish time, previous publish time.
    function _tick(
        uint256 price,
        uint256 conf,
        uint256 publishTime,
        uint256 prevPublishTime
    ) private pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), uint64(conf), -8, publishTime, prevPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _nextBlock(
        uint256 dt
    ) private {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Anyone installs a fresh accepted basket via the permissionless `router.updateMarkPrice`.
    function _publicRefresh(
        uint256 price,
        uint256 conf
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tick(price, conf, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    /// An LP funds junior through the vault's public request/claim flow. The epoch is settled permissionlessly.
    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicRefresh(MARK_PRICE, 0);
        vm.prank(lp);
        pool.settleLpEpoch(0, 0);

        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    /// A production V3 order request (slippage bound never binding).
    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card-9850-s1", trader, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
            })
        });
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    /// KEEPER (no role) executes with the unique first tick after commitTime (historical execution) at `price`/`conf`.
    /// `batch` routes the execution through `executeOrderBatch` (the batch resolver) instead of `executeOrder`.
    function _keeperExecute(
        uint64 orderId,
        uint256 price,
        uint256 conf,
        bool batch
    ) private {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        if (vm.getBlockTimestamp() < uint256(committedAt) + 2) {
            vm.warp(uint256(committedAt) + 2);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tick(price, conf, committedAt + 1, committedAt);
        vm.prank(KEEPER);
        if (batch) {
            OrderV3Types.BatchResult memory res = router.executeOrderBatch(orderId, data);
            assertEq(res.terminalCount, 1, "ghost: batch executed the order");
        } else {
            OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
            assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
        }
    }

    function _bootstrapPool() private {
        assertTrue(pool.isTradingActive(), "ghost: trading activated");
        _publicJuniorDeposit(LP, JUNIOR_LP_ASSETS);
    }

    /// ALICE (funded by the fixture through clearinghouse.deposit) opens a SHORT at 1.00 through the router; the basket
    /// moves above CAP (raw basket `raw`, per-feed confidence `perFeedConf`); ALICE closes through the router. Returns
    /// her settlement balance after the close and the close's execution price as the resolver reports it for the same
    /// tick.
    function _shortRoundTripAboveCap(
        uint256 raw,
        uint256 perFeedConf,
        bool batch
    ) private returns (RoundTrip memory r) {
        // scenario held in storage: keeps the via-IR stack of the inlined helpers below the limit
        _sRaw = raw;
        _sConf = perFeedConf;
        _sBatch = batch;
        _openShortAtMark();
        _sCloseId = _moveAboveCapAndCommitClose(_sRaw, _sConf);
        r.closePrice = _resolverClosePrice(_sCloseId, _sRaw, _sConf);
        _executeScenarioClose();
        r.aliceBalanceAfter = _closedBalance();
    }

    function _executeScenarioClose() private {
        _keeperExecute(_sCloseId, _sRaw, _sConf, _sBatch);
    }

    /// The basket moves above CAP (a public refresh installs it, and the engine stores CAP); ALICE commits a full close.
    function _moveAboveCapAndCommitClose(
        uint256 raw,
        uint256 perFeedConf
    ) private returns (uint64 closeId) {
        _nextBlock(10);
        _publicRefresh(raw, perFeedConf);
        assertEq(engine.lastMarkPrice(), engine.CAP_PRICE(), "ghost: engine mark clamped to CAP");
        _nextBlock(1);
        closeId = _commitV3(ALICE, CfdTypes.Side.SHORT, POSITION_SIZE, 0, true);
    }

    function _closedBalance() private view returns (uint256) {
        assertEq(_aliceSize(), 0, "ghost: SHORT closed");
        return clearinghouse.balanceAssets(ALICE);
    }

    /// ALICE's position size: the first word of the public `engine.positions(address)` getter (read through a
    /// staticcall so via-IR does not hold the getter's seven return values on the stack).
    function _aliceSize() private view returns (uint256 size) {
        (bool ok, bytes memory ret) =
            address(engine).staticcall(abi.encodeWithSignature("positions(address)", ALICE));
        assertTrue(ok, "ghost: positions getter");
        size = abi.decode(ret, (uint256));
    }

    /// ALICE opens a SHORT at 1.00 through commit -> keeper execute, after a public refresh.
    function _openShortAtMark() private {
        _bootstrapPool();
        _nextBlock(10);
        _publicRefresh(MARK_PRICE, 0);
        _nextBlock(1);
        uint64 openId = _commitV3(ALICE, CfdTypes.Side.SHORT, POSITION_SIZE, POSITION_MARGIN_ASSETS, false);
        _keeperExecute(openId, MARK_PRICE, 0, false);
        assertEq(_aliceSize(), POSITION_SIZE, "ghost: SHORT open");
    }

    /// The close's first tick after commit, priced through the oracle's public resolver by an unrelated EOA. The call
    /// only writes caller-paid Pyth storage, and the state snapshot is reverted before the keeper executes.
    function _resolverClosePrice(
        uint64 closeId,
        uint256 raw,
        uint256 perFeedConf
    ) private returns (uint256 price) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(closeId);
        uint64 committedAt = pending.commitTime;
        vm.warp(uint256(committedAt) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tick(raw, perFeedConf, committedAt + 1, committedAt);
        IPletherOracle.OrderExecutionRequest memory req = IPletherOracle.OrderExecutionRequest({
            commitTime: committedAt,
            targetPrice: 1,
            side: CfdTypes.Side.SHORT,
            isClose: true,
            revertOnHistoricalUnavailable: true
        });
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(READER, fee);
        uint256 snap = vm.snapshotState();
        vm.prank(READER);
        (bool ok, IPletherOracle.PriceSnapshot memory s) =
            pletherOracle.updateOrderExecutionPrice{value: fee}(READER, data, req);
        assertTrue(ok, "ghost: resolver produced a price");
        price = s.price;
        vm.revertToState(snap);
    }

    /// The same round trip twice from the same state: once with zero confidence, once with `perFeedConf`.
    function _pair(
        uint256 raw,
        uint256 perFeedConf,
        bool batch
    ) private returns (RoundTrip memory noConf, RoundTrip memory withConf, uint256 shift) {
        uint256 snap = vm.snapshotState();
        noConf = _shortRoundTripAboveCap(raw, 0, batch);
        vm.revertToState(snap);
        withConf = _shortRoundTripAboveCap(raw, perFeedConf, batch);
        shift = perFeedConf * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;

        uint256 cap = engine.CAP_PRICE();
        uint256 med10Ordering = raw - shift < cap ? raw - shift : cap; // MED-10: shift the raw basket, then clamp
        console2.log("CAP / raw basket / per-feed conf", cap, raw, perFeedConf);
        console2.log("adverseConfidenceMultiplierBps / shift (8dp)", pletherOracle.adverseConfidenceMultiplierBps(), shift);
        console2.log("SHORT close price conf=0 / conf>0", noConf.closePrice, withConf.closePrice);
        console2.log("MED-10 ordering min(raw - shift, CAP)", med10Ordering);
        console2.log("ALICE balance conf=0 / conf>0", noConf.aliceBalanceAfter, withConf.aliceBalanceAfter);
        console2.log(
            "ALICE worse off vs conf=0 (USDC 6dp)",
            noConf.aliceBalanceAfter > withConf.aliceBalanceAfter ? noConf.aliceBalanceAfter - withConf.aliceBalanceAfter : 0
        );
    }

    // -------------------------------------------------------------------------------------------------------------
    // Control
    // -------------------------------------------------------------------------------------------------------------

    /// CONTROL: with zero confidence there is no shift, so the SHORT close above CAP fills at CAP. This is the price the
    /// MED-10 ordering also gives whenever raw - shift stays above CAP.
    function test_Control_ZeroConfidenceShortCloseAboveCapSettlesAtCap() public {
        RoundTrip memory r = _shortRoundTripAboveCap(ABOVE_CAP_RAW, 0, false);
        console2.log("SHORT close price conf=0 / CAP", r.closePrice, engine.CAP_PRICE());
        console2.log("ALICE balance after close (conf=0)", r.aliceBalanceAfter);
        assertEq(r.closePrice, engine.CAP_PRICE(), "zero-confidence SHORT close above CAP is priced at CAP");
    }

    // -------------------------------------------------------------------------------------------------------------
    // Property: the order path should use the MED-10 ordering (shift the raw basket, then clamp)
    // -------------------------------------------------------------------------------------------------------------

    /// PROPERTY (MED-10's fix: "apply the directional confidence adjustment to the raw, uncapped basket, then clamp ...
    /// This removes the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already
    /// capped mark"): with the raw basket far above CAP (raw - shift > CAP), a SHORT close settles at CAP, exactly as
    /// with zero confidence. FAILS on the pinned code: the order path caps first, then shifts (PletherOracle.sol:557-558,
    /// :940), so the close is priced at CAP - shift.
    function test_Property_ShortCloseAboveCapSettlesAtCap() public {
        // per-feed confidence at the oracle's accepted ceiling: 10 bps of price (basketMaxConfidenceRatioBps, :84)
        (RoundTrip memory noConf, RoundTrip memory withConf, uint256 shift) =
            _pair(ABOVE_CAP_RAW, ABOVE_CAP_RAW * 10 / 10_000, false);
        assertEq(noConf.closePrice, engine.CAP_PRICE(), "ghost: zero-confidence close at CAP");
        assertGt(ABOVE_CAP_RAW - shift, engine.CAP_PRICE(), "ghost: raw - shift is still above CAP");
        assertEq(withConf.closePrice, engine.CAP_PRICE(), "SHORT close priced below CAP although raw - shift > CAP");
        assertEq(withConf.aliceBalanceAfter, noConf.aliceBalanceAfter, "SHORT close settles worse than the MED-10 ordering");
    }

    // -------------------------------------------------------------------------------------------------------------
    // Characterization: the exact price and cost, the batch path, and MED-10's own numbers
    // -------------------------------------------------------------------------------------------------------------

    /// CHARACTERIZE: the close is priced at exactly CAP - shift, and ALICE ends worse off than with zero confidence by
    /// at most shift x size (the fill also moves notional-based charges slightly).
    function test_Characterize_ShortCloseAboveCapPricedAtCapMinusShift() public {
        (RoundTrip memory noConf, RoundTrip memory withConf, uint256 shift) =
            _pair(ABOVE_CAP_RAW, ABOVE_CAP_RAW * 10 / 10_000, false);
        uint256 cap = engine.CAP_PRICE();
        assertEq(noConf.closePrice, cap, "conf=0: CAP");
        assertEq(withConf.closePrice, cap - shift, "conf>0: CAP - shift (cap first, then shift)");
        uint256 loss = noConf.aliceBalanceAfter - withConf.aliceBalanceAfter;
        uint256 shiftTimesSize = shift * POSITION_SIZE / 1e20; // 8-dp price x 18-dp size -> 6-dp USDC
        console2.log("shift x size (USDC 6dp)", shiftTimesSize);
        assertGt(loss, 0, "ALICE is worse off");
        assertLe(loss, shiftTimesSize, "bounded by shift x size");
    }

    /// CHARACTERIZE: the batch resolver (`updateBatchOrderExecutionPrice`, :283-284) has the same ordering. The close
    /// executed through `executeOrderBatch` settles exactly as through `executeOrder`, below the zero-confidence result.
    function test_Characterize_BatchExecutionSameOrdering() public {
        uint256 perFeedConf = ABOVE_CAP_RAW * 10 / 10_000;
        uint256 snap = vm.snapshotState();
        RoundTrip memory single = _shortRoundTripAboveCap(ABOVE_CAP_RAW, perFeedConf, false);
        vm.revertToState(snap);
        (RoundTrip memory noConfBatch, RoundTrip memory withConfBatch,) = _pair(ABOVE_CAP_RAW, perFeedConf, true);
        console2.log("ALICE balance single / batch (conf>0)", single.aliceBalanceAfter, withConfBatch.aliceBalanceAfter);
        assertEq(withConfBatch.aliceBalanceAfter, single.aliceBalanceAfter, "batch fill == single fill");
        assertLt(withConfBatch.aliceBalanceAfter, noConfBatch.aliceBalanceAfter, "batch close also pays the shift");
    }

    /// CHARACTERIZE: Cecuro MED-10's own example input (raw 200_020_000, aggregate confidence 200_000), one basis point
    /// above CAP. The order path prices the SHORT close at CAP - shift, below the MED-10 ordering min(raw - shift, CAP).
    function test_Characterize_Med10ExampleInputOnOrderPath() public {
        (RoundTrip memory noConf, RoundTrip memory withConf, uint256 shift) = _pair(MED10_RAW, MED10_CONF, false);
        uint256 cap = engine.CAP_PRICE();
        uint256 med10Ordering = MED10_RAW - shift < cap ? MED10_RAW - shift : cap;
        assertEq(noConf.closePrice, cap, "conf=0: CAP");
        assertEq(withConf.closePrice, cap - shift, "order path: CAP - shift");
        assertGt(med10Ordering, withConf.closePrice, "below the MED-10 ordering");
        assertLt(withConf.aliceBalanceAfter, noConf.aliceBalanceAfter, "ALICE is worse off");
    }
}
```

