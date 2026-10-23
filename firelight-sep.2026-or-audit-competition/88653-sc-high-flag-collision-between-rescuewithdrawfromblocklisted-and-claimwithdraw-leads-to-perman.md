# 88653 sc high flag collision between rescuewithdrawfromblocklisted and claimwithdraw leads to permanent freezing of user withdrawals

**Submitted on Aug 14th 2026 at 11:09:38 UTC by @georgibuilds for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88653
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`rescueWithdrawFromBlocklisted` records a rescued withdrawal by setting `isWithdrawClaimed[period][from] = true`—the same storage flag `claimWithdraw` uses to mean "this account has already been paid." No function ever resets it, and `removeFromBlocklist` clears only `isBlocklisted`.

In production, this means that once an account has been rescued for a given period, any withdrawal later credited to that account for that same period is permanently unclaimable by anyone: the shares are already burned, the assets are booked to an entry no address can claim, and the one function that touches the flag is itself blocked by it. Recovery requires a proxy upgrade.

### Vulnerability Details

The same storage flag carries two incompatible meanings:

```solidity
// claimWithdraw - "this account was PAID"                      (FirelightVault.sol:806)
isWithdrawClaimed[period][sender] = true;

// rescueWithdrawFromBlocklisted - "this entry was MOVED AWAY"  (FirelightVault.sol:870)
isWithdrawClaimed[periods[i]][from] = true;

// claimWithdraw - the gate that can no longer be passed        (FirelightVault.sol:794)
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

`removeFromBlocklist` (`:560`) clears only `isBlocklisted`. There is no setter for `isWithdrawClaimed` anywhere in the contract.

The flag is also the guard inside the rescue path itself (`:864`):

```solidity
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

So the one admin function that touches this state is blocked by the state it would need to fix.

`_requestWithdraw` credits `withdrawSharesOf[period][receiver]` (`:1017`)—the receiver, not the owner—and always targets `currentPeriod() + 1` (`:1008`). Those two facts define both victim classes and the exposure window.

#### Case 1: The cleared account loses its own re-request

Alice withdraws in period P, credited to P+1. Compliance flags her; RESCUER moves the entry to the treasury, setting `isWithdrawClaimed[P+1][alice] = true`.

The screen was a false positive, so `removeFromBlocklist(alice)` is called. `isBlocklisted` clears, but the claim flag does not. Period P+1 is still accumulating, so her new `redeem` credits `withdrawSharesOf[P+1][alice]` again and the contract accepts it silently.

When P+1 matures, `claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod`, permanently.

#### Case 2: An unrelated third party loses funds

Because the credit lands on the receiver, Bob—never blocklisted and with no connection to the compliance action—calls `redeem(shares, alice, bob)`.

His shares burn and `withdrawSharesOf[P+1][alice]` is credited. Alice cannot claim because of the stale flag; Bob cannot claim because he is not the credited receiver, so `claimWithdraw` reverts `NoWithdrawalAmount`.

Bob's assets are unreachable by anyone, and he had no way to know the condition existed.

#### Fix

Track "moved away" separately from "paid out". `FirelightVaultStorage` has a 48-slot `__gap`, so adding storage is upgrade-safe:

```solidity
mapping(uint256 => mapping(address => bool)) public isWithdrawRescued;

// rescueWithdrawFromBlocklisted - replace :870
isWithdrawRescued[periods[i]][from] = true;

// rescueWithdrawFromBlocklisted - replace the guard at :864
if (isWithdrawRescued[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

`claimWithdraw:794` then checks only `isWithdrawClaimed`, which now means solely "was paid", and a re-credited entry claims normally.

## Impact Details

The affected withdrawal is permanently frozen. The shares are already burned, the assets are booked to an entry no account can claim, and no rescue can re-target it: a second `rescueWithdrawFromBlocklisted` reverts on the same flag at `:864`. Only a proxy upgrade can clear the state.

Magnitude is bounded, and I would rather state that than leave it to be discovered. Only funds credited to the flagged period are affected: requests made during the remainder of period P. The exposure window is up to one full period and scales with the configured `PeriodConfiguration.duration`.

Protocol documentation describes a current \~1-day cadence, but the V2 vault is not yet deployed, so the configured duration should be confirmed rather than assumed. Waiting one period before re-requesting avoids the defect entirely.

{% hint style="info" %}
Explicitly not claimed: remaining stakers are not diluted and cover capacity is not understated. I measured both against a control and they are identical—`totalAssets()` and other holders' redeemable value are unchanged, because `pendingWithdrawAssets += assets` removes the assets from `totalAssets()` at request time (`:427-428`), so whether the entry is later claimable is invisible to everyone else.
{% endhint %}

Weighed against those limits, the strongest element is Case 2: that victim cannot apply the mitigation, because they cannot see the condition that requires it.

## References

* `contracts/core/FirelightVault.sol`
  * `:427-428` (`totalAssets`)
  * `:560` (`removeFromBlocklist`)
  * `:794` and `:806` (`claimWithdraw`)
  * `:864` and `:870` (`rescueWithdrawFromBlocklisted`)
  * `:1008`, `:1017`, `:1025` (`_requestWithdraw`)
* `contracts/core/FirelightVaultStorage.sol`
  * `:87` (48-slot `__gap`)

## Proof of Concept

Step-by-step reproduction for both cases, followed by a runnable test.

Save as `test/poc_rescue_withdraw_brick.test.js` and run:

```bash
npx hardhat test test/poc_rescue_withdraw_brick.test.js
```

Expected output—5 passing:

```
PoC: rescueWithdrawFromBlocklisted permanently bricks a re-credited withdrawal
   ATTACK: rescue -> un-blocklist -> re-request -> claimWithdraw reverts forever
   ATTACK: there is no recovery path, a second rescue also reverts on the same flag
   ATTACK: a THIRD PARTY who names the flagged address as receiver also loses funds
   BOUND: waiting one period out avoids the freeze entirely
   CONTROL: without the rescue, the identical re-request claims normally

