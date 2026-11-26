# \#99157 [SC-Low] A historical order fill blocks position-protection creation, replacement and attached staging until a new signed oracle refresh, contradicting the Book's own accepted-basket guard

**Submitted on Nov 12th 2026 at 07:07:13 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99157
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Scope commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Affected contracts: `packages/perps/src/PositionProtectionBook.sol` (primary), `packages/perps/src/PletherOracle.sol`.

## Description

Position protection (take-profit / stop-loss) is the protocol's risk-management feature for traders. Creating,
replacing, or attaching a protection to an open request goes through `PositionProtectionBook._freshProtectionObservation`
(`packages/perps/src/PositionProtectionBook.sol:862-886`).

The ordinary way a position is opened is a delayed order filled by a keeper from a signed **historical** Pyth basket
(`PletherOracle._resolveHistoricalOrderBasket`, `PletherOracle.sol:596-636`). That fill installs the historical tick as
the Engine mark, but it does **not** write the Oracle's accepted live basket, which is only written on the live path
(`PletherOracle.sol:786-799`). The documentation states this explicitly: "historical execution may advance the Engine
without advancing the accepted live basket" (`ORACLE_SYNCHRONIZATION.md:150-151`).

After such a fill the accepted basket is older than the Engine mark. `_freshProtectionObservation` then calls
`oracle.getLatestPrice(PriceMode.MarkRefresh)` (`PositionProtectionBook.sol:880`). That view enforces publish order
against the Engine mark and reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, engineMarkTime)`
(`PletherOracle.sol:892-896`, reached through `_getLatestPriceSnapshotWithConfidence`, `:571-580`).

The very next statement in the Book, `_acceptedMarkInstalled()` (`PositionProtectionBook.sol:888-895`), was written to
**accept** exactly this state:

```solidity
return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

It is never reached, because the preceding oracle view has already reverted. The tolerance is dead code on every
entrypoint that uses `_freshProtectionObservation`: `createPositionProtection` (`:207-254`, via `_create`),
`replacePositionProtection` (`:257-312`) and `commitOpenOrderWithProtection` staging (`:432-493`, `:463`).

The audit disposition of MED-28 describes the intended guard as rejecting arming "when the accepted oracle mark is
ahead of, or inconsistent with, the Engine mark". An accepted mark that is merely **behind** a newer Engine mark is
neither; the Book's code treats it as installed, yet the transaction reverts.

### Consequences

1. A trader who opens a position through the normal delayed-order path cannot protect it in the next transaction: the
   fill itself put the system in the blocking state.
2. The zero-fee empty replay (`router.updateMarkPrice(new bytes[](0))`, quoted at zero by `getUpdateFee`) does not
   repair it; it replays the older accepted basket and reverts with the same error.
3. Only a new signed live payload newer than the fill restores arming. It costs a Pyth update and has to be a separate
   transaction for an EOA, because the Book has no refresh-and-arm entrypoint.
4. Any fill of **any** account's order between that refresh and the arm call recreates the state (shown in the PoC
   with a third party's fill blocking both `replacePositionProtection` and another user's `createPositionProtection`).
   On an active market, fills are frequent, so an EOA can repeatedly fail to place or tighten a stop-loss exactly when
   it matters.

No funds are lost by this defect itself. The loss is the promised ability to arm or adjust a protection on a position
opened through the protocol's normal path.

## Root cause

`_freshProtectionObservation` takes its observation from `getLatestPrice(MarkRefresh)`, which enforces
`accepted.publishTime >= engine.lastMarkTime()`. Historical fills legitimately break that relation. The Book's own
synchronization predicate (`_acceptedMarkInstalled`) is the intended guard and accepts the relation, but it runs after
the reverting call.

## Recommendation

