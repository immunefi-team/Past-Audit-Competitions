# \#98884 [SC-Low] Incomplete fix for Cecuro LOW-59: CfdEngine can still renounce ownership, making the degraded-mode latch permanent and freezing all LP settlement

**Submitted on Nov 10th 2026 at 18:19:05 UTC by @Rareone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98884
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Background: LOW-59 and its fix

Cecuro LOW-59 ("Owner can leave emergency recovery ownerless") found that governance could permanently remove its own recovery authority by calling the inherited `Ownable.renounceOwnership()`. The fix comment reads:

> LOW-59 — Addressed. HousePool, OrderRouterAdmin, and EmergencyPauseCoordinator override renounceOwnership() to revert, preventing governance from permanently removing their recovery and guardian-management authority through renunciation.

At a82d4fe the override exists in exactly those three contracts:

```solidity
// HousePool.sol:264-265 (OrderRouterAdmin.sol:124-125 and EmergencyPauseCoordinator.sol:101-102 are identical in intent)
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
```

`CfdEngine` did not get it. `CfdEngine is ... CfdTreasury ...`, and `abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step` (`CfdTreasury.sol:10`) has no override either, so `CfdEngine.renounceOwnership()` is the stock OpenZeppelin version and sets the owner to `address(0)`.

That matters because `CfdEngine`'s owner holds the **only** recovery path for the protocol's most consequential latch:

```solidity
// CfdEngine.sol:658
function clearDegradedMode() external onlyOwner {
// CfdEngine.sol:632
function recapitalizePool(uint256 amountAssets) external onlyOwner nonReentrant {
```

The fix applied the principle "governance must retain emergency recovery authority" to the pause and guardian recovery paths. It missed the insolvency recovery path in the engine. The program rules say a known root cause is valid if "the fix at commit a82d4fe is incomplete". This report shows that.

## What breaks

`degradedMode` latches on any close or liquidation that leaves effective assets below the maximum liability (`_enterDegradedModeIfInsolvent`, `CfdEngine.sol` ~:1351-1369). That's an ordinary market event, not an attack. It clears only through `clearDegradedMode()`, which is owner-only and requires solvency to be restored first.

While the latch is set:
- **all LP epoch settlement reverts** with `HousePool__DegradedMode()` (`HousePool._requireWithdrawalsLive`, `HousePool.sol:1864-1869`), for deposits and redemptions of both tranches;
- **new opens are rejected**;
- **trader withdrawals with a live position are blocked.**

If the engine owner has renounced, at any time before or during the latch, then:
- nobody can ever call `clearDegradedMode()`, even after the pool is fully solvent again;
- nobody can call `recapitalizePool()`;
- every LP redemption that matures afterwards can never be funded. This is a permanent freeze of LP principal.

The other owners don't help. The `HousePool` owner, the `OrderRouterAdmin` owner and the emergency guardian have no path to clear the engine's latch (test 4).


## Impact

- **Effect:** permanent freezing of LP funds once the latch is set. The PoC shows about 3,000,000 USDG of junior shares stuck after 395 days, where the control branch funds 2,607,520 USDG of the same request. New positions can never be opened, and traders with open positions can't withdraw margin.
- **Trigger:** the governance owner calls `renounceOwnership()` on `CfdEngine`. This is a trusted-role action. It's exactly the action the team chose to block on the three sibling contracts, with NatSpec saying governance must retain recovery authority. An operator who knows that `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator` refuse renunciation could reasonably assume the core engine does too.


## Recommendation

Apply the LOW-59 override to every contract whose owner holds a recovery path, starting with `CfdEngine`. The simplest place is `CfdTreasury`, so every engine deployment inherits it:

```solidity
/// @notice Ownership cannot be renounced because governance must retain degraded-mode and recapitalization recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert CfdEngine__RenounceDisabled();
}
```

Review `CfdEngineAdmin` and `MarginClearinghouse` too. They are also still renounceable at this commit (test 2). Check whether their owners hold any recovery-only function, such as configuration finalisation or the engine binding, before deciding whether to block renunciation there as well.


