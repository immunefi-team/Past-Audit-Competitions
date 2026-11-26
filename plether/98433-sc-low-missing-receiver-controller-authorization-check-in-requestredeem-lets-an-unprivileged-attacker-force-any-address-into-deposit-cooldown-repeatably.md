# \#98433 [SC-Low] Missing receiver/controller-authorization check in requestRedeem lets an unprivileged attacker force any address into deposit cooldown, repeatably

**Submitted on Nov 8th 2026 at 01:57:33 UTC by @koshak01 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98433
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
TrancheVault.requestRedeem is missing an authorization check on the controller parameter that the same contract already enforces on its sibling entry points (requestRedeemFromClaimableDeposit via _requireControllerAuth, and the claim-side functions cancelRedeemRequest/claimRedeemRefund via both _requireControllerAuth and _requireShareReceiver). This lets an unprivileged attacker name an unrelated victim as controller on a dust redeem request they fund with their own trivial share balance, with zero consent from the victim.

## Vulnerability Details
requestRedeem(uint256 shares, address controller, address owner) (TrancheVault.sol, lines 606-630) checks zero-address, withdrawal size, the caller's own deposit cooldown, max-redeem bound, and spends allowance from owner -- but never calls any authorization check on controller. Its sibling requestRedeemFromClaimableDeposit (line 645 onward) calls _requireControllerAuth(controller) immediately. The four claim-side functions (cancelRedeemRequest, claimRedeemRefund, and two more) call both _requireControllerAuth and _requireShareReceiver. requestRedeem is the sole entry point in this family that assigns an attacker-chosen controller with no check at all.

An attacker funds a dust position with their own money (full remaining balance, satisfying the minimum-redeem-size check), then calls requestRedeem(dustShares, victim, attacker) naming an arbitrary victim as controller. This primes the victim's controller-redeem queue head with the attacker's dust request -- the victim never called requestRedeem themselves. When the request resolves refund-only (too small to fund) and the victim later clears their stuck head via claimRedeemRefund (the only available path), that call unconditionally resets lastDepositTime[victim] -- a fresh, unsolicited deposit cooldown, locking the victim's entire share balance under TransferDuringCooldown for the cooldown window (DEPOSIT_COOLDOWN = 1 hour). The queue is a linked list, not a single slot, and there is no repeat-counter or cumulative cap on this path, so an attacker willing to re-dust roughly once per cooldown cycle can keep a victim locked out continuously.

## Impact
An unprivileged attacker can impose a repeatable, unsolicited withdrawal cooldown on any address, with no fund loss to the attacker but real, sustained denial of the victim's liquidity/exit flexibility -- fits the program's Medium "Griefing" category. Submitted conservatively at the single-cycle (1 hour) Griefing/Medium level actually demonstrated by the PoC, not at a higher duration tier that was not itself run.

## Link to Proof of Concept

https://gist.github.com/koshak01/55665803893fe382b08d119a4c25deac

## Proof of Concept

Attacker funds a dust-sized junior position with their own money (full remaining balance after offloading the rest, satisfying the minimum-redeem-size check).
Attacker calls requestRedeem(dustShares, victim, attacker) -- naming a zero-balance, unrelated victim as controller. No check blocks this.

The dust request becomes the head of victim's controller-redeem queue (controllerRedeemHead(victim) == requestId), although victim never called requestRedeem.
At settlement, the dust amount is too small to fund and resolves refund-only.
Victim, needing to clear the stuck head to use redeem()/withdraw() again, calls claimRedeemRefund(requestId, victim, victim) -- the only available path.

claimRedeemRefund unconditionally resets lastDepositTime[victim] to the current block timestamp, even though victim made no deposit and never consented to any redeem action.

Control/disconfirming check: immediately before the claim, victim's transfer succeeds normally (no cooldown pending). Immediately after, the identical transfer reverts with TrancheVault__TransferDuringCooldown -- confirming the lock is caused exactly by the forced cooldown reset, not by any pre-existing state. The lock clears itself after DEPOSIT_COOLDOWN (1 hour), confirming the mechanism and its exact duration.

Test file provided as a secret Gist (raw Foundry test, runs standalone in the repo's existing Foundry suite at the pinned commit, no extra fixtures/secrets needed).
