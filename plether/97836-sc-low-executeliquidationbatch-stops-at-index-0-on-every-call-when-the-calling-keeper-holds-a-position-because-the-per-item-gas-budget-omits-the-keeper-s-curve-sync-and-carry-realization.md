# \#97836 [SC-Low] executeLiquidationBatch stops at index 0 on every call when the calling keeper holds a position, because the per-item gas budget omits the keeper's curve sync and carry realization

**Submitted on Nov 3rd 2026 at 14:09:29 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97836
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`OrderRouter.executeLiquidationBatch` runs each account in a self-call capped at `minEngineGas + 250,000` gas plus 600,000 per pending order (850,000 for an account without orders at the deployed `minEngineGas` of 600,000). The budget only counts the liquidated account's work. When the caller (the keeper) holds an open position, `CfdEngine._applyLiquidation` also re-syncs the keeper's terminal NAV curve and realizes the keeper's accrued carry: 286,104 more gas in the PoC. With the release parameters and the deployed via-IR build, the first liquidatable account runs out of gas, the empty revert stops the batch at index 0, and the revert also undoes the keeper's carry checkpoint, so every retry stops at index 0 again. A keeper that trades from its keeper address cannot liquidate anything through the batch.

## Vulnerability Details

1. The per-item budget is fixed and depends only on the liquidated account (`router/OrderLiquidationBatchLogic.sol:113-116`, `:761`, `:765-766`):

```solidity
/// @notice Fixed item budget above `minEngineGas` for router dispatch and account-level bounty accounting.
uint256 private constant LIQUIDATION_BATCH_ROUTER_GAS = 250_000;
...
uint256 engineGas = host.minEngineGas();
...
uint256 itemGas = engineGas + LIQUIDATION_BATCH_ROUTER_GAS
    + (host.pendingOrderCounts(account) * LIQUIDATION_BATCH_GAS_PER_ORDER);
```

`minEngineGas` is 600,000 (`router/OrderRouterBase.sol:76`; the Arbitrum Sepolia v2.0.0 Router returns 600000). The cap does not grow with the transaction gas limit.

2. The item is called with that cap and with the batch caller as `keeper` (`:777-784`, `msg.sender` at `:783`). An empty revert, which is what an out-of-gas item returns, ends the batch at the current index (`:796-800`):

```solidity
} catch (bytes memory revertData) {
    if (revertData.length == 0) {
        emit LiquidationBatchStopped(nextIndex);
        return nextIndex;
    }
```

3. Inside the item the Engine adds work for the keeper when the keeper holds a position (`CfdEngine.sol:1331-1338`, `:1346-1348`):

```solidity
bool syncKeeper = delta.keeperBountyAssets > 0 && keeper != delta.account && _positions[keeper].lots > 0;
bytes32 keeperExpectedOldHash;
if (syncKeeper) {
    keeperExpectedOldHash = _beginTerminalCurveMutation(keeper);
}
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
    _checkpointBountyRecipient(keeper, delta.price, publishTime);
}
...
if (syncKeeper) {
    _endTerminalCurveMutation(keeper, keeperExpectedOldHash);
}
```

The bracket calls `TerminalNavBookV2.authenticateEngineState` and `syncFromEngine` for the keeper (`CfdEngine.sol:187`, `:198`). `_checkpointBountyRecipient` realizes the keeper's carry through `_realizeCarryFromSettlement` (`:213-216`, `:1407-1449`): clearinghouse debit, transfer to the HousePool, pool inflow record, and a curve change that `syncFromEngine` then writes.

4. Measured in the PoC (default profile, via-IR as deployed; DEPLOYMENT.md release risk parameters; three underwater longs; keeper holding a 10,000 USDG short for one day):
   - flat keeper: the first item uses 609,996 gas; all three accounts are liquidated;
   - single path, same liquidation: 1,042,089 gas for a flat caller, 1,328,193 for the keeper with a position. The keeper-side work is 286,104 gas, so the first item needs about 896,000 gas against the 850,000 cap;
   - keeper with a position: `OutOfGas` inside the keeper's `TerminalNavBookV2.syncFromEngine` (from `CfdEngine.sol:1347`), then `LiquidationBatchStopped(nextIndex: 0)`.