## Proof of Concept

## Proof of Concept



File: `packages/perps/test/perps/P_RenounceLow59.t.sol` (`BasePerpTest`).

Degraded mode is entered through a real engine path, the same scenario the repo's own degraded-settlement tests use:
1. A SHORT of 999,000 and a LONG of 500,000 open at 1.00.
2. The LONG closes profitably at 0.20, which reveals E < L and latches `degradedMode`.

Then the pool is recapitalised with 3,000,000 USDG and the `HousePool` owner assigns the unassigned assets to a junior LP (`lp2`), who requests a full redemption. This step only gives the LP real value to redeem.

| Test | What it shows | Result |
|---|---|---|
| `test_Control_FixedContractsRejectRenounce` | As owner, `renounceOwnership()` reverts on `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`. The LOW-59 fix is in place there | PASS |
| `test_CfdEngineRenounceSucceeds` | `engine.renounceOwnership()` succeeds and `engine.owner() == address(0)`. `CfdEngineAdmin` and `MarginClearinghouse` are also renounceable | PASS |
| `test_Control_OwnerClearsLatchAndDegradedBlockerDisappears` | Without renouncing: settlement reverts `HousePool__DegradedMode`, then the owner calls `clearDegradedMode()` and lp2's epoch settles, with **2,607,520.54 USDG claimable**, and opens work again | PASS |
| `test_DegradedLatchBecomesPermanentAfterRenounce` | After renouncing: `clearDegradedMode()` reverts for every real caller and `recapitalizePool()` reverts `OwnableUnauthorizedAccount`. LP settlement still reverts `HousePool__DegradedMode` at **+30 days and +395 days**; lp2's full redemption is still pending with **0 claimable**; new opens are rejected; the trader cannot withdraw margin with a live position | PASS |
| `test_Control_NonEngineOwnerCannotRescue` | With the engine owned by a separate EOA: the `HousePool` owner, `OrderRouterAdmin` owner and a stranger cannot clear the latch, and only the engine owner can | PASS |

Run:

```bash
forge test --root packages/perps --match-path test/perps/P_RenounceLow59.t.sol -vv
```

Output:

```
[PASS] test_CfdEngineRenounceSucceeds()
[PASS] test_Control_FixedContractsRejectRenounce()
[PASS] test_Control_NonEngineOwnerCannotRescue()
[PASS] test_Control_OwnerClearsLatchAndDegradedBlockerDisappears()
[PASS] test_DegradedLatchBecomesPermanentAfterRenounce()
Suite result: ok. 5 passed; 0 failed; 0 skipped

Control (owner clears the latch):
  lp2 junior shares after assignment: 4963714080590291
  lp2 share value (assets):           2999999999999   (~3,000,000 USDG)
  control: settle revert selector (0 = settled): 0x00000000
  control: lp2 claimable redeem assets: 2607520541708   (2,607,520.54 USDG funded)

Renounced engine owner:
  lp2 junior shares after assignment: 4963714080590291
  lp2 share value (assets):           2999999999999
  lp2 pending redeem shares after 395d:  4963714080590291   (all still pending)
  lp2 claimable redeem assets after 395d: 0
  trader withdraw 40k with live position ok? (1=yes): 0
```

Note: the PoC pranks only real callers. `address(0)` would pass `onlyOwner` once the owner is zero, but no transaction can originate from it.

Full test:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

interface IHousePoolErr {
    error HousePool__DegradedMode();
}

