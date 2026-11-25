# \#99556 [SC-Low] CfdEngine remains renounceable after LOW-59 remediation, permanently disabling degraded-mode recovery

**Submitted on Nov 14th 2026 at 07:55:42 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99556
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Brief / Introduction

I found that the remediation for Cecuro **LOW-59 — “Owner can leave emergency recovery ownerless”** does not cover `CfdEngine`, even though the Engine holds Plether's two essential recovery actions: `clearDegradedMode()` and `recapitalizePool()`.

The LOW-59 fix prevents ownership renunciation in `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`. However, `CfdEngine` still inherits the unrestricted OpenZeppelin `Ownable2Step.renounceOwnership()` through `CfdTreasury`. The same protection is absent from `CfdEngineAdmin` and `MarginClearinghouse`.

This leaves an irreversible state transition in the recovery layer. After the Engine owner calls `renounceOwnership()`, `owner()` becomes `address(0)` and there is no remaining account that can call the Engine's `onlyOwner` recovery functions. If degraded mode is active, the system remains in that state after the underlying reserve condition has recovered. LP epoch settlement and the related withdrawal capacity remain blocked because clearing the latch is no longer possible through the deployed ownership mechanism.

I reproduced the missing override and its operational consequence in **two passing Foundry tests** at the exact audited commit. The second test triggers degraded mode through ordinary trading, removes Engine ownership, confirms both recovery calls fail, closes the remaining position, advances **30 days**, and verifies that LP settlement is still rejected.

### Vulnerability Details

#### A. LOW-59's ownership safeguard was applied to only three contracts

The previously acknowledged LOW-59 finding identified the risk of making emergency recovery authority ownerless and recommended preventing renunciation on every governance-bearing perps contract. The recorded fix is commit [`1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6`](https://github.com/Plether-Fi/plether-core/commit/1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6).

The fix added this override to `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`:

```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

I verified the overrides at:

- [`HousePool.sol:265`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L265)
- [`OrderRouterAdmin.sol:125`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L125)
- [`EmergencyPauseCoordinator.sol:102`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/EmergencyPauseCoordinator.sol#L102)

`CfdEngine` has no matching override. Its inheritance path is:

```text
CfdEngine
  -> CfdTreasury
      -> Ownable2Step
          -> Ownable.renounceOwnership()
```

[`CfdTreasury.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol) inherits `Ownable2Step`, while the Engine's deployed code does not prevent the inherited renunciation call. The same missing safeguard exists in [`CfdEngineAdmin.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol) and [`MarginClearinghouse.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol).

My first PoC test calls `renounceOwnership()` on each contract using its current owner. `HousePool` correctly reverts; `CfdEngine`, `CfdEngineAdmin`, and `MarginClearinghouse` all accept the call and end with `owner() == address(0)`.

#### B. The Engine controls the only documented degraded-mode recovery path

[`CfdEngine.sol:658–667`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667) defines:

```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

This function can clear the state only after the reserve check passes, and only when a caller satisfies `onlyOwner`. There is no permissionless recovery branch when `owner()` is zero.

The funding-side recovery function, [`recapitalizePool()` at `CfdEngine.sol:632–653`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653), is also `onlyOwner`. Therefore the loss of ownership simultaneously disables **clearing containment** and **supplying the designated recapitalization funding** through the Engine.

The degraded-mode latch itself is set by [`_enterDegradedModeIfInsolvent()` at `CfdEngine.sol:1351–1368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368). Once triggered, it is sticky until `clearDegradedMode()` succeeds. Renouncing Engine ownership removes the only caller capable of satisfying that recovery condition.

#### C. LP settlement remains blocked after the trading condition is resolved

