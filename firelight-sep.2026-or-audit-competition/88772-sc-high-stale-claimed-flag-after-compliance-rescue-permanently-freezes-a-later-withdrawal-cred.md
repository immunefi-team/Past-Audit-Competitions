# 88772 sc high stale claimed flag after compliance rescue permanently freezes a later withdrawal credited to the same period

**Submitted on Aug 15th 2026 at 00:37:58 UTC by @Maroutis for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88772
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][account] = true` even though that period can still receive new withdrawal requests. If compliance later unblocks the account, a normal withdrawal burns more shares and credits a new withdrawal to the same period. `claimWithdraw` and a second rescue both revert with `AlreadyClaimedPeriod`, leaving the underlying in the vault but excluded from `totalAssets()`.

## Vulnerability Details

The rescue moves the existing withdrawal shares and then marks the source `(period, account)` pair as claimed:

```solidity
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

Withdrawal requests made during the current period always target `currentPeriod()+1`. `_requestWithdraw` does not check the terminal flag:

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

The fresh request succeeds and irreversibly burns shares. At maturity, `claimWithdraw` checks the stale flag before calculating or transferring the new entitlement:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

The rescue path contains the same check. There is no admin or user function that clears the flag or moves an entitlement behind a true flag.

This requires no malicious privileged actor. Blocklist, rescue, and later unblock are normal compliance lifecycle actions. Once unblocked, the user is an ordinary caller and the vault publicly accepts the fatal withdrawal.

## Impact Details

The later withdrawal amount is permanently frozen. In the PoC, 600 FXRP is left in the vault after the user's 600 live shares are burned. `pendingWithdrawAssets` remains 600, `totalAssets()` excludes it, and `withdrawalsOf(P,user)` still reports 600, yet neither claimant nor rescuer can transfer it. The amount is bounded only by the user's remaining share balance, so this is the in-scope Critical impact “Permanent freezing of funds.”

## References