In `_freshProtectionObservation`, when the accepted basket is older than the Engine mark, use the Engine mark (the
newest validated price, installed by the fill) as the observation instead of calling the order-enforcing view. That
matches the Book's own `_acceptedMarkInstalled` predicate and preserves MED-28's protection (an accepted basket AHEAD of
the Engine mark is still rejected). Alternatively, add a payable refresh-and-arm entrypoint so EOAs can install a new
signed mark and arm in one transaction.

## Out-of-scope check

Not in `SECURITY.md` Known Limitations; not among the 33 titles of `audits/scan-plether-fi-plether-core-d4541b49.md`
(its item at line ~927 concerns orders blocked by a permissionlessly advanced mark, a different relation);
`ORACLE_SYNCHRONIZATION.md:150-151` documents the mark divergence as a monitoring fact, not as a protection limitation.

## Proof of Concept

## Proof of concept

Executable Foundry test against the scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, written and run on
2026-11-12 on the machine that wrote it; the run log is at the end. It uses the repository's own test stack
(`BasePerpTest`: real Router, real PletherOracle over the repository's MockPyth, real PositionProtectionBook, Engine,
Clearinghouse and HousePool), so it must be saved inside the repository's test tree.

Save the file as `packages/perps/test/perps/accounting/close/ZzChain05ProtectionArming.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzChain05ProtectionArming.t.sol' -vv
```

Expected: 3 tests pass.

### Step by step

1. Setup: release risk parameters (maintenance 10 bps, initial 20 bps, max skew 40%, bounty 10 bps / 1 USDG; price
   impact and carry off because they are irrelevant), a live (not FAD, not frozen) session, and one signed live refresh
   so that the Oracle's accepted basket and the Engine mark agree (both at time T).
2. Alice commits a LONG of 1,000 lots through `router.commitOrder`. A keeper executes it with `router.executeOrder`
   from the signed historical tick `commitTime + 1` (the normal delayed-order path). The fill installs that tick as the
   Engine mark; the accepted basket is not touched. Logged: accepted basket time 1709632801, Engine mark time 1709632802.
3. Alice calls `createPositionProtection` with a take-profit at 0.90 and a stop-loss at 1.10. It reverts with
   `PletherOracle__PriceOutOfOrder(1709632801, 1709632802)`. The test also calls the same view the Book calls first,
   `getLatestPrice(PriceMode.MarkRefresh)`, and gets the identical revert.
4. Alice tries the zero-fee repair, `router.updateMarkPrice(new bytes[](0))` (an empty payload is quoted at zero fee and
   replays the accepted basket). It reverts with the same error, and creation still reverts.
5. A new signed live refresh newer than the fill (time 1709632805) is submitted. Creation now succeeds and the
   protection is `Armed`.
6. Second test: Alice and Bob hold positions, Alice arms a protection right after a refresh. Alice refreshes again,
   intending to tighten her stop; Bob's ordinary order is filled first (Engine mark 1709632808, accepted 1709632807).
   Alice's `replacePositionProtection` reverts with `PletherOracle__PriceOutOfOrder`, and a third user, Carol, who opened
   through the same path, cannot `createPositionProtection` either.
7. Control: when the position is opened directly through the Engine (as the vendor test suite does) and no historical
   fill advances the Engine mark, accepted basket and Engine mark agree and creation succeeds.

What the test simplifies: Pyth data is the repository's MockPyth (it validates plumbing, not signatures), and the
keeper is a test address. Everything else is called through its real entrypoints and real callers.

