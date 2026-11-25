# \#99093 [SC-Low] Third-party requestRedeem plants a dust request at the head of another controller's queue, zeroing the standard ERC-7540 claim surface and forcing a cooldown restart to clear it

**Submitted on Nov 11th 2026 at 20:37:11 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99093
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

`requestRedeem(shares, controller, owner)` accepts any `controller` value without the controller's consent (`TrancheVault.sol:606-632`). The function validates the owner's balance, cooldown, allowance and minimum size, but never checks that `msg.sender` is the controller or an operator of the controller. The minimum size check exempts a full-balance request (`_requireMinimumRedeemSize`, `:1611-1618`), so an address holding a single share can create a redemption request naming any victim as controller.

The planted request lands at the head of the victim's controller queue (`_enqueueRedeemRequest`, `:688-708`). The standard ERC-7540 claim surface reads only `controllerRedeemHead`: `maxWithdraw` and `maxRedeem` (`:1299-1311`) return the claimable amount of the head request only, and `withdraw` and `redeem` (`:998-1012`) execute against that head. A one-share epoch is worth zero at nine-decimal precision, so the pool refunds it, and the refundable dust stays at the victim's head until the victim acknowledges it.

The victim clears it with `claimRedeemRefund`, which transfers the single share to the receiver and sets `lastDepositTime[receiver] = block.timestamp` (`:942-944`), restarting the victim's entire one hour deposit cooldown. During the cooldown the victim cannot transfer shares (`TrancheVault__TransferDuringCooldown`) and cannot create a new redemption request (`maxRequestRedeem` returns zero). This contradicts the vault NatSpec stating that share delivery cannot impose another account's cooldown (`IAsyncTrancheVault.sol` lines 12-16).

Value disclosure, stated explicitly: no USDG is permanently frozen. The explicit-id claim path `claimRedeem(requestId, shares, receiver, controller)` still works and bypasses the head-only standard surface; in the PoC the victim claims about 999.999954 USDG through it. The impact is therefore limited to the standard interface and to the cooldown: the standard ERC-7540 surface reports zero and reverts, breaking wallets and integrations that use only the standard methods, and clearing the planted head restarts a one hour cooldown. The attack is repeatable once per epoch at a cost of one share, about 1e-9 USDG, plus gas.

Known issue separation. The Cecuro low finding LOW-40 fixed the delivery side of a similar problem by requiring `receiver == controller` on claim, cancellation and refund paths, and SECURITY.md documents that rule. The request side was outside that fix: `requestRedeem` still accepts any controller without consent. The NatSpec describing that a refundable head must be reclaimed before standard claims advance, and the documented statement that refunds restart the cooldown, describe the mechanism for a controller's own requests, not for third-party requests planted without consent. The missing consent check on `requestRedeem` is a gap in the LOW-40 fix rather than a documented design choice.

Recommendation: require controller consent on `requestRedeem`, either `msg.sender == controller` or an operator relation, matching the delivery-side rule; or let the standard claim surface skip refundable zero-value heads automatically so a planted dust request cannot block it; and do not restart the cooldown for refunds of requests the controller did not create.

## Link to Proof of Concept

https://gist.github.com/crypt0b0y/606ad4145472dc7de24d461680783ad2

## Proof of Concept

Environment: forge 1.8.1, solc 0.8.35, optimizer 200, via_ir, HEAD a82d4fef. Local deployment only, no fork, no network transactions, in-scope contracts unmodified; only the Pyth feed and the settlement token are mocked.

PoC file: L2_RedeemHeadPoisoning.t.sol, provided in the secret gist linked above. Download it into packages/perps/test/poc/ and run:

forge test --root packages/perps --match-path "test/poc/L2_RedeemHeadPoisoning.t.sol" -vv

Result: 2 passed, 0 failed.

Setup: the victim funds 10,000 USDG into the junior tranche; the attacker funds 1 USDG and the test advances two hours so the attacker can transfer. The attacker transfers one share to a fresh address, which calls requestRedeem(1, VICTIM, DUST), naming the victim as controller without consent. The test asserts that the planted request is the victim's controller queue head. The victim then requests its own redemption in a later epoch, and that epoch is settled and funded.

Step by step, test_attack_DustHeadBlocksStandardClaimsAndForcesCooldown:
1. The victim has claimable shares in its own epoch, yet maxRedeem(VICTIM) returns 0 and maxWithdraw(VICTIM) returns 0, because both read controllerRedeemHead, which is the planted dust request.
2. redeem(claimable, VICTIM, VICTIM) reverts ERC4626ExceededMaxRedeem and withdraw(1, VICTIM, VICTIM) reverts ERC4626ExceededMaxWithdraw.
3. The explicit-id path still pays: claimRedeem(f, claimable, VICTIM, VICTIM) increases the victim USDG balance by about 999.999954, which is the disclosed limitation of this report.
4. After claimRedeemRefund of the planted head, lastDepositTime(VICTIM) equals block.timestamp, maxRequestRedeem(VICTIM) is 0, and a one-share transfer reverts TrancheVault__TransferDuringCooldown. The third-party request restarted the victim's one hour cooldown.

Control, test_control_NoPlantedHead, passes: the same flow without the planted request, where maxRedeem equals the claimable amount and redeem succeeds. This isolates the effect to the planted head rather than to the settlement itself.

Attacker cost: one share, about 1e-9 USDG, plus gas. No USDG is permanently frozen, and the explicit-id claim path is disclosed above.

Not tested: the same construction on the deposit side, multiple planted requests in one epoch, and interaction with the senior tranche.
