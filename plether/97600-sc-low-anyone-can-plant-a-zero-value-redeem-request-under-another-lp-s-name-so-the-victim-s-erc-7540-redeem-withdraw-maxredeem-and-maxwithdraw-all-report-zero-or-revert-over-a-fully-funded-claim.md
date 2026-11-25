# \#97600 [SC-Low] Anyone can plant a zero-value redeem request under another LP's name, so the victim's ERC-7540 redeem, withdraw, maxRedeem and maxWithdraw all report zero or revert over a fully funded claim

**Submitted on Nov 2nd 2026 at 14:11:21 UTC by @Griffinf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97600
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Severity claimed: Low (Medium if the program weighs integrator freezes; stated plainly below).

`TrancheVault.requestRedeem(shares, controller, owner)` lets the share owner name any `controller`. The request is appended to that controller's FIFO redeem list. The ERC-7540 claim functions `redeem` and `withdraw`, and the views `maxRedeem` and `maxWithdraw`, read only the controller's head position. A one-share request (the full-balance exemption from the minimum redeem size allows it) is worth zero assets, so the pool refunds that epoch; a refunded position stays at the head until the controller explicitly calls `claimRedeemRefund`, and that call restarts the controller's deposit cooldown. The victim's own later redemption is funded behind it, but every standard claim call sees the planted head and returns zero or reverts.

An EOA can still exit through the non-standard `claimRedeem(requestId, …)` or by clearing the planted head with `claimRedeemRefund`, which costs them a one-hour cooldown restart. A contract integrator that claims through the ERC-7540 surface alone has its funds stuck until it is upgraded or its operator intervenes. No funds are stolen. That is why I claim Low, and why Medium is arguable for the integrator case.

The expected pushback, answered: ERC-7540 does allow a separate `controller`. It also expects implementations to decide who may create requests for a controller; the protocol already enforces `receiver == controller` on share delivery (`_requireShareReceiver`) precisely to stop third parties from pushing state onto someone else, and it does not apply the same rule to request creation. One planting account per victim is all it takes; up to about 999 victims can share one planted epoch before the planted shares add up to a nonzero asset value.

