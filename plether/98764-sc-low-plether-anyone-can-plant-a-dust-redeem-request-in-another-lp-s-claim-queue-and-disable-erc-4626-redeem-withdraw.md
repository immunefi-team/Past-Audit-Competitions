# \#98764 [SC-Low] Plether: anyone can plant a dust redeem request in another LP's claim queue and disable ERC-4626 redeem/withdraw

**Submitted on Nov 10th 2026 at 02:41:35 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98764
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Anyone can plant a dust redeem request in another LP's claim queue and disable that LP's ERC-4626 `redeem` / `withdraw`

**Severity:** Low
**Impact:** Contract fails to deliver promised returns, but doesn't lose value
**Asset:** `TrancheVault.sol` (both the Senior and the Junior instance)
**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## What is and is not claimed

Claimed: an unprivileged third party, at the cost of gas and one share-wei, makes the documented claim functions `redeem` and `withdraw` revert for a victim LP whose redemption is fully funded, and makes `maxRedeem` and `maxWithdraw` report zero for them. No price move, liquidity shortage, or privileged role is involved.

Not claimed: loss or permanent freezing of the victim's USDG. The funded amount stays in vault escrow and a victim who knows the non-standard functions can still reach it (see "Limits"). This is not a liquidity or exit-delay report: the redemption in the PoC is funded in full, and the control test shows the same flow paying out when no request is planted.

## Summary

`TrancheVault.requestRedeem(shares, controller, owner)` checks that the caller may spend the owner's shares, but never checks that `controller` agreed to receive the request. The request is appended to the controller's per-account redeem queue.

The ERC-4626 claim surface reads only the head of that queue: `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` all act on `controllerRedeemHead[controller]`. A one-share-wei request planted by a stranger becomes the victim's head, carries zero claimable assets, and stays there until the victim removes it. While it is the head, the victim's real funded redemption behind it cannot be claimed through those four functions.

`CANONICAL_ENTRYPOINTS.md` documents that path as the way LPs exit: funded requests are "claimed through ERC-4626 `redeem` or `withdraw`".

## Root cause

All references are to `packages/perps/src/TrancheVault.sol` at the pin.

- `requestRedeem` (`:606-632`): the only authorisation is on the owner side (`:625-627`). `controller` only has to be nonzero. Every other request, claim and cancel route calls `_requireControllerAuth(controller)`; this one does not.
- `_enqueueRedeemRequest` (`:688-708`): appends the epoch to the controller's queue with `_appendControllerRedeem(controller, requestId)`.
- `_requireMinimumRedeemSize` (`:1611-1618`): the minimum request size is skipped when `shares == maxShares`, so an account whose whole balance is one share-wei may request.
- `redeem` (`:1006`), `withdraw` (`:998`), `maxRedeem` (`:1306`), `maxWithdraw` (`:1299`): all use `controllerRedeemHead[controller]` and nothing else.
- `_maybeCloseRedeemPosition` (`:1216-1227`): a position leaves the controller's queue only when the controller itself processes it.
- `claimRedeemRefund` (`:916-947`): the one call that removes a refunded dust head. It returns the planted share-wei to the controller and sets `lastDepositTime[controller] = block.timestamp` (`:943`), which starts a new one-hour cooldown on the controller's whole balance.

On the pool side, an epoch whose shares are worth zero USDG is refunded, not funded (`HousePool.sol:960`), so the planted position never becomes claimable.

## Attack path

The attacker is any LP whose shares are past the one-hour cooldown. No role is used.

1. The attacker transfers one share-wei to a fresh account `P`.
2. `P` calls `requestRedeem(1, victim, P)`. The whole-balance rule skips the minimum size. `controllerRedeemHead[victim]` now points at the planted epoch.
3. The victim later requests a real redemption. It lands in a later epoch and is funded normally.
4. The victim calls `redeem` or `withdraw`, as documented. Both revert, and `maxRedeem` / `maxWithdraw` return 0, although the victim's USDG is funded and held in vault escrow.

The attacker can repeat step 1-2 once per hourly epoch. Each plant is a separate head, and the victim has to clear each one in turn.

## Measurements

Local Foundry tests at the pin, on the repository's own LP settlement harness (`test/perps/spec/lp/AsyncLpEpochSettlement.t.sol`). Deposit sizes are test values; no vault or pool state is altered directly.

