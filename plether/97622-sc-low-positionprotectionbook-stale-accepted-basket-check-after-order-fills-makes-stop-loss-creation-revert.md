# \#97622 [SC-Low] PositionProtectionBook: stale accepted-basket check after order fills makes stop-loss creation revert

**Submitted on Nov 2nd 2026 at 14:50:26 UTC by @jack0x0000001 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97622
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every delayed-order fill moves the Engine mark forward without moving the Oracle's accepted basket. `PositionProtectionBook` then reads the accepted basket through `getLatestPrice(MarkRefresh)`, which reverts when the basket is older than the Engine mark. As a result, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` all revert after any fill until someone pays for a fresh `updateMarkPrice`, and the next fill breaks them again. Traders lose reliable access to take-profit and stop-loss exactly when they want it most: right after their own order fills.

## Vulnerability Details

Delayed orders use the historical path. `PletherOracle._resolveHistoricalOrderBasket` parses the post-commit basket but does not write `_acceptedBasket`; only `_resolveCurrentBasket` writes it (`PletherOracle.sol#L795-L800`). The execution sidecar still advances the Engine mark:

```solidity
// OrderRouterExecutionSidecar.sol#L1058-L1060
if (result.publishTime > engine_.lastMarkTime()) {
    engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
}
```

So after a fill, `acceptedBasket.publishTime < engine.lastMarkTime()`. This is the normal state, and the Book's own guard treats it as valid:

```solidity
// PositionProtectionBook.sol#L894
return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

That guard is never reached in this state. Just before that check, `_freshProtectionObservation` calls:

```solidity
// PositionProtectionBook.sol#L880
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
```

`getLatestPrice` reaches `_snapshotFromBasket(basket, policy, true)`, which rejects a basket older than the mark:

```solidity
// PletherOracle.sol#L892-L896
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

`_freshProtectionObservation` is the common path for `createPositionProtection` (`#L229`), `replacePositionProtection` (`#L295`) and `commitOpenOrderWithProtection` (`#L463`).

The protocol documents this state as normal. `ORACLE_SYNCHRONIZATION.md` says "historical execution may advance the Engine without advancing the accepted live basket". `afterOrderTerminal` also arms an attached protection in this state, using `_acceptedMarkInstalled()` and the Engine mark, without the `getLatestPrice` call.

### Relation to Cecuro MED-28

This bug was introduced by the MED-28 fix. MED-28 (trader arms protection from a stale Engine mark after oracle state already crossed it) was addressed by validating new protection against the current accepted oracle observation. That change added `_freshProtectionObservation` and its `getLatestPrice(MarkRefresh)` call. The audit report links the fix to commit `8f6cfd3d84d1289aa1209e3b08a06e9bfdf38a0d`, and the code reached the main branch through PR #124 (`eadc5c587`). At the previous main-branch revision of the Book (`819192182`), neither existed.

The fix works for the case MED-28 describes: an accepted basket ahead of the Engine mark is rejected with `PositionProtectionBook__OracleMarkAhead`. The new oracle read also rejects the reverse case, where the Engine mark is ahead of the accepted basket — a state the Book's own guard accepts. This report does not re-raise MED-28. It shows that the fix at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` introduces a new bug.

## Impact Details

No funds move. The harm is a user-facing protection feature that fails in a normal protocol state:

- A trader with an open position cannot create a stop-loss until a live mark refresh lands after the most recent fill.
- A trader cannot replace an armed protection under the same condition.
- Calling `commitOpenOrderWithProtection` from a flat account reverts under the same condition.

An EOA has to send `updateMarkPrice` with a live signed payload in a separate transaction, and any fill landing between the two transactions triggers the revert again. A contract account can batch both calls. Whether this occurs in production depends on how often keepers refresh the mark relative to fills. The impact claimed is the closest entry on the Low list: the promised service, placing protection on an open position, is not delivered while the state lasts.

## Risk Breakdown

- Attacker: none required. Ordinary order flow from any account produces the state. Only the non-frozen market is affected; while the oracle is frozen, protection creation reverts earlier by design.
- Preconditions: an accepted basket exists, and at least one order has filled since the last live mark refresh.
- Complexity: none. It occurs in normal operation.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L903
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1054-L1061
- https://github.com/Plether-Fi/plether-core/commit/8f6cfd3d84d1289aa1209e3b08a06e9bfdf38a0d
- https://github.com/Plether-Fi/plether-core/pull/124

## Recommendation