* [`rescueWithdrawFromBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845-L875)
* [`claimWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L810)
* [`_requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1027)

## Proof of Concept

From the repository root:

```bash
npx hardhat test poc/submissions/01_compliance_rescue_freeze.js
```

The test performs these steps:

{% stepper %}
{% step %}

## Alice deposits and requests a withdrawal

Alice deposits 1,000 FXRP and requests a 400-FXRP withdrawal into period `P`, retaining 600 shares.
{% endstep %}

{% step %}

## Alice is blocklisted

The blocklister blocklists Alice.
{% endstep %}

{% step %}

## The pending withdrawal is rescued

The rescuer moves Alice's pending 400 to Bob. The test confirms `isWithdrawClaimed(P,Alice) == true`.
{% endstep %}

{% step %}

## Alice is removed from the blocklist

The blocklister removes Alice from the blocklist.
{% endstep %}

{% step %}

## Alice makes a new withdrawal

Alice calls the public `withdraw(600, Alice, Alice)`. It succeeds, burns all 600 shares, increases `pendingWithdrawAssets`, and credits `withdrawSharesOf(P,Alice)`.
{% endstep %}

{% step %}

## Bob claims the rescued withdrawal

After two period transitions, Bob claims the rescued 400 normally.
{% endstep %}

{% step %}

## Alice cannot claim her new withdrawal

Alice's `claimWithdraw(P)` reverts `AlreadyClaimedPeriod(P)`.
{% endstep %}

{% step %}

## A second rescue also fails

Re-blocklisting Alice and attempting another rescue also reverts `AlreadyClaimedPeriod(P)`.
{% endstep %}

{% step %}

## Frozen funds are confirmed

The final assertions show 600 assets still held, `pendingWithdrawAssets == 600`, `totalAssets() == 0`, and `withdrawalsOf(P,Alice) == 600`.
{% endstep %}
{% endstepper %}

#### Full runnable test code

The exact standalone test executed for this submission is reproduced in full below.

```javascript
/*
 * Submission C-1: compliance rescue terminalises a still-open withdrawal key.
 *
 * Run from the repository root:
 *   npx hardhat test poc/submissions/01_compliance_rescue_freeze.js
 */
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../../test/setup/fixtures.js')
const { expect } = require('chai')

const M = (n) => BigInt(n) * 1_000_000n

async function vaultCtx(cfg = {}) {
  return loadFixture(
    deployVault.bind(null, Object.assign({ decimals: 6, initial_deposit_limit: M(10_000_000).toString() }, cfg))
  )
}

describe('Submission C-1  Compliance rescue permanently freezes a later withdrawal', () => {
  it('a later, ordinary withdrawal into the same period can never be claimed or rescued', async () => {
    const ctx = await vaultCtx()
    const { firelight_vault: vault, token_contract, blocklister, rescuer, users, config } = ctx
    const alice = users[0]
    const bob = users[1]

    // Alice deposits 1,000 and requests a partial withdrawal of 400.
    await ctx.utils.mintAndApprove(M(1000), alice)
    await vault.connect(alice).deposit(M(1000), alice.address)

    const P = (await vault.currentPeriod()) + 1n
    await vault.connect(alice).withdraw(M(400), alice.address, alice.address)

    expect(await vault.withdrawSharesOf(P, alice.address)).to.equal(M(400))
    expect(await vault.balanceOf(alice.address)).to.equal(M(600))

    // Normal compliance lifecycle: blocklist, rescue, then unblock.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [P])

    expect(await vault.withdrawSharesOf(P, alice.address)).to.equal(0n)
    expect(await vault.withdrawSharesOf(P, bob.address)).to.equal(M(400))
    expect(await vault.isWithdrawClaimed(P, alice.address)).to.equal(true)

    await vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await vault.isBlocklisted(alice.address)).to.equal(false)

    // Alice is ordinary again. The public path accepts a fresh withdrawal into P,
    // burns every remaining share, and leaves the terminal flag untouched.
    const pendingBefore = await vault.pendingWithdrawAssets()
    await vault.connect(alice).withdraw(M(600), alice.address, alice.address)

    expect(await vault.balanceOf(alice.address)).to.equal(0n)
    expect(await vault.withdrawSharesOf(P, alice.address)).to.equal(M(600))
    expect(await vault.pendingWithdrawAssets()).to.equal(pendingBefore + M(600))
    expect(await vault.isWithdrawClaimed(P, alice.address)).to.equal(true)

    await time.increase(config.period_configuration_duration * 2)
    expect(await vault.currentPeriod()).to.be.greaterThan(P)

    // Control: Bob can claim the entitlement moved by the rescue.
    await vault.connect(bob).claimWithdraw(P)
    expect(await token_contract.balanceOf(bob.address)).to.equal(M(400))

    // Alice's new entitlement is behind an irreversible terminal flag.
    await expect(vault.connect(alice).claimWithdraw(P))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(P)

    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [P]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(P)

    const frozenAssets = await vault.withdrawalsOf(P, alice.address)
    const pendingAssets = await vault.pendingWithdrawAssets()
    console.log(
      `    witness: burnedShares=${M(600)}, frozenAssets=${frozenAssets}, pendingAssets=${pendingAssets}, claimedFlag=true`
    )
    expect(pendingAssets).to.equal(M(600))
    expect(await token_contract.balanceOf(vault.target)).to.equal(M(600))
    expect(await vault.totalAssets()).to.equal(0n)
    expect(frozenAssets).to.equal(M(600))
  })
})
```

#### Actual execution result

```
  Submission C-1  Compliance rescue permanently freezes a later withdrawal
    witness: burnedShares=600000000, frozenAssets=600000000, pendingAssets=600000000, claimedFlag=true
    ✔ a later, ordinary withdrawal into the same period can never be claimed or rescued (3063ms)


  1 passing (3s)
```