5. Retrying does not help. The revert rolls back the keeper's carry checkpoint too, so the next call meets the same unrealized carry at the same account. The documentation promises a safe resume. `SECURITY.md:1259-1261`: "the returned cursor leaves any low-gas or empty-revert item unattempted so a keeper can resume safely." `IPerpsKeeper.sol:83-85`: "Low gas or an empty item revert leaves the returned index unattempted. ... resume by submitting the suffix at `nextIndex` with a fresh Pyth update." For this caller the suffix stops at the same index on every call (PoC: three calls, three stops at 0).

6. The single path is not affected: `executeLiquidation` calls the same item without a cap (`router/OrderLiquidationBatchLogic.sol:727-729`).

Conditions: the caller holds a position with carry accrued since its last checkpoint (the normal state of a held position); the liquidated account has no pending orders (each order adds 600,000 to its budget); a keeper bounty is due (`minBountyAssets` makes it nonzero). The stop does not depend on the liquidated account: the PoC fuzz test covers long and short accounts from 10,000 to 100,000 USDG notional (256 runs, all stop at 0; a flat keeper liquidates each).

Why existing tests miss it: every batch test calls from an address without a position. `test_Batch_MaxPendingOrdersLiquidatesWithinItemGasBudget` (`test/perps/spec/keeper/LiquidationBatch.t.sol:242`) uses the flat fixture `KEEPER` and raises `minEngineGas` to 5,000,000 (`:246`). The production gas test (`test/perps/gas/SourceBoundBountyGas.t.sol:125`, `:311-345`) uses `BOUNTY_KEEPER`, also flat. The tests of a keeper with an open position (`TerminalNavIntegrationSecurity.t.sol:339`, `MarginFirstCarry.t.sol:306`) call `engine.liquidatePosition` directly as the Router, outside the item cap. `LiquidationBatchFixture.sol:87-88` notes that coverage bytecode "can exhaust the production item cap when it also collects carry"; only the liquidated account's carry is considered.

Not Cecuro LOW-52: that was the 63/64 forwarding shortfall (fixed in b3a7665b). Here the full 850,000 is forwarded and is too small on every call because it does not count the caller's work; LOW-52's residual concerns accounts padded with pending orders, and these accounts have none.

## Impact Details

Keepers are permissionless and the batch credits each bounty to the caller's own clearinghouse account (`IPerpsKeeper.sol:82-83`, "The original caller receives each successful engine-planned bounty"). The Engine explicitly supports keepers with open positions (the `syncKeeper` bracket and the two tests above). A trader who runs a keeper bot from the address that trades and collects its bounties is an expected user of this entry point.

For that keeper, `executeLiquidationBatch` liquidates zero accounts on every call, while each attempt pays the Pyth fee and about 1.1 to 1.3 million gas, and the documented resume never progresses. Liquidations stay available through `executeLiquidation` (one Pyth update per account) or from a flat address, so no principal is lost. The batch entry point does not deliver its documented function for these callers, and a keeper that relies on it delays the liquidation of underwater accounts until it switches path.

Selected impact: Low, "Contract fails to deliver promised returns, but doesn't lose value".

Recommended fix (any one):

- Add the caller-side cost to the budget when the caller holds a position (read once before the loop):

```solidity
bool callerHasPosition = _callerHasPosition(msg.sender); // engine.positions(msg.sender).size > 0
...
uint256 itemGas = engineGas + LIQUIDATION_BATCH_ROUTER_GAS
    + (host.pendingOrderCounts(account) * LIQUIDATION_BATCH_GAS_PER_ORDER)
    + (callerHasPosition ? LIQUIDATION_BATCH_BOUNTY_RECIPIENT_GAS : 0); // about 350_000
```

- Or checkpoint the caller's carry once in the outer batch frame before the loop, so items only pay the curve bracket.
- Or accumulate batch bounties and credit the caller once after the loop, outside the per-item frames.

Add a regression test that calls the batch from an address holding a position with accrued carry.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L113-L116
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L761-L800
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L727-L729
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1324-L1349
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L181-L217
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L76
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1259-L1261
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPerpsKeeper.sol#L80-L92

