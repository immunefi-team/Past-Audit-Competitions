# \#98187 [SC-Insight] A rejected deposit epoch or a refund-only redeem epoch stays at the head of the controller's queue, so the standard ERC-4626/7540 claim surface reports 0 and reverts while a later epoch of the same controller is already claimable

**Submitted on Nov 5th 2026 at 19:43:34 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98187
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Brief/Intro

`TrancheVault` serves the request-id-less ERC-4626/7540 claim surface from the controller's queue head:
- the views `maxDeposit`, `maxMint`, `maxWithdraw` and `maxRedeem`;
- the claim functions `deposit`, `mint`, `withdraw` and `redeem` with no `requestId`.

When HousePool rejects a deposit epoch (zero-share quote), the vault removes it from the global epoch queue. The same happens when HousePool turns a redeem epoch's remainder into a refund (zero-value quote). In both cases the vault does not remove the epoch from the controller's queue, so it stays as `controllerDepositHead` / `controllerRedeemHead`.

The rejected head reads as 0 claimable, so the whole standard surface follows it: the views return 0 and the claims revert. This holds even after a later epoch of the same controller has settled and is claimable. The surface recovers only after the controller calls a vault-specific function: `cancelPendingDeposit(requestId, …)` or `claimRedeemRefund(requestId, …)`.

No funds are lost; the escrow is fully refundable. The impact is on integrations: routers, aggregators, vault-of-vaults and keepers that follow ERC-4626/7540 and poll `max*` before claiming. They treat the controller as having nothing to claim and stall on it. Front ends see the same 0. The fix is local and cheap.

## Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe.

### 1. The standard surface reads only the controller's head

```solidity
// packages/perps/src/TrancheVault.sol:1285-1311
function maxDeposit(address controller) public view override returns (uint256) {
    uint256 requestId = controllerDepositHead[controller];
    return requestId == 0 ? 0 : claimableDepositRequest(requestId, controller);
}
// maxMint (:1292), maxWithdraw (:1299) and maxRedeem (:1306) follow the same pattern:
// they read controllerDepositHead / controllerRedeemHead and report only that epoch.

// :953-1011: deposit / mint / withdraw / redeem without requestId claim the head epoch
return _claimDeposit(controllerDepositHead[controller], assets, receiver, controller);
```

`_claimDeposit` reverts `TrancheVault__DepositEpochNotFinalized` for an epoch that was never finalized (`TrancheVault.sol:1033-1035`). A rejected epoch is never finalized.

### 2. Rejection and refund remove the epoch globally, not from the controller's queue

```solidity
// packages/perps/src/TrancheVault.sol:1460-1465 (finalizeDepositEpochFromPool, zero-share branch)
assets = epoch.assets;
_removeDepositEpoch(epochId);          // global queue only
if (shares == 0) {
    queueState.rejected = true;
    emit DepositEpochRejected(epochId, IS_SENIOR, assets);
    return assets;                     // controllerDepositHead still points to epochId
}

// packages/perps/src/TrancheVault.sol:1505-1523 (refundRedeemEpochRemainder)
epoch.refundEnabled = true;
epoch.refundableShares = shares;
_removeRedeemEpoch(epochId);           // global queue only; controllerRedeemHead unchanged
```

HousePool calls these two paths here:
- **Deposit side, `HousePool.sol:1047-1053`:** the epoch's marginal quote `quoteDepositFromState(...)` rounds to 0 shares, so HousePool calls `finalizeDepositEpochFromPool(epochId, 0)`.
- **Redeem side, `HousePool.sol:956-970`:** `fullHeadAssets == 0`, i.e. the remaining shares are worth 0 under the frozen phase price, for example a wiped Junior principal. HousePool then calls `refundRedeemEpochRemainder`.

The per-controller list is updated only by `_removeControllerDeposit` (`TrancheVault.sol:1758-1772`) and its redeem twin. Only the claim and cancel paths call them: `cancelPendingDeposit` (`:861`) and `claimRedeemRefund`, through `_maybeCloseRedeemPosition` (`:941`).

### 3. The result

After a rejection or refund, if the same controller has a later epoch that settles normally:
- `claimableDepositRequest(laterEpoch, controller)` reports the full amount;
- `maxDeposit(controller)` and `maxMint(controller)` return 0;
- `deposit(assets, controller)` reverts with `DepositEpochNotFinalized`.

The redeem side behaves the same way (`maxRedeem`, `maxWithdraw`, `redeem`, `withdraw`).

This is self-consistent under ERC-4626, since `maxDeposit` equals what `deposit` accepts. But ERC-7540 integrators expect the request-id-less surface to reflect the controller's claimable state, and here it hides a claimable request behind one that can never become claimable.

The rejected epoch's own escrow is visible only through the vault-specific views:
- deposit side: `refundableDepositRequest` (`TrancheVault.sol:720`) reports it, while `pendingDepositRequest` and `claimableDepositRequest` report 0;
- redeem side: `refundableRedeemRequest` (used inside `claimRedeemRefund`, `:916-941`).

### Note on who can trigger it

`requestDeposit(assets, controller, owner)` authorizes only the asset `owner` or its operator (`_requireRequestDepositPreflight`); the `controller` can be any address. So a third party can also queue a request at the head of another controller's queue. Today this matters only in the rare zero-quote states above. If those states become reachable (parameter changes, a very high share price, dust minimums), the head pin becomes something a third party can impose on any controller that has no earlier queued request.

## Recommendation