When the Engine mark is newer than the accepted basket, use the Engine mark and `lastMarkTime` as the observation instead of calling `getLatestPrice`. This matches what `afterOrderTerminal` already does when arming after a fill, and it keeps the arming baseline at the Engine's publish time so a later trigger still requires a newer publication.

## Proof of Concept

## Proof of Concept

The test below runs against the repository's `BasePerpTest` fixture, which uses `MockPyth` and the `LegacyOrderRouterHarness` commit adapter into the production V3 entrypoint. Risk parameters are overridden to the v2.0.0 manifest values. No storage is written and no privileged component is pranked.

1. `router.updateMarkPrice` installs a live basket, so accepted basket time equals Engine mark time.
2. A trader commits an open, and a keeper executes it with the first post-commit tick. The Engine mark moves past the accepted basket: accepted `1709539200`, mark `1709539202` in this fixture.
3. The trader calls `createPositionProtection`. It reverts with `PletherOracle__PriceOutOfOrder(1709539200, 1709539202)`.
4. After a new `router.updateMarkPrice`, the same call succeeds and returns protection id 1.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "@perps-test/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";


/// Release-configuration base: risk parameters come from the v2.0.0 manifest economics.
/// Trades go through Router entrypoints (commitOrder + executeOrder), not BasePerpTest prank shortcuts.
abstract contract ReleaseConfigBase is BasePerpTest {

    address internal attacker = makeAddr("attacker");

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

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function setUp() public virtual override {
        super.setUp();
        uint256 seedAmount = 1e6; // manifest: juniorSeedAssets / seniorSeedAssets
        settlementToken.mint(address(this), seedAmount * 2);
        settlementToken.approve(address(pool), seedAmount * 2);
        pool.initializeSeedPosition(false, seedAmount, address(this));
        pool.initializeSeedPosition(true, seedAmount, address(this));
        pool.activateTrading();
        _fundJunior(address(0xB0B), 1_000_000e6);
    }

}


/// A routine order execution advances the Engine mark without advancing the Oracle accepted basket.
/// `_freshProtectionObservation` then calls `getLatestPrice(MarkRefresh)`, which reverts PriceOutOfOrder,
/// although `_acceptedMarkInstalled` right below it explicitly accepts acceptedTime < markTime.
contract ProtectionBlockedAfterExecution is ReleaseConfigBase {
    function _live(uint256 price) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory t = new MockPyth.FeedUpdate[](ids.length);
        uint256 prev = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            t[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), prev));
        }
        d = new bytes[](1); d[0] = abi.encode(t);
    }
    function _params() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.stopLossTriggerPrice = 1.01e8;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits(type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max);
    }
    function test_entry_PoC() public {
        IPositionProtectionActions book = IPositionProtectionActions(address(router.positionProtectionBook()));
        _fundTrader(attacker, 100_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        router.updateMarkPrice(_live(1e8));                 // keeper-style refresh: accepted basket == mark
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, 50_000e18, 1000e6, 0, false);
        router.executeOrder(1, _mockPythUpdateData());      // ordinary fill: mark moves past accepted basket
        (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        console2.log("acceptedTime / engine.lastMarkTime", acceptedTime, engine.lastMarkTime());
        assertLt(acceptedTime, engine.lastMarkTime());
        uint64 markTime = engine.lastMarkTime();
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime));
        vm.prank(attacker);
        book.createPositionProtection(_params());
        console2.log("createPositionProtection reverted PriceOutOfOrder");
        // Only a paid refresh in an earlier step repairs it; any fill in between re-breaks it.
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_live(1e8));
        vm.prank(attacker);
        uint64 id = book.createPositionProtection(_params());
        assertGt(id, 0);
        console2.log("after refresh protection id", id);
    }
}
```

## Build configuration

Foundry 1.5.1, Solidity 0.8.35, optimizer on with 200 runs, `via_ir = true`. These are the repository's own `foundry.toml` settings at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with no changes. For the fixture, one remapping is added on the command line and `FOUNDRY_TEST` points at a new folder, so only this file is built.

Run from the repository root with submodules initialized:

```bash
mkdir poc
# save the test above as poc/C01.t.sol
FOUNDRY_TEST=poc forge test --remappings "@perps-test/=packages/perps/test/perps/" --match-contract ProtectionBlockedAfterExecution -vv
```

## Output

```text
Ran 1 test for poc/C01.t.sol:ProtectionBlockedAfterExecution
[PASS] test_entry_PoC() (gas: 3735765)
Logs:
  acceptedTime / engine.lastMarkTime 1709539200 1709539202
  createPositionProtection reverted PriceOutOfOrder
  after refresh protection id 1

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.94ms (2.62ms CPU time)
```