## Proof of Concept

## Proof of Concept

Built on the repository's `LiquidationBatchTestFixture` (real Router, batch sidecar, Engine, TerminalNavBookV2, Clearinghouse, HousePool; MockPyth only supplies prices) with the DEPLOYMENT.md release risk parameters. No in-scope file is modified.

1. Save the file below as `packages/perps/test/perps/poc/LiquidationBatchKeeperPositionGas.t.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. Run from the repository root with the default profile (via-IR, the deployed build): `forge test --root packages/perps --match-path test/perps/poc/LiquidationBatchKeeperPositionGas.t.sol -vv`
3. Expected: 5 passed. Add `-vvvv --match-test test_KeeperWithPositionBatchStopsAtIndexZero` to see `OutOfGas` in the keeper's `syncFromEngine` and `LiquidationBatchStopped(nextIndex: 0)`.

Do not set `FOUNDRY_VIA_IR=false`: legacy codegen makes the Engine about 5% cheaper and some long-account items then fit, so the fuzz test can fail there. The v2.0.0 release was built with via-IR.

What the tests show:

- (a) `test_Control_FlatKeeperBatchLiquidatesAll`: a keeper without a position liquidates all three underwater accounts in one batch (first item 609,996 gas).
- (b) `test_KeeperWithPositionBatchStopsAtIndexZero`: same state, the caller holds a short opened a day earlier. Three consecutive calls each return 0 and emit `LiquidationBatchStopped(0)`; all three accounts stay open and liquidatable; the keeper earns nothing.
- (c) `testFuzz_StopForAnyUnderwaterAccount`: any long or short account of 10,000 to 100,000 USDG notional; the keeper with a position stops at 0, a flat keeper liquidates the same account from the same state.
- (d) `test_Control_SinglePathWorksForKeeperWithPosition`: the same keeper liquidates each account through `executeLiquidation`.
- (e) `test_GasPerItem`: item budget 850,000; single-path gas 1,042,089 (flat caller) vs 1,328,193 (caller with a position), so the caller-side work outside the budget is 286,104 gas.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {LiquidationBatchTestFixture} from "../shared/LiquidationBatchFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A batch item gets minEngineGas + 250,000 gas (+600,000 per pending order). If the caller holds a position,
///         the Engine also syncs the caller's terminal curve and realizes its carry (CfdEngine.sol:1331-1338). That
///         work is outside the budget, so item 0 runs out of gas and the batch stops at index 0 on every call.
contract LiquidationBatchKeeperPositionGasTest is LiquidationBatchTestFixture {

    address internal constant TRADING_KEEPER = address(0xBA7CB0B1);
    address internal constant VICTIM_THREE = address(0xBA7C0005);
    bytes32 internal constant STOPPED = keccak256("LiquidationBatchStopped(uint256)");

    /// @dev DEPLOYMENT.md release risk parameters.
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
        address[] memory v = _victims();
        for (uint256 i; i < v.length; ++i) {
            _openUnderwater(v[i], CfdTypes.Side.LONG, 10_000e18);
        }
        // A keeper bot that also trades from its keeper address.
        _fundTrader(TRADING_KEEPER, 2000e6);
        _open(TRADING_KEEPER, CfdTypes.Side.SHORT, 10_000e18, 1000e6, NEUTRAL_PRICE);
        vm.warp(block.timestamp + 1 days);
    }

    /// @dev Margin 1% of notional: underwater after a 2% adverse move.
    function _openUnderwater(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        _fundTrader(account, size / 1e14 + 200e6);
        _open(account, side, size, size / 1e14, NEUTRAL_PRICE);
    }

    function _victims() internal pure returns (address[] memory v) {
        v = new address[](3);
        (v[0], v[1], v[2]) = (ELIGIBLE_ONE, ELIGIBLE_TWO, VICTIM_THREE);
    }

    function _batch(
        address keeper,
        address[] memory accounts,
        uint256 price
    ) internal returns (uint256 next, bool stopped) {
        bytes[] memory data = _liveLiquidationData(price);
        vm.recordLogs();
        vm.prank(keeper);
        next = IPerpsKeeper(address(router)).executeLiquidationBatch{gas: 30_000_000}(accounts, data);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == STOPPED) {
                stopped = true;
            }
        }
    }

    /// (a) Control: a flat keeper liquidates all three accounts.
    function test_Control_FlatKeeperBatchLiquidatesAll() public {
        assertEq(_positionSize(KEEPER), 0);
        (uint256 next, bool stopped) = _batch(KEEPER, _victims(), LIQUIDATION_PRICE);
        assertEq(next, 3);
        assertFalse(stopped);
        for (uint256 i; i < 3; ++i) {
            assertEq(_positionSize(_victims()[i]), 0, "liquidated");
        }
    }

    /// (b) Same state, caller with a position: nothing is liquidated; resuming at nextIndex stops again.
    function test_KeeperWithPositionBatchStopsAtIndexZero() public {
        uint256 keeperFunds = _settlementBalance(TRADING_KEEPER);
        for (uint256 attempt; attempt < 3; ++attempt) {
            (uint256 next, bool stopped) = _batch(TRADING_KEEPER, _victims(), LIQUIDATION_PRICE);
            assertEq(next, 0, "no account processed");
            assertTrue(stopped, "LiquidationBatchStopped(0)");
        }
        for (uint256 i; i < 3; ++i) {
            address v = _victims()[i];
            assertEq(_positionSize(v), 10_000e18, "still open");
            assertTrue(engineLens.previewLiquidation(v, LIQUIDATION_PRICE).liquidatable, "still liquidatable");
        }
        assertEq(_settlementBalance(TRADING_KEEPER), keeperFunds, "no bounty");
    }

    /// (c) Any underwater account without pending orders: long or short, 10k to 100k USDG notional.
    function testFuzz_StopForAnyUnderwaterAccount(
        bool isShort,
        uint256 lots
    ) public {
        address victim = address(0xBA7CF00D);
        uint256 size = bound(lots, 100, 1000) * 1e20;
        _openUnderwater(victim, isShort ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG, size);
        vm.warp(block.timestamp + 1 days);
        uint256 price = isShort ? 98_000_000 : LIQUIDATION_PRICE;
        address[] memory list = new address[](1);
        list[0] = victim;

        uint256 snap = vm.snapshotState();
        (uint256 next, bool stopped) = _batch(TRADING_KEEPER, list, price);
        assertEq(next, 0);
        assertTrue(stopped);
        assertEq(_positionSize(victim), size);

        vm.revertToState(snap);
        (next, stopped) = _batch(KEEPER, list, price);
        assertEq(next, 1);
        assertEq(_positionSize(victim), 0, "flat keeper liquidates the same account");
    }

    /// (d) The same keeper liquidates each account through executeLiquidation (no item cap).
    function test_Control_SinglePathWorksForKeeperWithPosition() public {
        for (uint256 i; i < 3; ++i) {
            address v = _victims()[i];
            vm.prank(TRADING_KEEPER);
            IPerpsKeeper(address(router)).executeLiquidation(v, _liveLiquidationData(LIQUIDATION_PRICE));
            assertEq(_positionSize(v), 0);
        }
    }

    /// (e) Item budget vs the extra work a caller position adds to the same liquidation.
    function test_GasPerItem() public {
        uint256 snap = vm.snapshotState();
        uint256 flat = _singleGas(KEEPER);
        vm.revertToState(snap);
        uint256 positioned = _singleGas(TRADING_KEEPER);
        emit log_named_uint("item budget, account without orders", router.minEngineGas() + 250_000);
        emit log_named_uint("executeLiquidation gas, flat caller", flat);
        emit log_named_uint("executeLiquidation gas, caller with position", positioned);
        emit log_named_uint("extra item work, not in the budget", positioned - flat);
    }

    function _singleGas(
        address keeper
    ) internal returns (uint256 used) {
        bytes[] memory data = _liveLiquidationData(LIQUIDATION_PRICE);
        vm.prank(keeper);
        used = gasleft();
        IPerpsKeeper(address(router)).executeLiquidation(ELIGIBLE_ONE, data);
        used -= gasleft();
        assertEq(_positionSize(ELIGIBLE_ONE), 0);
    }

}
```
