# \#99919 [SC-Low] Incomplete renounceOwnership fix: CfdEngine (via CfdTreasury), CfdEngineAdmin and MarginClearinghouse remain one-call renounceable, making a degradedMode latch permanently irreversible and freezing LP withdrawals/settlement with no on-chain recovery

**Submitted on Nov 15th 2026 at 18:07:27 UTC by @niroh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99919
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro
The protocol previously acknowledged (and fixed) a finding where the owner could leave emergency-recovery authority unusable by renouncing ownership. The fix added reverting `renounceOwnership()` overrides to only three contracts: `OrderRouterAdmin`, `HousePool` and `EmergencyPauseCoordinator`. Three other ownable contracts - `CfdTreasury` (the abstract base of `CfdEngine`), `CfdEngineAdmin` and `MarginClearinghouse` - still inherit OpenZeppelin's one-transaction `renounceOwnership()`. A single call zeroes the owner to `address(0)` irreversibly. If the engine later enters degraded mode (which latches automatically from ordinary closes/liquidations), the sole recovery function `clearDegradedMode()` is owner-gated and therefore permanently unreachable: LP epoch settlement, LP withdrawals, matured-escrow cancellation, withdrawals for positioned traders and all new risk are blocked forever, freezing LP capital and trader free margin with no on-chain recovery path.

## Vulnerability Details
The remediation for "Owner can leave emergency recovery ownerless" covered only three of the six ownable in-scope contracts. The fixed contracts, for comparison:

```solidity
// packages/perps/src/OrderRouterAdmin.sol:125-127
function renounceOwnership(...) { revert ...; }
// packages/perps/src/HousePool.sol:265-267 — same reverting override
// packages/perps/src/EmergencyPauseCoordinator.sol:102-104 — same reverting override
```

The unfixed contracts keep plain `Ownable2Step` with no override:

- [`packages/perps/src/CfdTreasury.sol:5-40`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L5-L40) - abstract base of `CfdEngine`; constructed with `Ownable(msg.sender)` at line 39 and contains no `renounceOwnership` override, so `CfdEngine` inherits OZ's one-call renounce.
- [`packages/perps/src/CfdEngineAdmin.sol:17-96`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L17-L96) - plain `Ownable2Step`, no override.
- `packages/perps/src/MarginClearinghouse.sol:30, 298` - plain `Ownable2Step`, no override.

The faulty assumption is that the emergency-recovery protection from the earlier fix is complete. It is not: a single `renounceOwnership()` transaction on the engine irreversibly sets `owner = address(0)`, and `clearDegradedMode` - the only place `degradedMode` is set back to `false` in the codebase ([`packages/perps/src/CfdEngine.sol:655-666`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L666)) - is `onlyOwner`:

