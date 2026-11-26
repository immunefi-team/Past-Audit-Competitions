# \#99367 [SC-Insight] No contract can call HousePool.reconcile(), yet its NatSpec and the security notes describe it as a live vault hook

**Submitted on Nov 13th 2026 at 11:51:20 UTC by @krlz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99367
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

## Brief/Intro

`HousePool.reconcile()` books the elapsed senior coupon and the marked gains and losses into the pool's stored tranche principals. Only the two tranche vaults may call it, and neither vault does, so in production the pool reconciles only inside LP epoch settlement and three owner actions. The NatSpec, the interface and the security notes still describe a live vault hook, and the notes promise that reconciliation stays live during a settlement hold, when nothing can reconcile. The dead hook also takes 100 bytes of `HousePool`, whose runtime code is 5 bytes below the EIP-170 size limit.

## Vulnerability Details

The hook admits only the two vaults:

```solidity
// HousePool.sol#L206-L211, HousePool.sol#L1471-L1473
modifier onlyVault() {
    if (msg.sender != seniorVault && msg.sender != juniorVault) {
        revert HousePool__NotAVault();
    }
    _;
}

function reconcile() external onlyVault {
    _reconcile(_getHousePoolInputSnapshot());
}
```

Both vault addresses are set once and point to the two `TrancheVault` instances, whose only state-changing calls into the pool are `reserveSeniorDeposit` and `releaseSeniorDepositReservation` (`TrancheVault.sol#L584`, `TrancheVault.sol#L870`). No contract in `packages/perps/src` calls `reconcile()`, and the whole package compiles with it deleted. The call belonged to the earlier synchronous vault, whose `withdraw` and `redeem` began with `POOL.reconcile()` (SC Audit Studio pre-audit, SC-H2). The asynchronous redesign removed the call and kept the hook. Reconciliation now happens only in `_settleLpEpoch` and in the owner actions `finalizePoolConfig`, `assignUnassignedAssets` and `initializeSeedPosition`.

The documentation describes a different system. `IHousePool` declares the hook "Callable only by a configured vault" (`IHousePool.sol#L487-L492`), and the caller matrix of the security notes lists it as an "Internal integration hook" for "Either configured tranche vault" (`SECURITY.md#L318`). The notes also state that during a settlement hold "Junior deposit and redemption requests, reconciliation, and funded claims remain live" (`SECURITY.md#L1154-L1156`, repeated at `SECURITY.md#L496-L499`). During a hold settlement reverts, and the hook refuses every production account.

## Impact Details

No funds are at risk, since vault pricing reads the pool's projections and the next settlement books what the stored state lacks. The harm is that the documentation describes behaviour the code does not have. An operator running a settlement hold expects reconciliation to continue, yet the stored principals and the coupon checkpoint keep their last settled values and no account can refresh them. In the PoC, after a hold of 86,402 seconds the stored senior principal is still 6,000.018267 USDG against a projection of 6,001.333369 USDG. The caller matrix also endorses a coupon checkpoint without queue progress, which `settleLpEpoch` reverts with `HousePool__NoLpEpochProgress` precisely to prevent. The hook takes 100 bytes of the live `HousePool`, whose 24,571 bytes of runtime code sit 5 bytes below the EIP-170 limit.

## Recommendation

Delete `reconcile()`, its `onlyVault` modifier and its `IHousePool` entry, keep the test seam in a test-only subclass as the repository already does for the Router with `LegacyOrderRouterHarness`, and drop the hook from the caller matrix and the two settlement-hold statements. Without the hook `HousePool` compiles to 24,471 bytes, and the repository's snapshot-parity, coupon-checkpoint and waterfall tests pass against the test subclass.

## Link to Proof of Concept

https://gist.github.com/KirillZiborov/1ad5e8e5a8c2c8bc147b072bda6d0a47

## Proof of Concept

## Proof of Concept