### PoC file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// CHAIN-05 PoC: a historical order fill blocks position-protection arming until a new signed live refresh.
//
// Mechanism (source references at a82d4fef):
//   - A live/FAD delayed order is filled from a signed HISTORICAL basket (`_resolveHistoricalOrderBasket`,
//     PletherOracle.sol:596-636). The fill installs that tick as the Engine mark, but the Oracle's accepted live
//     basket is written only on the live path (PletherOracle.sol:786-799). ORACLE_SYNCHRONIZATION.md:150-151: "historical
//     execution may advance the Engine without advancing the accepted live basket".
//   - Protection creation, replacement and attached-open staging all call `_freshProtectionObservation`
//     (PositionProtectionBook.sol:862-886), whose first oracle read is `getLatestPrice(MarkRefresh)` (:880). That view
//     enforces publish order against the Engine mark (PletherOracle.sol:892-896) and reverts with
//     `PletherOracle__PriceOutOfOrder` while the accepted basket is older than the Engine mark.
//   - Two lines later the Book's own `_acceptedMarkInstalled()` (:888-895) explicitly ACCEPTS that state
//     (`acceptedTime < markTime`). The tolerance is unreachable on create / replace / attach.
//   - The zero-fee empty replay cannot repair it (it replays the older accepted basket and hits the same order check);
//     only a new signed live payload newer than the fill does. Any fill landing between that refresh and the arm call
//     (another user's order, executed by any keeper) re-creates the state.
//
// Release risk parameters except price impact and carry (off; irrelevant to the mechanism).
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzChain05ProtectionArming.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

