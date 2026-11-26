# \#99333 [SC-Low] Permissionless controller gifts make TrancheVault's head-based claim functions revert for the controller's own settled request

**Submitted on Nov 13th 2026 at 08:06:35 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99333
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

`TrancheVault` lets an owner create a deposit or redemption request naming an arbitrary third party as `controller`, without that controller's consent. Requests are inserted into a FIFO keyed only by controller, while the standard `deposit`, `mint`, `withdraw`, `redeem`, and `max*` claim interfaces read only the FIFO head. An attacker can therefore prepend a tiny, fully funded gift request to a victim's queue. The victim's own later request can be settled and fully claimable, yet the standard claim with the victim's requested amount reverts because the head contains only the attacker's tiny request.

No value is stolen and the victim receives the attacker's assets if it claims the head. The selected Low impact is therefore exact: **Contract fails to deliver promised returns, but doesn't lose value.**

### Vulnerability details

The report targets commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/TrancheVault.sol`.

1. `requestDeposit(assets, controller, owner)` and `requestRedeem(shares, controller, owner)` authorize the `owner` side but accept a caller-selected `controller`. The controller does not consent.
2. The request is linked into `controllerDepositHead[controller]` or `controllerRedeemHead[controller]`.
3. The standard claim functions use only that head:
   - deposit side: `deposit`, `mint`, `maxDeposit`, and `maxMint`;
   - redemption side: `withdraw`, `redeem`, `maxWithdraw`, and `maxRedeem`.
4. If the victim's own claim amount is greater than the tiny foreign head, the standard call reverts with `ERC4626ExceededMaxDeposit` or `ERC4626ExceededMaxRedeem`, even though the victim's own request is already funded and fully claimable by its explicit request ID.

Concrete deposit trace:

- Attacker requests `1e6` assets (1 USDG) with `controller = VICTIM`, `owner = ATTACKER`.
- Victim later requests `50_000e6` assets.
- Both requests settle through the normal pool settlement path.
- `claimableDepositRequest(victimId, VICTIM) == 50_000e6`.
- `maxDeposit(VICTIM) == 1e6`, because it reports only the attacker's FIFO head.
- `deposit(50_000e6, VICTIM, VICTIM)` reverts with `ERC4626ExceededMaxDeposit(VICTIM, 50_000e6, 1e6)`.

The redemption path is symmetric: a funded attacker redemption becomes the victim controller's head; `maxRedeem(VICTIM)` reports the attacker's shares; and `redeem(victimOwnShares, VICTIM, VICTIM)` reverts.

### Impact

The attacker needs no victim authorization and spends only its own tiny request. It can repeat the grief in later request epochs. A wallet or integration following the standard head-based ERC-4626/7540 claim path cannot claim the controller's own settled amount in one transaction and will revert.

Observed same-snapshot treatment/control:

| Path | Gift arm | No-gift control |
| --- | --- | --- |
| Deposit | `maxDeposit(VICTIM) = 1e6`; `deposit(50_000e6, ...)` reverts; clearing the gift and then claiming the victim request takes two successful claims after the failed call | `maxDeposit(VICTIM) = 50_000e6`; one claim succeeds |
| Redeem | `maxRedeem(VICTIM)` reports only the attacker's funded shares; `redeem(victimOwnShares, ...)` reverts; clearing the gift and then claiming takes two successful claims after the failed call | one claim succeeds |

Limits are explicit:

- No funds are lost; the victim gains the attacker's gift.
- `claimDeposit(requestId, ...)` and `claimRedeem(requestId, ...)` let a knowledgeable victim claim its own funded request in the same block without clearing the foreign head.
- Therefore this report does **not** claim a time-based freeze, Medium griefing, or theft. It claims only the selected Low promised-return failure of the standard claim interface.

### Recommended fix

Either:

1. require controller consent when `controller != owner` (`msg.sender == controller` or an authorized controller operator); or
2. make head-based claim functions consume enough funded FIFO entries to satisfy the requested amount, and make `max*` report aggregate claimable assets/shares for the controller.

Requiring consent prevents strangers from injecting requests into another controller's queue without changing self-request behavior.

## Proof of Concept

## Proof of Concept

### Environment

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Forge: `1.5.1-v1.5.1`
- Solidity: `0.8.35`
- Test: `test/perps/v1x/V1xGift.t.sol`
- Fixture uses release-style pool, router, and oracle parameters and settles requests through the normal `HousePool` path. No production storage writes or mocked claims are used.

### Reproduction steps

1. Seed `VICTIM` as a Junior LP and wait past its existing deposit cooldown.
2. From `ATTACKER`, mint/approve 1 USDG and call:

```solidity
juniorVault.requestDeposit(1e6, VICTIM, ATTACKER);
```

3. In a later request epoch, have `VICTIM` request 50,000 USDG and settle both requests through the normal LP settlement path.
4. Confirm the victim's explicit request is fully claimable but the controller head reports only 1 USDG:

```solidity
assertEq(juniorVault.claimableDepositRequest(victimId, VICTIM), 50_000e6);
assertEq(juniorVault.maxDeposit(VICTIM), 1e6);
```

5. As `VICTIM`, call:

```solidity
juniorVault.deposit(50_000e6, VICTIM, VICTIM);
```

It reverts with `ERC4626ExceededMaxDeposit(VICTIM, 50_000e6, 1e6)`. In a control restored from the same snapshot but without the attacker's request, the same call succeeds.

6. Repeat on the redemption side: seed attacker shares, have the attacker call `requestRedeem(attackerShares, VICTIM, ATTACKER)`, then let the victim create and settle its own later redemption. `redeem(victimOwnShares, VICTIM, VICTIM)` reverts in the gift arm and succeeds in the same-snapshot no-gift control.

### Exact test command

```bash
FOUNDRY_PROFILE=quick \
FOUNDRY_OUT=out-gift-submit-151 \
FOUNDRY_CACHE_PATH=cache-gift-submit-151 \
E:/bug-bounty-work/toolchains/foundry-v1.5.1/forge.exe test \
  --offline --no-auto-detect -j 1 \
  --match-path 'test/perps/v1x/V1xGift.t.sol' -vv
```

Observed on the exact pin:

```text
Ran 3 tests for test/perps/v1x/V1xGift.t.sol:V1xGiftTest
[PASS] test_G9_Cooldown_GiftVsControl()
[PASS] test_G9_DepositHead_GiftVsControl()
[PASS] test_G9_RedeemHead_GiftVsControl()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The focused workaround control was also run under Forge 1.5.1:

```text
Ran 2 tests for test/perps/v1x/V1xGiftRecheck.t.sol:V1xGiftRecheckTest
[PASS] test_DepositGiftHead_DoesNotDelayVictimByIdClaim()
[PASS] test_RedeemGiftHead_DoesNotDelayVictimByIdClaim()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

That control proves the exact impact boundary stated above: the standard head-based claim fails, while explicit by-ID recovery remains immediate.