LP epoch settlement enters [`HousePool.settleLpEpoch()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L831), which reaches the degraded-mode guard and reverts with `HousePool__DegradedMode` while the Engine's flag remains active.

In the PoC I first enter degraded mode through the Router trading sequence using the release-style risk parameters. I then renounce Engine ownership and confirm that the previous owner receives `OwnableUnauthorizedAccount` from both `clearDegradedMode()` and `recapitalizePool()`.

Next I close the remaining position and verify the tested position size is zero. I advance the timestamp by 30 days, refresh the mark, and call LP epoch settlement. The result remains:

```text
HousePool__DegradedMode()
```

Both tranche withdrawal-cap views also return zero:

```text
getMaxJuniorWithdraw() == 0
getMaxSeniorWithdraw() == 0
```

Unlike the three contracts protected by LOW-59, the Engine can still be placed into this ownerless recovery state at the audited commit. The protected `HousePool` ownership does not restore control over the Engine's separate `degradedMode` flag.

### Impact Details

The impact is the loss of the protocol's recovery authority, not merely the existence of an unused inherited method. The PoC establishes that:

1. `CfdEngine` accepts a renunciation that the fixed recovery contracts reject.
2. The Engine's two central recovery entrypoints become inaccessible immediately afterward.
3. The degraded-mode restriction continues after the remaining tested trader position is closed.
4. LP epoch settlement continues to revert even after 30 simulated days.
5. Senior and Junior withdrawal capacity both remain zero for the tested degraded state.

Because LP deposits activate and redemptions are funded through epoch settlement, a persistent block on that function prevents those normal settlement transitions until Engine recovery authority is restored. The `Ownable2Step` inheritance does not provide an authorized caller who can simply reclaim ownership after it has been set to the zero address.

The same missed ownership protection in `CfdEngineAdmin` and `MarginClearinghouse` extends the gap to risk-configuration administration and Clearinghouse owner-only operations. The demonstrated irreversible degraded-mode effect, however, comes specifically from `CfdEngine`.

### Recommended Remediation

Apply the non-renounceable ownership protection consistently across contracts that hold recovery or configuration authority. The most direct fix is to put the override in `CfdTreasury` so `CfdEngine` inherits it, and apply an equivalent override to `CfdEngineAdmin` and `MarginClearinghouse`.

```solidity
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

Keep the existing two-step ownership transfer process. Add regression tests for every recovery-bearing contract, verifying that renunciation reverts, ownership remains unchanged, and the Engine can still clear degraded mode after reserve coverage is restored.

### References

- [Cecuro LOW-59, *Owner can leave emergency recovery ownerless* — audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [LOW-59 remediation commit `1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6`](https://github.com/Plether-Fi/plether-core/commit/1151d01fd30d5fe6f6d80d604c5ddd4b90845dc6)
- [`CfdEngine.sol` — only-owner recapitalization and degraded-mode recovery](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L667)
- [`CfdTreasury.sol` — inherited `Ownable2Step`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol)
- [`HousePool.sol` — blocked LP settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L831)
- [`CfdEngineAdmin.sol` — unprotected ownership inheritance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol)
- [`MarginClearinghouse.sol` — unprotected ownership inheritance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol)
- [Plether competition scope and in-scope contracts](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept


### Reproduction environment

- **Repository:** `https://github.com/immunefi-team/audit-comp-plether`
- **Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Solidity:** `0.8.35`
- **Framework:** Foundry
- **PoC path:** `packages/perps/test/perps/insight/Low59IncompleteFixPoC.t.sol`
- **Fixture:** repository `BasePerpTest` with a test Router harness and mocked external Pyth / USDG dependencies; no in-scope production contracts are modified.

The test consists of two independent cases:

**`test_RenounceStillPossibleOnRecoveryContracts()`** demonstrates the incomplete protection. `HousePool.renounceOwnership()` reverts while Engine, EngineAdmin and Clearinghouse renunciation succeeds. Each unprotected contract's ownership becomes zero.

**`test_RenouncedEngineMakesDegradedModePermanent()`** demonstrates the recovery consequence. Normal market activity enters degraded mode, Engine ownership is renounced, both owner-only recovery calls revert, the remaining tested position is closed, and a settlement attempt after 30 days still reverts with `HousePool__DegradedMode`. Both LP withdrawal limits are zero.

### Commands

Place the Solidity file included below at `packages/perps/test/perps/insight/Low59IncompleteFixPoC.t.sol`, then run:

```bash
cd /workspaces/audit-comp-plether/packages/perps
export PATH="$HOME/.foundry/bin:$PATH"

test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8" || exit 1

export FOUNDRY_TEST=test/perps/insight
export FOUNDRY_OUT=out-low59-final
export FOUNDRY_CACHE_PATH=cache-low59-final

forge test --offline \
  --match-path 'test/perps/insight/Low59IncompleteFixPoC.t.sol' \
  --match-contract '^Low59IncompleteFixPoC$' \
  -vv

git diff -- src/
```

### Verified Foundry results

I ran the two tests against the frozen commit. Both passed:

```text
Ran 2 tests for test/perps/insight/Low59IncompleteFixPoC.t.sol:Low59IncompleteFixPoC
[PASS] test_RenounceStillPossibleOnRecoveryContracts() (gas: 55400)
[PASS] test_RenouncedEngineMakesDegradedModePermanent() (gas: 19248553)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The production-source check `git diff -- src/` produced no changes.

### Complete Solidity PoC

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @notice PoC for an incomplete fix of Cecuro LOW-59 ("Owner can leave emergency recovery ownerless").
///         The fix overrides renounceOwnership() to revert only on HousePool, OrderRouterAdmin and
///         EmergencyPauseCoordinator. CfdEngine (through CfdTreasury), CfdEngineAdmin and MarginClearinghouse still
///         inherit Ownable2Step.renounceOwnership(), although CfdEngine owns the protocol's emergency recovery
///         actions (clearDegradedMode, recapitalizePool). After a renounce, a degraded-mode latch can never be
///         cleared, so LP epoch settlement (every LP redemption and deposit activation) and all new opens stay
///         frozen permanently.
contract Low59IncompleteFixPoC is BasePerpTest {

    address internal constant LONG_ACCT = address(0xA11CE);
    address internal constant SHORT_ACCT = address(0xB0B);
    address internal constant KEEPER = address(0xCAFE);

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

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(orderId, update);
    }

    /// @dev The fixed contracts reject renunciation; the recovery-owning engine, its admin module and the
    ///      clearinghouse do not.
    function test_RenounceStillPossibleOnRecoveryContracts() public {
        // Fixed by LOW-59: HousePool reverts.
        vm.prank(pool.owner());
        vm.expectRevert();
        pool.renounceOwnership();

        // Not fixed: CfdEngine, CfdEngineAdmin and MarginClearinghouse can still be renounced in one call.
        vm.prank(engine.owner());
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "engine ownerless");

        vm.prank(engineAdmin.owner());
        engineAdmin.renounceOwnership();
        assertEq(engineAdmin.owner(), address(0), "engine admin ownerless");

        vm.prank(clearinghouse.owner());
        clearinghouse.renounceOwnership();
        assertEq(clearinghouse.owner(), address(0), "clearinghouse ownerless");
    }

    /// @dev After the engine owner renounces, degraded mode becomes irreversible on-chain, which permanently freezes
    ///      LP epoch settlement and blocks new trading. The degraded latch can be entered through ordinary market
    ///      activity; it is reached here through the production router with release parameters.
    function test_RenouncedEngineMakesDegradedModePermanent() public {
        _fundTrader(LONG_ACCT, 60_000e6);
        _fundTrader(SHORT_ACCT, 20_000e6);
        uint256 step = 250_000e18;
        for (uint256 i = 0; i < 3; ++i) {
            _commitAndExecute(LONG_ACCT, CfdTypes.Side.LONG, step, 2000e6, false, 1e8);
            _commitAndExecute(SHORT_ACCT, CfdTypes.Side.SHORT, step, 2000e6, false, 1e8);
        }
        _commitAndExecute(LONG_ACCT, CfdTypes.Side.LONG, 245_000e18, 2000e6, false, 1e8);
        _commitAndExecute(SHORT_ACCT, CfdTypes.Side.SHORT, 240_000e18, 2000e6, false, 1e8);
        vm.prank(LONG_ACCT);
        engine.addMargin(LONG_ACCT, 30_000e6);
        (uint256 shortSize,,,,,,) = engine.positions(SHORT_ACCT);
        _commitAndExecute(SHORT_ACCT, CfdTypes.Side.SHORT, shortSize, 0, true, 1.02e8);
        assertTrue(engine.degradedMode(), "engine degraded");

        // Governance renounces engine ownership (still permitted, unlike HousePool/OrderRouterAdmin/coordinator).
        address formerOwner = engine.owner();
        vm.prank(formerOwner);
        engine.renounceOwnership();

        // The recovery actions are now unreachable for everyone, including the former owner.
        vm.prank(formerOwner);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, formerOwner));
        engine.clearDegradedMode();
        vm.prank(formerOwner);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, formerOwner));
        engine.recapitalizePool(1_000_000e6);

        // Even after the book is flattened, the latch persists: only the (now nonexistent) owner can clear it.
        (uint256 longSize,,,,,,) = engine.positions(LONG_ACCT);
        _commitAndExecute(LONG_ACCT, CfdTypes.Side.LONG, longSize, 0, true, 1.02e8);
        (longSize,,,,,,) = engine.positions(LONG_ACCT);
        assertEq(longSize, 0, "book flat");
        assertTrue(engine.degradedMode(), "degraded mode is permanent");

        // LP epoch settlement stays frozen, so queued redemptions and deposit activations can never settle.
        vm.warp(vm.getBlockTimestamp() + 30 days);
        vm.prank(address(router));
        engine.updateMarkPrice(1.02e8, SafeCast.toUint64(vm.getBlockTimestamp()));
        uint256 markPrice = engine.lastMarkPrice();
        uint256 markTime = engine.lastMarkTime();
        vm.prank(address(router));
        vm.expectRevert(bytes4(keccak256("HousePool__DegradedMode()")));
        pool.settleLpEpoch(markPrice, markTime);
        assertEq(pool.getMaxJuniorWithdraw(), 0, "junior withdrawals frozen");
        assertEq(pool.getMaxSeniorWithdraw(), 0, "senior withdrawals frozen");
    }

}
```