contract ZzChain05ProtectionArmingTest is BasePerpTest {

    uint256 internal constant LOT_ = 1e20;
    address internal constant KEEPER0 = address(0xCAFE);
    address internal constant ALICE_ = address(0xC05A);
    address internal constant BOB_ = address(0xC05B);

    PositionProtectionBook internal book;
    uint256 internal markNow = 1e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 0;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function setUp() public override {
        super.setUp();
        book = PositionProtectionBook(address(router.positionProtectionBook()));
        vm.deal(KEEPER0, 100 ether);
        _fundTrader(ALICE_, 50_000e6);
        _fundTrader(BOB_, 50_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 days + 3 hours);
        _liveRefresh();
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "setup must start in a live session");
    }

    // ------------------------------------------------------------------ helpers

    function _pythData(
        uint256 publishTime
    ) internal returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = new bytes32[](2);
        feedIds[0] = bytes32(uint256(1));
        feedIds[1] = bytes32(uint256(2));
        baseMockPyth.setAllUniquePrices(feedIds, int64(int256(markNow)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(markNow);
    }

    /// @dev New block, new second, signed live payload at `now`: installs BOTH the accepted basket and the Engine mark.
    function _liveRefresh() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _pythData(vm.getBlockTimestamp());
        vm.prank(KEEPER0);
        router.updateMarkPrice(data);
    }

    /// @dev Executes the queue head from a signed historical tick one second after its commit (the normal keeper path).
    function _fillHead() internal {
        uint64 head = router.nextExecuteId();
        require(head != 0, "no head");
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _pythData(uint256(pending.commitTime) + 1);
        vm.prank(KEEPER0);
        router.executeOrder(head, data);
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _marks(
        string memory label
    ) internal returns (uint64 acceptedTime, uint64 engineTime) {
        emit log_string(label);
        acceptedTime = _logAccepted();
        engineTime = engine.lastMarkTime();
        emit log_named_uint("  engine mark time       ", engineTime);
        emit log_named_uint("  engine mark price      ", engine.lastMarkPrice());
    }

    function _logAccepted() internal returns (uint64 at) {
        (, uint256 acceptedPrice, uint64 acceptedAt) = pletherOracle.getAcceptedBasketMark();
        at = acceptedAt;
        emit log_named_uint("  accepted basket time   ", acceptedAt);
        emit log_named_uint("  accepted basket price  ", acceptedPrice);
    }

    function _create(
        address account,
        PositionProtectionTypes.PositionProtectionParams memory params
    ) internal returns (bool ok, bytes memory reason, uint64 id) {
        vm.prank(account);
        try book.createPositionProtection(params) returns (uint64 protectionId) {
            ok = true;
            id = protectionId;
        } catch (bytes memory r) {
            reason = r;
        }
    }

    function _selector(
        bytes memory reason
    ) internal pure returns (bytes4 sel) {
        if (reason.length >= 4) {
            assembly ("memory-safe") {
                sel := mload(add(reason, 32))
            }
        }
    }

    // ------------------------------------------------------------------ PoC

    /// 1. A position opened through the ordinary delayed-order path cannot be protected right after its fill.
    /// 2. The zero-fee replay does not repair it. 3. A new signed refresh does.
    function test_Chain05_FillBlocksArmingUntilSignedRefresh() public {
        _marks("after setUp live refresh (accepted == engine)");

        vm.prank(ALICE_);
        router.commitOrder(CfdTypes.Side.LONG, 1000 * LOT_, 5_000e6, 0, false);
        _fillHead();
        (uint256 size,,,,,,) = engine.positions(ALICE_);
        assertEq(size, 1000 * LOT_, "Alice's open must fill");
        (uint64 acceptedTime, uint64 engineTime) = _marks("after Alice's historical fill");
        assertLt(acceptedTime, engineTime, "the fill must advance the Engine mark past the accepted basket");

        // The Book's own state predicate accepts this state: acceptedTime < engineTime. Yet creation reverts first.
        (bool ok, bytes memory reason,) = _create(ALICE_, _params(0.9e8, 1.1e8));
        emit log_named_bytes("create revert data", reason);
        assertFalse(ok, "CHAIN-05: protection creation must revert after the fill");
        assertEq(
            _selector(reason), IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "reverts on the oracle order check"
        );
        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__PriceOutOfOrder.selector, uint256(acceptedTime), uint256(engineTime)
        );
        assertEq(keccak256(reason), keccak256(expected), "revert carries (accepted time, engine mark time)");

        // The same view the Book calls first.
        vm.expectRevert(expected);
        pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);

        // 2. Zero-fee empty replay (getUpdateFee == 0) cannot repair it: it replays the older accepted basket.
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(ALICE_);
        try router.updateMarkPrice(new bytes[](0)) {
            emit log_string("empty replay succeeded");
        } catch (bytes memory r) {
            emit log_named_bytes("empty replay revert", r);
        }
        (ok, reason,) = _create(ALICE_, _params(0.9e8, 1.1e8));
        assertFalse(ok, "CHAIN-05: still blocked after the zero-fee replay");

        // 3. A new signed live payload newer than the fill repairs it (paid Pyth update, separate transaction).
        _liveRefresh();
        _marks("after a new signed live refresh");
        uint64 id;
        (ok, reason, id) = _create(ALICE_, _params(0.9e8, 1.1e8));
        assertTrue(ok, "a signed refresh newer than the fill restores arming");
        assertEq(
            uint8(book.getPositionProtection(id).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "protection armed"
        );
    }

    /// 4. Any other account's fill between a trader's refresh and its arm call re-creates the block, for creation and
    ///    for replacement of an already armed protection.
    function test_Chain05_ThirdPartyFillReBlocksCreateAndReplace() public {
        // Alice and Bob both hold positions; Alice arms a protection after a clean refresh.
        vm.prank(ALICE_);
        router.commitOrder(CfdTypes.Side.LONG, 1000 * LOT_, 5_000e6, 0, false);
        _fillHead();
        vm.prank(BOB_);
        router.commitOrder(CfdTypes.Side.SHORT, 1000 * LOT_, 5_000e6, 0, false);
        _fillHead();
        _liveRefresh();
        (bool ok,, uint64 id) = _create(ALICE_, _params(0.9e8, 1.1e8));
        assertTrue(ok, "arming works right after a refresh with no fill in between");

        // Alice refreshes the mark herself, intending to tighten her stop next. Bob's ordinary order fills first.
        _liveRefresh();
        vm.prank(BOB_);
        router.commitOrder(CfdTypes.Side.SHORT, 100 * LOT_, 500e6, 0, false);
        _fillHead();
        (uint64 acceptedTime, uint64 engineTime) = _marks("after Bob's fill");
        assertLt(acceptedTime, engineTime, "Bob's fill advanced the Engine mark past the accepted basket");

        vm.prank(ALICE_);
        try book.replacePositionProtection(id, _params(0.9e8, 1.05e8)) {
            fail();
        } catch (bytes memory r) {
            emit log_named_bytes("replace revert", r);
            assertEq(_selector(r), IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "replace blocked by Bob's fill");
        }

        // Carol (fresh account with a position opened by the same path) cannot create either.
        address carol = address(0xC05C);
        _fundTrader(carol, 50_000e6);
        vm.prank(carol);
        router.commitOrder(CfdTypes.Side.LONG, 100 * LOT_, 500e6, 0, false);
        _fillHead();
        bytes memory reason;
        (ok, reason,) = _create(carol, _params(0.9e8, 1.1e8));
        assertFalse(ok, "create blocked after the fill");
        assertEq(_selector(reason), IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "same oracle order check");
    }

    /// 5. Control: the state the vendor tests exercise (accepted basket AHEAD of the Engine mark) is the one the Book
    ///    names `OracleMarkAhead`; the state above (accepted basket BEHIND) is neither covered nor documented as a
    ///    protection limitation. Positions opened directly through the Engine (as in the vendor suite) never reach it.
    function test_Chain05_ControlNoFillNoBlock() public {
        _open(ALICE_, CfdTypes.Side.LONG, 1000 * LOT_, 5_000e6, markNow);
        _liveRefresh();
        (uint64 acceptedTime, uint64 engineTime) = _marks("direct open + live refresh");
        assertEq(acceptedTime, engineTime, "no historical fill: accepted basket and Engine mark agree");
        (bool ok,,) = _create(ALICE_, _params(0.9e8, 1.1e8));
        assertTrue(ok, "control: arming works when no fill advanced the Engine mark");
    }

}
```

### Run log (2026-11-12, scope commit)

```
cmdline: forge test --root packages/perps --match-path test/perps/accounting/close/ZzChain05ProtectionArming.t.sol -vv  (CHAIN-05 PoC, run 2: stack-too-deep fixed)
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 106.84s
Compiler run successful with warnings:
Ran 3 tests for test/perps/accounting/close/ZzChain05ProtectionArming.t.sol:ZzChain05ProtectionArmingTest
[PASS] test_Chain05_ControlNoFillNoBlock() (gas: 1726430)
Logs:
  direct open + live refresh
    accepted basket time   : 1709632802
    accepted basket price  : 100000000
    engine mark time       : 1709632802
    engine mark price      : 100000000