```solidity
// packages/perps/src/CfdEngine.sol:655-666
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

Degraded mode latches automatically from ordinary closes/liquidations via `_enterDegradedModeIfInsolvent` ([`packages/perps/src/CfdEngine.sol:1355-1366`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1355-L1366)) whenever a settlement leaves effective pool assets below the larger side's maximum liability. While latched:

- Every trader withdrawal for a positioned account hard-reverts ([`packages/perps/src/CfdEngineSettlementSidecar.sol:274-276`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276)).
- LP epoch settlement reverts ([`packages/perps/src/libraries/HousePoolFreshnessLib.sol:43-46`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolFreshnessLib.sol#L43-L46); `HousePool__DegradedMode`), and matured Junior deposit/redemption escrow cancellation is blocked ([`packages/perps/src/HousePool.sol:1864-1870`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870)).
- All new risk (opens/increases) reverts (`packages/perps/src/CfdEnginePlanLib.sol:318`; `OrderLiquidationBatchLogic.sol:328`).

This is a missing access-control hardening / irrecoverable-state class of vulnerability: an inherited OpenZeppelin `renounceOwnership()` that was supposed to be disabled on every privileged contract but was only disabled on three.

State required: (1) one `renounceOwnership()` call on `CfdEngine`, `CfdEngineAdmin` and/or `MarginClearinghouse`; (2) any subsequent close/liquidation that latches `degradedMode` on the engine. The `CfdEngineAdmin` renounce additionally freezes the 72-hour propose/finalize/cancel configuration state machine - an in-flight proposal can neither be finalized nor cancelled, so risk/fee/calendar/freshness/router configuration is permanently frozen. A `MarginClearinghouse` renounce makes treasury rotation and non-USDG token rescue unreachable.

## Impact Details
This maps to Immunefi's *Permanent freezing of funds* - Critical

Worst case: after renounce, any ordinary close/liquidation that latches `degradedMode` converts all LP withdrawal/settlement paths and positioned-account withdrawals into permanent reverts. `clearDegradedMode()` reverts with `OwnableUnauthorizedAccount` for every caller (owner is `address(0)`), and `recapitalizePool` - the recapitalization path an owner would use to restore solvency - is likewise unreachable. All LP capital in the Senior/Junior tranches and all trader free settlement held in the `MarginClearinghouse` become permanently frozen; there is no in-scope code path that ever sets `degradedMode = false` outside the owner-gated function. 



## Proof of Concept

Repo: `immunefi-team-audit-comp-plether`
Commit: `a82d4fef…` (frozen audit commit)

Setup:
```bash
git checkout a82d4fef
forge install
forge test --match-contract RenounceDegradedLatchPoC -vvv
```

Place the following in `packages/perps/test/perps/`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract RenounceDegradedLatchPoC is BasePerpTest {

    address longTrader = address(0xD001);
    address shortTrader = address(0xD002);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _enterDegradedMode() internal {
        _fundTrader(longTrader, 100_000e6);
        _fundTrader(shortTrader, 100_000e6);
        _open(shortTrader, CfdTypes.Side.SHORT, 999_000e18, 50_000e6, 1e8);
        _open(longTrader, CfdTypes.Side.LONG, 500_000e18, 50_000e6, 1e8);
        _close(longTrader, CfdTypes.Side.LONG, 500_000e18, 20_000_000);
    }

    function test_PoC_RenounceMakesDegradedLatchIrreversible() public {
        // Fixed contracts revert on renounce (for comparison)
        vm.expectRevert();
        pool.renounceOwnership();

        // 1. Unfixed contracts: one-call renounce succeeds
        assertEq(engine.owner(), address(this));
        engine.renounceOwnership();
        engineAdmin.renounceOwnership();
        clearinghouse.renounceOwnership();
        assertEq(engine.owner(), address(0), "engine ownerless");
        assertEq(engineAdmin.owner(), address(0), "engineAdmin ownerless");
        assertEq(clearinghouse.owner(), address(0), "clearinghouse ownerless");

        // 2. Ordinary trading latches degraded mode
        _enterDegradedMode();
        assertTrue(engine.degradedMode(), "degraded latched");

        // 3. Even after the pool is fully recapitalized (solvency restored), latch cannot be cleared
        settlementToken.mint(address(pool), 500_000e6);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            500_000e6, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        vm.prank(address(juniorVault));
        pool.reconcile();

        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.clearDegradedMode();
        // nobody, incl. address(0)-impersonation-free callers, can clear it
        vm.prank(address(0xBEEF));
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(0xBEEF)));
        engine.clearDegradedMode();
        // recapitalizePool also gone
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.recapitalizePool(1e6);
        assertTrue(engine.degradedMode(), "still degraded");

        // 4. LP epoch settlement is blocked forever
        vm.warp(block.timestamp + 1 hours + 1);
        uint256 requestedShares = juniorVault.estimateWithdrawShares(1e6);
        uint256 requestId = juniorVault.requestRedeem(requestedShares, address(this), address(this));
        vm.warp(pool.lpEpochStart(requestId));
        uint256 markPrice = engine.lastMarkPrice();
        if (markPrice == 0) markPrice = 1e8;
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice, uint64(vm.getBlockTimestamp()));
        uint256 markTime = engine.lastMarkTime();
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        vm.prank(address(router));
        pool.settleLpEpoch(markPrice, markTime);

        // 5. Engine admin config machine is frozen
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = _riskParams();
        config.executionFeeBps = engine.executionFeeBps();
        config.frozenCloseSpreadBps = engine.frozenCloseSpreadBps();
        config.settlementBufferBps = engine.settlementBufferBps();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engineAdmin.proposeRiskConfig(config);

        // 6. Clearinghouse rescue unreachable
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        clearinghouse.rescueToken(address(0x1234), address(this));
    }

}
```

Expected output: `pool.renounceOwnership()` reverts (the fixed contract), while `engine.renounceOwnership()`, `engineAdmin.renounceOwnership()` and `clearinghouse.renounceOwnership()` succeed in one call each, leaving `owner() == address(0)` on all three. Ordinary trading then latches `engine.degradedMode = true`. Even after the pool is fully recapitalized to solvency, `clearDegradedMode()` and `recapitalizePool()` revert with `OwnableUnauthorizedAccount` for every caller, and `degradedMode` remains `true`. `HousePool.settleLpEpoch` then reverts with `HousePool__DegradedMode` on every call, so LP redemptions can never settle. `CfdEngineAdmin.proposeRiskConfig` and `MarginClearinghouse.rescueToken` also revert.

Why the test demonstrates the bug: the test establishes that (a) the renounce guard is inconsistently applied - three ownable in-scope contracts remain one-call renounceable; (b) `degradedMode` latches from ordinary trading alone; (c) even with solvency restored, the latch cannot be cleared because `CfdEngine.sol:665` inside the `onlyOwner` `clearDegradedMode` is the only place `degradedMode` is set back to `false` and the owner is `address(0)`; and (d) while latched, LP epoch settlement is permanently blocked. The freeze of LP capital is therefore permanent.