| Case | `maxRedeem(victim)` | `maxWithdraw(victim)` | `redeem` / `withdraw` by victim | Funded USDG owed to victim |
|---|---|---|---|---|
| Control, no plant | funded shares | about 5,000 USDG | pays in full | about 5,000 USDG |
| One planted request | 0 | 0 | revert `ERC4626ExceededMaxRedeem` / `ERC4626ExceededMaxWithdraw` | about 5,000 USDG, in escrow |
| Two planted requests, first cleared | 0 | 0 | still blocked | about 5,000 USDG, in escrow |

Attacker cost: gas, plus one share-wei per plant. `estimateRedeemAssets` values two share-wei at 0 USDG atoms, and the attacker's own position stays redeemable.

Cost to the victim of clearing a plant: one `claimRedeemRefund` transaction per planted epoch, and a restarted one-hour cooldown. The PoC shows the victim's next `requestRedeem` reverting with `TrancheVault__DepositCooldown` straight after clearing.

## Impact

- An LP who follows the documented exit path cannot claim a funded redemption. Wallets and integrations that read `maxRedeem` / `maxWithdraw` show nothing to claim.
- A contract that holds LP shares and exposes only the ERC-4626 functions has no way to remove the planted head. Unless it can also call `claimRedeem(requestId, ...)` or `claimRedeemRefund`, or appoint an operator who can, its funded USDG stays in escrow. We did not look for a deployed integration of this kind and do not claim one exists.
- The attacker gains nothing. The effect is a targeted denial of the standard claim surface for chosen LPs, at a cost of gas.

## Limits

- The victim's USDG is not lost. `claimRedeem(requestId, shares, receiver, controller)` takes an explicit request id and is not affected.
- `claimRedeemRefund(dustId, controller, controller)` removes a planted head, at the cost of the cooldown described above.
- A victim with no later redemption request is unaffected until they make one.
- The tests use the repository's mock price feed and settlement helper, not a live keeper.

## Why this is not a known issue

- **Cecuro LOW-39 and LOW-40** dealt with unsolicited dust disabling a controller's path and a third party imposing a cooldown on an unrelated address. They were fixed by requiring `receiver == controller` on share deliveries. That fix covers the delivery side. This report is the request side, which still accepts an arbitrary controller.
- **Cecuro LOW-43** is a documentation note that the `max*` arguments are controllers. It does not describe a third party adding to another controller's queue.
- **Cecuro MED-24 and SECURITY.md "LP claim and allocation limitations"** cover zero-payout epochs consuming pool settlement work and a user's own dust request rounding to zero. Both concern the pool-level queue or the requester's own position, not another controller's claim head.
- **SC Audit Studio SC-H2** let any address force `maxWithdraw` / `maxRedeem` to zero through a zero-amount cooldown reset in the earlier synchronous vault. The symptom is similar; the code path no longer exists and the root cause here is different.
- Not in `packages/perps/SECURITY.md` Known Limitations or the AI scan findings.

## Recommended fix

Require the controller's consent in `requestRedeem`, as the other routes already do:

```solidity
if (controller != owner && msg.sender != controller && !isOperator[controller][msg.sender]) {
    revert TrancheVault__NotControllerOrOperator();
}
```

An alternative that keeps third-party requests is to make the head-based functions skip positions with nothing claimable, so a refunded or zero-value head cannot hide a funded request behind it.




## Link to Proof of Concept

https://gist.github.com/dexloom/85a0a34e2b52c4b7c0b58b5f00263428

## Proof of Concept

## Proof of concept

PoC: https://gist.github.com/dexloom/85a0a34e2b52c4b7c0b58b5f00263428

At the pin, initialise submodules, add the file to `packages/perps/test/perps/`, then:

    git submodule update --init --recursive
    cd packages/perps
    forge test --match-path test/perps/PoisonedRedeemHead.t.sol --match-contract PoisonedRedeemHeadPoC --match-test "test_PoC_|test_Control_" -vv

Expected: 3 tests pass.

- `test_PoC_ThirdPartyPoisonsVictimRedeemHead`: the attack, the reverts, and the cooldown the victim incurs by clearing the head.
- `test_Control_WithoutPlantStandardRedeemPays`: the same flow without the plant pays the victim through `redeem`.
- `test_PoC_RepeatedPlantsStackInVictimQueue`: two plants in consecutive epochs; clearing one leaves the victim still blocked.

The test contract inherits the repository's `AsyncLpEpochSettlementTest` to reuse its helpers. The `--match-test` filter keeps that suite's own tests from running.