[PASS] test_Chain05_FillBlocksArmingUntilSignedRefresh() (gas: 3502160)
Logs:
  after setUp live refresh (accepted == engine)
    accepted basket time   : 1709632801
    accepted basket price  : 100000000
    engine mark time       : 1709632801
    engine mark price      : 100000000
  after Alice's historical fill
    accepted basket time   : 1709632801
    accepted basket price  : 100000000
    engine mark time       : 1709632802
    engine mark price      : 100000000
  create revert data: 0x8e3e110d0000000000000000000000000000000000000000000000000000000065e6ed210000000000000000000000000000000000000000000000000000000065e6ed22
  empty replay revert: 0x8e3e110d0000000000000000000000000000000000000000000000000000000065e6ed210000000000000000000000000000000000000000000000000000000065e6ed22
  after a new signed live refresh
    accepted basket time   : 1709632805
    accepted basket price  : 100000000
    engine mark time       : 1709632805
    engine mark price      : 100000000
[PASS] test_Chain05_ThirdPartyFillReBlocksCreateAndReplace() (gas: 8722268)
Logs:
  after Bob's fill
    accepted basket time   : 1709632807
    accepted basket price  : 100000000
    engine mark time       : 1709632808
    engine mark price      : 100000000
  replace revert: 0x8e3e110d0000000000000000000000000000000000000000000000000000000065e6ed270000000000000000000000000000000000000000000000000000000065e6ed28
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 38.28ms (32.86ms CPU time)
Ran 1 test suite in 40.58ms (38.28ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
exit_code=0
```