Do one of the following (the first is the smallest change):
1. **Unlink the rejected epoch from the controller's head.** When an epoch is rejected or turned refund-only, let the `max*` views and the request-id-less claims skip it: walk `nextRequestId` past rejected or refund-only epochs to the first claimable one. The escrow stays refundable via `cancelPendingDeposit` / `claimRedeemRefund` exactly as today.
2. **Unlink it from the controller queue directly.** Keep the refund in a per-controller `refundable` mapping that `cancelPendingDeposit` / `claimRedeemRefund` read, so the head advances right away.

In both cases, document in the vault NatSpec and README that a rejected or refund-only epoch needs an explicit refund call, and emit the controller address in `DepositEpochRejected` / `RedeemEpochRefundable` (or a per-controller event) so off-chain integrators can react.

## When this insight matters (conditions)

All four conditions must hold:
1. A controller has a deposit epoch rejected (zero-share quote: a dust epoch against a high share price) or a redeem epoch made refund-only (zero-value quote: e.g. a wiped Junior tranche).
2. The same controller has a later epoch of the same type that settles normally.
3. A client uses the standard request-id-less ERC-4626/7540 surface: an aggregator, router or vault-of-vaults polling `maxDeposit` / `maxRedeem`, an auto-claim keeper, or a front end showing `max*`.
4. The controller does not know it must call the vault-specific `cancelPendingDeposit` / `claimRedeemRefund` first.

When they hold, the integration sees 0 claimable and its claim reverts, even though the controller has a claimable epoch and full escrow for the rejected one. Nothing is lost: the controller recovers 100% with one call, and can always claim the later epoch by id (`claimDeposit(requestId, …)`, `TrancheVault.sol:968`).

## Proof of Concept

## Proof of Concept

The Foundry test below (`H043HeadPin.t.sol`) runs on the competition repo at a82d4fe and extends the repo's own `AsyncLpEpochSettlementTest`. It drives the vault hook exactly as `HousePool._settleDepositPhase` does in its zero-quote branch (`HousePool.sol:1050-1053`). It checks five things in order:
- the head stays on the rejected epoch;
- the later epoch is claimable by id;
- `maxDeposit` and `maxMint` return 0, and the standard `deposit` reverts;
- the vault-specific refund returns the full escrow and advances the head;
- after that, the standard surface works.

Place it in `packages/perps/test/perps/H043HeadPin.t.sol` and run:

```bash
forge test --root packages/perps --match-path test/perps/H043HeadPin.t.sol --match-test test_H043_RejectedHeadPinsStandardClaim -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {AsyncLpEpochSettlementTest} from "./spec/lp/AsyncLpEpochSettlement.t.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// Rejected deposit epoch pins controllerDepositHead; standard 4626/7540 claim surface.
contract H043HeadPinTest is AsyncLpEpochSettlementTest {
    address internal constant C = address(0xC0FFEE);

    function test_H043_RejectedHeadPinsStandardClaim() public {
        uint256 a1 = 5_000e6;
        uint256 e1 = _requestDeposit(juniorVault, C, a1);
        _warpToEpoch(e1);
        _refreshMark();
        // HousePool._settleDepositPhase zero-quote branch (HousePool.sol:1050-1053)
        vm.prank(address(pool));
        assertEq(juniorVault.finalizeDepositEpochFromPool(e1, 0), a1);

        assertEq(_async(juniorVault).pendingDepositRequest(e1, C), 0, "pending view");
        assertEq(_async(juniorVault).claimableDepositRequest(e1, C), 0, "claimable view");
        assertEq(_async(juniorVault).refundableDepositRequest(e1, C), a1, "refundable view");
        assertEq(juniorVault.controllerDepositHead(C), e1, "head stays on rejected epoch");

        uint256 a2 = 7_000e6;
        uint256 e2 = _requestDeposit(juniorVault, C, a2);
        assertGt(e2, e1);
        _settleAt(e2);
        assertEq(_async(juniorVault).claimableDepositRequest(e2, C), a2, "later epoch claimable");
        assertEq(juniorVault.controllerDepositHead(C), e1, "head still pinned");

        // standard surface: consistent (max=0, call reverts) but blind to e2
        assertEq(juniorVault.maxDeposit(C), 0, "maxDeposit 0 while e2 claimable");
        assertEq(juniorVault.maxMint(C), 0, "maxMint 0 while e2 claimable");
        vm.prank(C);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochNotFinalized.selector);
        juniorVault.deposit(a2, C);

        // recovery: the controller refunds the rejected epoch, the head advances, the standard claim works
        uint256 balBefore = settlementToken.balanceOf(C);
        vm.prank(C);
        assertEq(_async(juniorVault).cancelPendingDeposit(e1, C, C), a1);
        assertEq(settlementToken.balanceOf(C) - balBefore, a1, "full refund");
        assertEq(juniorVault.controllerDepositHead(C), e2, "head advanced");
        assertEq(juniorVault.maxDeposit(C), a2, "maxDeposit restored");
        vm.prank(C);
        uint256 shares = juniorVault.deposit(a2, C);
        assertGt(shares, 0, "standard claim works after refund");
    }
}
```

Result at a82d4fe:

```
[PASS] test_H043_RejectedHeadPinsStandardClaim() (gas: 2419441)
```

The redeem side behaves the same way. `refundRedeemEpochRemainder` leaves `controllerRedeemHead` on the refund-only epoch, so `maxRedeem` / `maxWithdraw` report 0 and `redeem` / `withdraw` revert until `claimRedeemRefund` closes the position (`TrancheVault.sol:916-941`).