This Insight is valuable to reviewers, integrators and operators who rely on the NatSpec, the interface or the caller matrix to decide who can change the tranche waterfall, above all an operator running a settlement hold, and to every future change to `HousePool`, which has 5 bytes of size headroom. It holds on every deployment of the audited code, because the vault addresses are set once and the vault code never calls the hook.

`testNoReconcileDuringSettlementHold` runs in the repository's perps test suite (`BasePerpTest`) through the public vault functions and the Router's `settleLpEpoch`, called by a keeper.

1. A Senior LP deposits 5,000 USDG, the pauser puts settlement on hold, and 86,402 seconds pass.
2. Settlement reverts with `HousePool__LpEpochSettlementPaused`. `reconcile()` reverts with `HousePool__NotAVault` for the owner, the pauser and a keeper, and the owner's `assignUnassignedAssets` reverts with `HousePool__NoUnassignedAssets`.
3. The coupon checkpoint and the stored senior principal of 6,000.018267 USDG have not moved, while `getPendingTrancheState` projects 6,001.333369 USDG.
4. A call from the Senior vault's address, impersonated, passes the guard and books exactly the projection.

Save the code as `packages/perps/test/perps/PoC.t.sol` and run `forge test --root packages/perps --match-path test/perps/PoC.t.sol -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract NoReconcileDuringSettlementHold is BasePerpTest {
    address internal constant SENIOR_LP = address(0x5555);
    address internal constant PAUSER = address(0xFA05E);
    address internal constant KEEPER = address(0xBEEF);

    function setUp() public override {
        super.setUp();
        pool.setPauser(PAUSER);
        vm.label(SENIOR_LP, "senior-lp");
        vm.label(PAUSER, "pauser");
        vm.label(KEEPER, "keeper");
    }

    function testNoReconcileDuringSettlementHold() public {
        // A Senior LP deposits 5,000 USDG, and a keeper settles the epoch.
        settlementToken.mint(SENIOR_LP, 5_000e6);
        vm.startPrank(SENIOR_LP);
        settlementToken.approve(address(seniorVault), 5_000e6);
        uint256 requestId = seniorVault.requestDeposit(5_000e6, SENIOR_LP, SENIOR_LP);
        vm.stopPrank();
        vm.warp(seniorVault.depositEpochStart(requestId));
        bytes[] memory settleUpdate = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(settleUpdate);

        vm.prank(PAUSER);
        pool.pauseLpEpochSettlement();
        uint256 couponCheckpoint = pool.lastSeniorCouponCheckpointTime();
        uint256 storedSenior = pool.seniorPrincipal();

        vm.warp(block.timestamp + 1 days);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__LpEpochSettlementPaused.selector);
        router.settleLpEpoch(update);

        vm.expectRevert(IHousePool.HousePool__NotAVault.selector);
        pool.reconcile();
        vm.prank(PAUSER);
        vm.expectRevert(IHousePool.HousePool__NotAVault.selector);
        pool.reconcile();
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__NotAVault.selector);
        pool.reconcile();

        // The owner's assignUnassignedAssets reconciles first but reverts as a whole when nothing is unassigned.
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        vm.expectRevert(IHousePool.HousePool__NoUnassignedAssets.selector);
        pool.assignUnassignedAssets(false, address(this));

        (uint256 projectedSenior,,,) = pool.getPendingTrancheState();
        assertEq(pool.lastSeniorCouponCheckpointTime(), couponCheckpoint, "coupon checkpoint frozen");
        assertEq(pool.seniorPrincipal(), storedSenior, "stored senior principal frozen");
        assertGt(projectedSenior, storedSenior, "projection has moved on");
        emit log_named_uint("hold length (s)", block.timestamp - couponCheckpoint);
        emit log_named_decimal_uint("stored senior principal (USDG)", storedSenior, 6);
        emit log_named_decimal_uint("projected senior principal (USDG)", projectedSenior, 6);

        // Only a configured vault address passes the guard, and it books exactly the projection.
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), projectedSenior, "the hook books the projection");
        assertEq(pool.lastSeniorCouponCheckpointTime(), block.timestamp, "the hook advances the checkpoint");
    }
}
```
