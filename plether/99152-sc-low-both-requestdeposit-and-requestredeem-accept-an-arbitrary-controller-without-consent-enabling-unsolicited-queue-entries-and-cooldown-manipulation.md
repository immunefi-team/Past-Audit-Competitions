# \#99152 [SC-Low] Both requestDeposit and requestRedeem accept an arbitrary controller without consent, enabling unsolicited queue entries and cooldown manipulation

**Submitted on Nov 12th 2026 at 06:27:03 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99152
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

The Cecuro low finding LOW-40 fixed an attack where a third party could force a zero-balance receiver into the LP cooldown by requiring `receiver == controller` on claim, cancel and refund paths, and SECURITY.md documents that rule. The fix did not cover the creation side. Neither `requestDeposit(assets, controller, owner)` nor `requestRedeem(shares, controller, owner)` checks that the named controller consents to being named.

`requestDeposit` (`TrancheVault.sol:569-575`) delegates to `_requestDeposit`, whose preflight (`_requireRequestDepositPreflight`, `:1574-1589`) validates only the owner side: `msg.sender` must be `owner` or an operator of `owner`, and the deposit amount must meet the minimum. The `controller` parameter is never compared with `msg.sender`, with `owner`, or with an operator relation on `controller`. The same is true of `requestRedeem` (`:606-632`), which checks the owner's balance, cooldown and allowance and then enqueues the request under `controller`.

The consequence on the redemption side is concrete and already reproduced: a single-share request naming a victim as controller lands at the head of the victim's controller queue, and the standard ERC-7540 surface reads only `controllerRedeemHead`, so `maxRedeem`, `maxWithdraw`, `redeem` and `withdraw` report zero and revert for the victim's own claimable epoch. Clearing the planted head through `claimRedeemRefund` transfers a share to the receiver and sets `lastDepositTime[receiver] = block.timestamp` (`:942-944`), restarting the victim's one hour cooldown, during which transfers and new redemption requests are blocked. Cost to the attacker: one share, about 1e-9 USDG, plus gas.

The consequence on the deposit side is weaker but real: an attacker can deposit their own USDG naming a victim as controller, which pollutes the victim's deposit queue with an entry the victim did not create. It is economically neutral for the victim, but it breaks the expectation that a controller's queue contains only entries the controller authorized, and it interacts with integrations that track deposit state per controller.

Root cause: the ERC-7540 `controller` parameter is treated as a routing label rather than an authorization principal. The LOW-40 fix addressed delivery (claim, cancel, refund) but left creation (request) unpatched, so the same missing consent check exists on both async vault flows.

Recommendation: on `requestRedeem` require `controller == msg.sender` or `isOperator[controller][msg.sender]`, matching the rule already applied to claim, cancel and refund; apply the same check to `requestDeposit`, or document explicitly that permissionless deposit-queue creation is intended; and let `claimRedeemRefund` skip the cooldown restart when the refunded request was created by a third party.

## Link to Proof of Concept

https://gist.github.com/crypt0b0y/606ad4145472dc7de24d461680783ad2

## Proof of Concept

This is an insight about a missing authorization check, not an exploit, so the reproduction is a direct call sequence. It was verified against the code at commit a82d4fef and, for the redemption consequence, with the runnable test L2_RedeemHeadPoisoning.t.sol in the secret gist linked above.

Step 1, deposit side. From any address that is not the victim and holds USDG, call requestDeposit(assets, VICTIM, attacker) with assets at or above minTrancheDepositAssets. The call succeeds. The preflight validates that msg.sender is the owner (the attacker) and that the amount is large enough; it never checks the controller. The result is a pending deposit entry in the victim's controllerDepositHead queue that the victim did not create and cannot cancel, because cancellation requires being the controller or an operator of the controller.

Step 2, redemption side. From an address holding one share, call requestRedeem(1, VICTIM, dust). The request is accepted because the full-balance minimum-size exemption applies. It becomes the head of the victim's controllerRedeemHead queue.

Step 3, observable effect. With a claimable epoch of its own, the victim calls maxRedeem(VICTIM) and maxWithdraw(VICTIM) and both return zero, because the standard surface reads only the head. redeem(claimable, VICTIM, VICTIM) reverts ERC4626ExceededMaxRedeem and withdraw(1, VICTIM, VICTIM) reverts ERC4626ExceededMaxWithdraw. The explicit-id claimRedeem(requestId, shares, receiver, controller) still pays the victim about 999.999954 USDG, which is why this report is an insight rather than a theft.

Step 4, cooldown. claimRedeemRefund of the planted head sets lastDepositTime(VICTIM) to block.timestamp; maxRequestRedeem(VICTIM) becomes zero and a one-share transfer reverts TrancheVault__TransferDuringCooldown. The third-party request therefore restarted the victim's one hour cooldown.

To run the redemption part: download L2_RedeemHeadPoisoning.t.sol from the gist into packages/perps/test/poc/ and run forge test --root packages/perps --match-path "test/poc/L2_RedeemHeadPoisoning.t.sol" -vv. Result: 2 passed, 0 failed.

Suggested fix verification: after adding the controller consent check on requestRedeem() and requestDeposit(), the step 2 call from a non-controller address should revert, while a controller calling for itself still succeeds. There is no state or value change to assert beyond that, which is the nature of this class of finding.
