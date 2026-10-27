# 57399 sc critical erc4626 staking lockbook breaks share fungibility partial transfers can dos withdrawals

**Submitted on Oct 25th 2025 at 20:36:18 UTC by @TECHFUND\_inc for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57399
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Permanent freezing of funds

{% hint style="danger" %}
Critical: transferred shares are not tracked in the per-deposit lockbook. This can permanently freeze balances or force users into penalized emergency flows.
{% endhint %}

## Description

Brief/Intro\
The Staking vault enforces time-locks via per-deposit “Stake” entries that are only created on deposit. Withdraw/redeem consume only unlocked Stake entries. Since share transfers do not create Stake entries for recipients, transferred-in sLONG are invisible to the unlock logic. This breaks ERC4626 share fungibility and causes stuck balances or forces users into penalized emergency flows.

## Vulnerability Details

* Root cause
  * On deposit: `_deposit` pushes `Stake({shares, timestamp})` to `stakes[to]`.
  * On withdraw/redeem: `_withdraw` calls `_consumeUnlockedSharesOrRevert(owner, shares)`, which iterates `stakes[owner]` and must fully cover the requested shares from unlocked entries.
  * On transfer: no Stake entries are created/updated; the receiver’s transferred shares have no corresponding Stake entries, so they can never be “consumed” by the unlock logic.

{% stepper %}
{% step %}

### BUG — transferred-in sLONG are non-redeemable

Scenario:

* Alice holds 100 sLONG and transfers 50 to Bob.
* Bob deposits 100 LONG (mints 100 sLONG). Even after unlocking, Bob’s stakes sum to 100 (his deposit only).
* `redeem(150)` reverts with `MinStakePeriodNotMet`; the extra 50 sLONG (received by transfer) are irredeemable via withdraw/redeem and only convertible via emergency flow with penalty.

Impact:

* Recipients of transferred sLONG can end up with permanent or penalized-only access to those shares.
  {% endstep %}

{% step %}

### BUG — partial transfer can DoS standard withdrawals before unlock

Scenario:

* Alice deposits 100 sLONG (single stake entry).
* Alice transfers 50 away.
* Alice then attempts to withdraw/redeem 50 (the remaining balance). The call reverts with `MinStakePeriodNotMet` because the single Stake entry (100) remains locked and the algorithm cannot source unlocked shares from any other entry.

Impact:

* Legitimate positions become partially inaccessible via standard withdraw/redeem paths until the original stake entry unlocks.
* Demonstrates brittle coupling between balance and per-deposit stake entries.
  {% endstep %}
  {% endstepper %}

## Recommendations

* Ensure transferred shares are represented in the lockbook (e.g., create/merge Stake entries on incoming transfers), or track a single per-user lock bucket that follows the shares on transfer.
* Alternatively, disallow transferring locked shares (enforce transfer restrictions) to prevent creation of irredeemable balances.

## Impact Details

* Funds stuck for recipients of sLONG; only recoverable via `emergencyWithdraw`/`emergencyRedeem` with penalty.
* Legitimate positions can become partially inaccessible via standard withdraw/redeem.

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L258-L290>

## Proof of Concept

Include the below code in `staking.test.ts` file:

{% code title="staking.test.ts" %}

```ts
describe('Bug reproductions', () => {
  it('BUG-1: transferred sLONG received cannot be withdrawn/redeemed by receiver', async () => {
    const { staking, long, admin, user1: alice, user2: bob } = await loadFixture(fixture);

    const oneHundred = ethers.utils.parseEther('100');
    const fifty = ethers.utils.parseEther('50');

    // Alice deposits 100 LONG and receives 100 sLONG
    await long.connect(admin).transfer(alice.address, oneHundred);
    await long.connect(alice).approve(staking.address, oneHundred);
    await staking.connect(alice).deposit(oneHundred, alice.address);

    // Alice transfers 50 sLONG to Bob
    await staking.connect(alice).transfer(bob.address, fifty);

    // Bob deposits 100 LONG and receives 100 sLONG
    await long.connect(admin).transfer(bob.address, oneHundred);
    await long.connect(bob).approve(staking.address, oneHundred);
    await staking.connect(bob).deposit(oneHundred, bob.address);

    // Unlock shares
    await staking.connect(admin).setMinStakePeriod(1);

    // Bob tries to redeem 150 sLONG (100 deposited + 50 transferred) -> reverts; only deposited shares are withdrawable
    await expect(
      staking.connect(bob).redeem(oneHundred.add(fifty), bob.address, bob.address),
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

    // Bob can redeem his own deposited 100 sLONG
    const expectedAssets = await staking.previewRedeem(oneHundred);
    const tx = await staking.connect(bob).redeem(oneHundred, bob.address, bob.address);
    await expect(tx)
      .to.emit(staking, 'Withdraw')
      .withArgs(bob.address, bob.address, bob.address, expectedAssets, oneHundred);

    // Remaining transferred 50 sLONG at Bob are now stuck (cannot be redeemed/withdrawn)
    expect(await staking.balanceOf(bob.address)).to.eq(fifty);
    await expect(staking.connect(bob).redeem(fifty, bob.address, bob.address)).to.be.revertedWithCustomError(
      staking,
      'MinStakePeriodNotMet',
    );
  });

  it('BUG-2: after transferring out some sLONG, withdrawing the remaining 50 via withdraw/redeem reverts', async () => {
    const { staking, long, admin, user1: alice, user2: bob } = await loadFixture(fixture);

    const oneHundred = ethers.utils.parseEther('100');
    const fifty = ethers.utils.parseEther('50');

    // Alice deposits 100 LONG and receives 100 sLONG
    await long.connect(admin).transfer(alice.address, oneHundred);
    await long.connect(alice).approve(staking.address, oneHundred);
    await staking.connect(alice).deposit(oneHundred, alice.address);

    // Alice transfers 50 sLONG to Bob
    await staking.connect(alice).transfer(bob.address, fifty);

    // Alice attempts to withdraw her remaining 50 LONG via withdraw/redeem -> reverts due to stake tracking
    await expect(
      staking.connect(alice).withdraw(fifty, alice.address, alice.address),
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
    await expect(
      staking.connect(alice).redeem(fifty, alice.address, alice.address),
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
  });
});
```

{% endcode %}

Observed test output:

```
      ✔ BUG-1: transferred sLONG received cannot be withdrawn/redeemed by receiver (762ms)
      ✔ BUG-2: after transferring out some sLONG, withdrawing the remaining 50 via withdraw/redeem reverts

  2 passing (785ms)
```

***