Asset: `packages/perps/src/TrancheVault.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Relationship to published known issues

- `SECURITY.md` LP accounting limitations (program known-issues entry "SECURITY.md: Known Limitations") state that "redemption cancellation or terminal refund still begins a new cooldown when shares return to a wallet". The cooldown restart on clearing the planted head is therefore documented behaviour and is NOT claimed as impact here; it is noted only as the cost of the victim's workaround. The finding is that a third party can create the request in the victim's name at all, and that the ERC-7540 claim surface then reports zero and reverts over a funded claim.
- Program known-issues entry "Cecuro MED-24 and MED-29: LP refund epochs and redemption rounding" covers the victim's own small or zero-payout requests being refunded or delayed. It does not cover a third party creating a request in the victim's name, nor the head-only ERC-7540 claim surface hiding a funded claim behind a planted refund.
- The LP claim known issues (Cecuro LOW-3/4/5, MED-24, MED-29; `SECURITY.md` "LP claim/allocation") concern the victim's own small or partial claims reverting at rounding boundaries and zero-payout epochs delaying later epochs. This report is a third party creating the zero-value request in the victim's name and the head-only claim surface hiding a full-size funded claim behind it.
- SC Audit Studio H2 ("zero-amount withdraw/redeem resets a victim's cooldown", fixed in #21) and the historical "third-party deposits resetting cooldowns" test class show the team treats third-party cooldown effects as bugs. The reset here is on `claimRedeemRefund` (`TrancheVault.sol:943`), reachable only because of the planted request.

## Root cause

`requestRedeem` validates owner, cooldown and size, but never relates `controller` to `msg.sender` (`TrancheVault.sol:606-631`):

```solidity
function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId) {
    ...
    if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) revert TrancheVault__DepositCooldown();
    uint256 maxShares = maxRequestRedeem(owner);
    ...
    _requireMinimumRedeemSize(shares, maxShares);           // full-balance exemption: 1 share passes when it is the owner's whole balance
    if (msg.sender != owner && !isOperator[owner][msg.sender]) _spendAllowance(owner, msg.sender, shares);
    _transfer(owner, address(this), shares);
    requestId = _enqueueRedeemRequest(shares, controller);  // appended to the *controller's* list
}
```

The standard claim and view surface reads only the head (`TrancheVault.sol:998-1012`, `:1299-1311`):

```solidity
function withdraw(uint256 assets, address receiver, address controller) public override returns (uint256 shares) {
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}
function maxWithdraw(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemAssets(requestId, controller);
}
```

A refunded head is removed only after the controller claims the refund (`TrancheVault.sol:1216-1226`), and claiming the refund restarts the controller's cooldown (`TrancheVault.sol:943`).

## Vulnerability details

Paths relative to the repository root at `a82d4fef`.

**Planting.** The attacker moves one junior share (about 0.001 of a USDG atom at the PoC's NAV) to a fresh account whose cooldown has elapsed and calls `requestRedeem(1, victim, planterAccount)`. The request is enqueued in the victim's list and becomes their head if the list was empty.

**Refund.** At settlement, `HousePool._fundRedemptionPhase` (`HousePool.sol:960-986`) values the one-share epoch at zero assets and marks it refunded (`refundEnabled`). The position stays at the victim's head.

**Victim's real redemption.** The victim later requests their full redemption; it is enqueued behind the planted request and funded at the next settlement. `claimableRedeemAssets(ownId, victim)` is the full amount, but `maxWithdraw(victim)` and `maxRedeem(victim)` read the planted head and return 0, and `withdraw`/`redeem` revert.

**Clearing.** The victim must call `claimRedeemRefund(plantedId, …)`, which sets `lastDepositTime[victim] = block.timestamp` and restarts the one-hour cooldown, or must know to call the non-standard `claimRedeem(ownId, …)`.

## Impact

- `maxWithdraw` and `maxRedeem` return 0 over a funded claim of 9,999,999,819 USDG atoms (about 10,000 USDG) in the PoC; `withdraw` and `redeem` revert.
- Any integrator that relies on the ERC-7540 surface alone cannot claim until it is changed.
- The victim's only ways out are the non-standard `claimRedeem(requestId, ...)` or `claimRedeemRefund` on the planted id; the latter restarts their cooldown, which is documented behaviour and not counted as impact.
- Attacker cost: one share and gas per victim. Repeatable every epoch.

## Recommended fix

1. In `requestRedeem` (and `requestDeposit`, which has the same shape), require `controller == msg.sender` or `isOperator[controller][msg.sender]` whenever `controller != owner`.
2. Have `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` skip refunded or zero-entitlement head positions and read the first claimable position instead.
3. Do not restart the controller's cooldown when a refund claim returns zero shares.

## Link to Proof of Concept

https://gist.github.com/aussinfosec/7c24e94980b192bf9f3bb12263231d2b

## Proof of Concept

Gist (secret): https://gist.github.com/aussinfosec/7c24e94980b192bf9f3bb12263231d2b

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc   # copy PoC_D_RedeemHeadGrief.t.sol from the gist here
forge test --root packages/perps --match-path test/perps/poc/PoC_D_RedeemHeadGrief.t.sol -vv
```

Stages, all passing (`test_PlantedRefundedRedeemBlocksVictimStandardClaims`, on `BasePerpTest`):

1. A funder deposits 1 USDG of junior and transfers 1 share to the planter after the cooldown. The victim deposits 10,000 USDG of junior.
2. The planter calls `requestRedeem(1, victim, planter)`. After settlement the planted epoch has `refundEnabled == true` and `controllerRedeemHead(victim) == plantedId`.
3. The victim requests their full redemption; after settlement `claimableRedeemAssets(ownId, victim) > 9,990 USDG`.
4. Assert `maxWithdraw(victim) == 0`, `maxRedeem(victim) == 0`; `withdraw(funded, victim, victim)` and `redeem(shares, victim, victim)` revert.
5. The victim calls `claimRedeemRefund(plantedId, victim, victim)`; assert `lastDepositTime(victim) == block.timestamp` and only now `maxWithdraw(victim) == funded`.

Observed output:

```text
[PASS] test_PlantedRefundedRedeemBlocksVictimStandardClaims()
  planted request id 474875
  victim funded claim (USDG atoms) 9999999819
  victim maxWithdraw 0
  victim maxRedeem 0
  victim lastDepositTime after clearing 1709553600
```