contract P_RenounceLow59Test is BasePerpTest {
    address longTrader = address(0xB011);
    address shortTrader = address(0xBEA2);
    address newTrader = address(0xCAFE);
    address lp = address(0x1F0);
    address stranger = address(0x5712);
    address engOwner = address(0xE0E0);

    // Risk params copied from DegradedSettlementBehavior (the scenario that latches degraded mode needs
    // vpi 0 / maxSkew 1e18); other params as release.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 5e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    uint256 shares;

    function setUp() public override {
        super.setUp();
        _fundTrader(longTrader, 100_000e6);
        _fundTrader(shortTrader, 100_000e6);
        _fundTrader(newTrader, 100_000e6);
        shares = _fundJuniorDelayed(lp, 10_000e6);
        vm.warp(block.timestamp + 2 hours);
        vm.prank(lp);
        juniorVault.requestRedeem(shares / 2, lp, lp);
    }

    // Real path: engine.processOrderTyped open short 999k + long 500k at price 1.00, then profitable long close at 0.20
    // (price falls so long profits) revealing E < L and latching degradedMode.
    function _degrade() internal {
        _open(shortTrader, CfdTypes.Side.SHORT, 999_000e18, 50_000e6, 1e8);
        _open(longTrader, CfdTypes.Side.LONG, 500_000e18, 50_000e6, 1e8);
        _close(longTrader, CfdTypes.Side.LONG, 500_000e18, 20_000_000);
        assertTrue(engine.degradedMode(), "must be degraded");
    }

    function _recap() internal {
        settlementToken.mint(address(this), 3_000_000e6);
        settlementToken.approve(address(engine), 3_000_000e6);
        engine.recapitalizePool(3_000_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function test_Control_FixedContractsRejectRenounce() public {
        EmergencyPauseCoordinator coord = new EmergencyPauseCoordinator(address(routerAdmin), address(pool), address(this));
        address[3] memory cs = [address(pool), address(routerAdmin), address(coord)];
        for (uint256 i; i < 3; i++) {
            address o = Ownable(cs[i]).owner();
            vm.prank(o);
            (bool ok,) = cs[i].call(abi.encodeWithSignature("renounceOwnership()"));
            assertFalse(ok, "renounce must revert");
            assertEq(Ownable(cs[i]).owner(), o, "owner unchanged");
        }
    }

    function test_CfdEngineRenounceSucceeds() public {
        assertEq(engine.owner(), address(this));
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        // also the other unprotected ones
        address eo = engineAdmin.owner();
        vm.prank(eo);
        engineAdmin.renounceOwnership();
        assertEq(engineAdmin.owner(), address(0), "engineAdmin renounced");
        address co = clearinghouse.owner();
        vm.prank(co);
        clearinghouse.renounceOwnership();
        assertEq(clearinghouse.owner(), address(0), "clearinghouse renounced");
    }

    /// Returns the 4-byte selector settleLpEpoch reverts with (0 if it does not revert).
    function _settleSelector() internal returns (bytes4 sel) {
        _refreshMark();
        try this.externalSettle() {
            sel = bytes4(0);
        } catch (bytes memory err) {
            if (err.length >= 4) sel = bytes4(err);
        }
    }

    address lp2 = address(0x1F2);

    /// Pool owner (still present) assigns the recapitalised unassigned assets to a junior LP, who then requests a full redeem.
    function _fundedLpRedeem() internal returns (uint256 rid, uint256 lpShares) {
        _refreshMark();
        pool.assignUnassignedAssets(false, lp2);
        lpShares = juniorVault.balanceOf(lp2);
        emit log_named_uint("lp2 junior shares after assignment", lpShares);
        emit log_named_uint("lp2 share value (assets)", juniorVault.convertToAssets(lpShares));
        vm.warp(block.timestamp + 2 hours);
        vm.prank(lp2);
        rid = juniorVault.requestRedeem(lpShares, lp2, lp2);
        vm.warp(block.timestamp + 3 hours);
    }

    /// Router-path LP settlement (the test helper settles as the router, the production caller).
    function externalSettle() external {
        _settleLpEpochForTest();
    }

    function test_Control_OwnerClearsLatchAndDegradedBlockerDisappears() public {
        _degrade();
        _recap();
        assertEq(_settleSelector(), IHousePoolErr.HousePool__DegradedMode.selector, "degraded blocks LP settlement");
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "control: owner cleared the latch");
        (uint256 rid,) = _fundedLpRedeem();
        bytes4 sel = _settleSelector();
        emit log_named_bytes32("control: settle revert selector (0 = settled)", bytes32(sel));
        assertEq(sel, bytes4(0), "control: LP epoch settles once the owner clears the latch");
        uint256 claimable = juniorVault.claimableRedeemAssets(rid, lp2);
        emit log_named_uint("control: lp2 claimable redeem assets", claimable);
        assertGt(claimable, 0, "control: lp2 redemption funded");
        assertTrue(_tryOpenNew(), "control: opens work again after clear");
    }

    function test_DegradedLatchBecomesPermanentAfterRenounce() public {
        _degrade();
        _recap();
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        address[4] memory callers = [address(this), stranger, address(pool), address(router)];
        for (uint256 i; i < 4; i++) {
            vm.prank(callers[i]);
            (bool ok,) = address(engine).call(abi.encodeWithSignature("clearDegradedMode()"));
            assertFalse(ok, "clear must revert for every caller");
        }
        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(engine), 1e6);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.recapitalizePool(1e6);
        assertTrue(engine.degradedMode(), "latched");

        (uint256 rid, uint256 lpShares) = _fundedLpRedeem();
        vm.warp(block.timestamp + 30 days);
        assertEq(_settleSelector(), IHousePoolErr.HousePool__DegradedMode.selector, "30d: LP settlement blocked by latch");
        vm.warp(block.timestamp + 365 days);
        assertEq(_settleSelector(), IHousePoolErr.HousePool__DegradedMode.selector, "395d: still blocked by latch");
        emit log_named_uint("lp2 pending redeem shares after 395d", juniorVault.pendingRedeemRequest(rid, lp2));
        emit log_named_uint("lp2 claimable redeem assets after 395d", juniorVault.claimableRedeemAssets(rid, lp2));
        assertEq(juniorVault.pendingRedeemRequest(rid, lp2), lpShares, "lp2 redemption never funded");
        assertFalse(_tryOpenNew(), "new opens rejected while latched");

        (uint256 sz,,,,,,) = engine.positions(shortTrader);
        assertGt(sz, 0, "short still live");
        vm.prank(shortTrader);
        (bool wOk,) = address(clearinghouse).call(abi.encodeWithSignature("withdrawMargin(uint256)", uint256(40_000e6)));
        emit log_named_uint("trader withdraw 40k with live position ok? (1=yes)", wOk ? 1 : 0);
    }

    function _refreshMark() internal {
        uint256 mp = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(mp == 0 ? 1e8 : mp, uint64(block.timestamp));
    }

    function _refreshAndSettle() internal {
        _refreshMark();
        _settleLpEpochForTest();
    }

    function _tryOpenNew() internal returns (bool ok) {
        CfdTypes.Order memory o = CfdTypes.Order({
            account: newTrader,
            sizeDelta: 10_000e18,
            marginDelta: 1000e6,
            targetPrice: 1e8,
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: CfdTypes.Side.LONG,
            isClose: false
        });
        uint256 d = pool.totalAssets();
        vm.prank(address(router));
        (ok,) = address(engine).call(
            abi.encodeWithSignature(
                "processOrderTyped((address,uint256,uint256,uint256,uint64,uint64,uint64,uint8,bool),uint256,uint256,uint64)",
                o, 1e8, d, uint64(block.timestamp)
            )
        );
    }

    function test_Control_NonEngineOwnerCannotRescue() public {
        _degrade();
        _recap();
        // engine owner becomes a distinct EOA; this (pool owner, routerAdmin owner) is no longer engine owner
        engine.transferOwnership(engOwner);
        vm.prank(engOwner);
        engine.acceptOwnership();
        address[4] memory callers = [address(this), routerAdmin.owner(), pool.owner(), stranger];
        for (uint256 i; i < 4; i++) {
            vm.prank(callers[i]);
            (bool ok,) = address(engine).call(abi.encodeWithSignature("clearDegradedMode()"));
            assertFalse(ok, "non engine owner cannot clear");
        }
        assertTrue(engine.degradedMode());
        // positive control: engine owner can
        vm.prank(engOwner);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());
    }
}
```