5 passing
```

### Step by step: Case 1

{% stepper %}
{% step %}

## Alice and Bob each deposit 10

Period P is current.
{% endstep %}

{% step %}

## Alice calls `redeem(6)`

`_requestWithdraw` credits `withdrawSharesOf[P+1][alice] = 6`, `withdrawAssets[P+1] = 6`, `pendingWithdrawAssets = 6`, and burns her shares.
{% endstep %}

{% step %}

## Blocklist Alice

`BLOCKLIST_ROLE` calls `addToBlocklist(alice)`.
{% endstep %}

{% step %}

## Rescue Alice's withdrawal

`RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, treasury, [P+1])`.

Assert:

```solidity
withdrawSharesOf[P+1][treasury] == 6
withdrawSharesOf[P+1][alice] == 0
isWithdrawClaimed[P+1][alice] == true
```

{% endstep %}

{% step %}

## Remove Alice from the blocklist

`BLOCKLIST_ROLE` calls `removeFromBlocklist(alice)`—the false positive is reversed.

Assert `isBlocklisted(alice) == false`, but `isWithdrawClaimed[P+1][alice]` is still true.
{% endstep %}

{% step %}

## Alice calls `redeem(4)`

Period P+1 is still accumulating, so it credits `withdrawSharesOf[P+1][alice] = 4`. Her share balance is now 0.
{% endstep %}

{% step %}

## Advance past period P+1

{% endstep %}

{% step %}

## Treasury claims the rescued entry

`treasury.claimWithdraw(P+1)` succeeds and emits `CompleteWithdraw`.
{% endstep %}

{% step %}

## Alice attempts to claim

`alice.claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod(P+1)`, permanently.
{% endstep %}
{% endstepper %}

The asymmetry between the treasury claim and Alice's claim is the defect: same period, same vault, the rescued entry pays out while the freshly credited one cannot.

### Step by step: Case 2 (third-party victim)

{% stepper %}
{% step %}

## Leave Alice with the stale claim flag

Perform steps 3–5 above, leaving `isWithdrawClaimed[P+1][alice] == true`.
{% endstep %}

{% step %}

## Bob withdraws to Alice as receiver

Bob, who was never blocklisted, calls `redeem(10, alice, bob)`, naming Alice as receiver.

His shares burn and `withdrawSharesOf[P+1][alice]` is credited.
{% endstep %}

{% step %}

## Advance past period P+1

{% endstep %}

{% step %}

## Alice attempts to claim

`alice.claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod`.
{% endstep %}

{% step %}

## Bob attempts to claim

`bob.claimWithdraw(P+1)` reverts `NoWithdrawalAmount`—he is not the credited receiver.
{% endstep %}

{% step %}

## Bob's assets are unreachable

Bob's assets are unreachable by any address.
{% endstep %}
{% endstepper %}

### No recovery path

Re-blocklisting Alice and attempting a second rescue reverts `AlreadyClaimedPeriod` at `FirelightVault.sol:864`: the only admin function touching this state is blocked by the state it would fix. Demonstrated by test 2.

### Test source

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')

describe('PoC: rescueWithdrawFromBlocklisted permanently bricks a re-credited withdrawal', function () {
  // decimals 0 keeps every number checkable by hand. Share price stays exactly 1
  // throughout, so no rounding is involved in this PoC at all.
  async function setup() {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 100n }),
    )
    const [alice, bob, treasury] = ctx.users

    await ctx.utils.mintAndApprove(10n, alice)
    await ctx.firelight_vault.connect(alice).deposit(10n, alice.address)
    await ctx.utils.mintAndApprove(10n, bob)
    await ctx.firelight_vault.connect(bob).deposit(10n, bob.address)

    const period = await ctx.firelight_vault.currentPeriod()
    const withdrawPeriod = period + 1n

    // Alice files a withdrawal for 6 of her 10 shares. Credited to period P+1.
    await ctx.firelight_vault.connect(alice).redeem(6n, alice.address, alice.address)
    expect(await ctx.firelight_vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(6n)
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(6n)

    return { ...ctx, alice, bob, treasury, withdrawPeriod }
  }

  it('ATTACK: rescue -> un-blocklist -> re-request -> claimWithdraw reverts forever', async () => {
    const { firelight_vault, blocklister, rescuer, alice, treasury, withdrawPeriod, config } =
      await setup()

    // 1. Compliance flags Alice.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)

    // 2. RESCUER moves her pending withdrawal to the treasury.
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [withdrawPeriod])

    expect(await firelight_vault.withdrawSharesOf(withdrawPeriod, treasury.address)).to.equal(6n)
    expect(await firelight_vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(0n)
    // The flag now says Alice "claimed" a period she was never paid for.
    expect(await firelight_vault.isWithdrawClaimed(withdrawPeriod, alice.address)).to.equal(true)

    // 3. The flag was a FALSE POSITIVE. Compliance reverses it.
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.equal(false)
    // removeFromBlocklist does NOT clear the claim flag.
    expect(await firelight_vault.isWithdrawClaimed(withdrawPeriod, alice.address)).to.equal(true)

    // 4. Period P+1 is still accumulating, so Alice's new request lands in the SAME period.
    await firelight_vault.connect(alice).redeem(4n, alice.address, alice.address)
    expect(await firelight_vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(4n)
    expect(await firelight_vault.balanceOf(alice.address)).to.equal(0n)

    // 5. Period P+1 matures.
    await time.increase(config.period_configuration_duration * 2)

    // The treasury's rescued entry claims fine...
    await expect(firelight_vault.connect(treasury).claimWithdraw(withdrawPeriod)).to.emit(
      firelight_vault,
      'CompleteWithdraw',
    )

    // ...but Alice's legitimate, freshly-credited 4 assets are unreachable. Forever.
    await expect(firelight_vault.connect(alice).claimWithdraw(withdrawPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawPeriod)
  })

  it('ATTACK: there is no recovery path, a second rescue also reverts on the same flag', async () => {
    const { firelight_vault, blocklister, rescuer, alice, treasury, withdrawPeriod } = await setup()

    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [withdrawPeriod])
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    await firelight_vault.connect(alice).redeem(4n, alice.address, alice.address)

    // The only admin function that touches this state requires `from` to be blocklisted
    // again, and then reverts on the very flag we need cleared (FirelightVault.sol:864).
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault
        .connect(rescuer)
        .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [withdrawPeriod]),
    )
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawPeriod)
  })

  it('ATTACK: a THIRD PARTY who names the flagged address as receiver also loses funds', async () => {
    const { firelight_vault, blocklister, rescuer, alice, bob, treasury, withdrawPeriod, config } =
      await setup()

    // Alice is flagged, rescued, then cleared — a legitimate user again.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [withdrawPeriod])
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)

    // Bob has never been blocklisted and has no connection to the compliance action.
    // He withdraws and names Alice as the receiver — _requestWithdraw credits
    // withdrawSharesOf[period][RECEIVER], so the entry lands on Alice.
    await firelight_vault.connect(bob).redeem(10n, alice.address, bob.address)
    expect(await firelight_vault.balanceOf(bob.address)).to.equal(0n)
    expect(await firelight_vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.be.gt(0n)

    await time.increase(config.period_configuration_duration * 2)

    // Alice is the credited receiver but carries the stale flag -> cannot claim.
    await expect(firelight_vault.connect(alice).claimWithdraw(withdrawPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawPeriod)

    // Bob is not the credited receiver -> has nothing to claim.
    await expect(firelight_vault.connect(bob).claimWithdraw(withdrawPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'NoWithdrawalAmount')
      .withArgs(withdrawPeriod)
  })

  it('BOUND: waiting one period out avoids the freeze entirely', async () => {
    const { firelight_vault, blocklister, rescuer, alice, treasury, withdrawPeriod, config } =
      await setup()

    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [withdrawPeriod])
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)

    // Alice waits for the next period rather than re-requesting immediately.
    await time.increase(config.period_configuration_duration)
    await firelight_vault.connect(alice).redeem(4n, alice.address, alice.address)
    const laterPeriod = (await firelight_vault.currentPeriod()) + 1n
    expect(laterPeriod).to.be.gt(withdrawPeriod)

    await time.increase(config.period_configuration_duration * 2)

    // Claims normally — the defect is confined to the flagged period.
    await expect(firelight_vault.connect(alice).claimWithdraw(laterPeriod)).to.emit(
      firelight_vault,
      'CompleteWithdraw',
    )
  })

  it('CONTROL: without the rescue, the identical re-request claims normally', async () => {
    const { firelight_vault, alice, withdrawPeriod, config } = await setup()

    // Same second request, no blocklist/rescue in between.
    await firelight_vault.connect(alice).redeem(4n, alice.address, alice.address)
    expect(await firelight_vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(10n)

    await time.increase(config.period_configuration_duration * 2)

    await expect(firelight_vault.connect(alice).claimWithdraw(withdrawPeriod)).to.emit(
      firelight_vault,
      'CompleteWithdraw',
    )
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(0n)
  })
})
```

The CONTROL test proves the identical re-request settles normally when no rescue intervenes and `pendingWithdrawAssets` returns to zero, so the fixture is sound and the rescue is the cause. The BOUND test documents the mitigation rather than hiding it.
