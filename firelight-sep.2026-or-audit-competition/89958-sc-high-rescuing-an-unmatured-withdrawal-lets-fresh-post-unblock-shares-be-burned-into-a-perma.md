# 89958 sc high rescuing an unmatured withdrawal lets fresh post unblock shares be burned into a permanently closed period

**Submitted on Aug 22nd 2026 at 03:41:55 UTC by @Grant4969 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89958
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted()` can rescue a real, non-zero withdrawal assigned to the still-unmatured period `P+1`. The rescue permanently sets `isWithdrawClaimed[P+1][source] = true`.

If the source account is then unblocked during the same withdrawal-request window, the public `withdraw()` and `redeem()` paths can write fresh withdrawal shares back into the already-closed `(P+1, source)` key. `_requestWithdraw()` does not check `isWithdrawClaimed[period][receiver]`; it accepts the request, increases `pendingWithdrawAssets`, and burns the user's shares.

Once the period matures, both `claimWithdraw()` and a second rescue revert with `AlreadyClaimedPeriod`. The newly requested assets therefore have no reachable withdrawal or rescue path and remain permanently frozen in the Vault.

## Why the privileged-role exclusion does not apply

This issue does not require any privileged role to submit false data, approve an invalid claim, exceed a business limit, or act outside its granted authority.

* The rescued item is a real, non-zero `P+1` pending withdrawal created by the public `withdraw()` flow.
* `rescueWithdrawFromBlocklisted()` accepts this request and does not require the target period to have matured.
* `removeFromBlocklist()` is an authorized lifecycle operation with no enforced period restriction, cooldown, or rollover requirement.
* The blocklist, rescue, and unblock transactions all satisfy their existing on-chain validation.

The irreversible loss occurs afterward, when the unblocked account calls the public `_requestWithdraw()` path. That path burns fresh shares into a `(period, receiver)` key that the contract has already permanently closed. Therefore, the vulnerable operation is a public entry point bypassing the existing claimed-key write barrier, not a trusted role maliciously freezing funds.

## Existing invariant and distinct trigger

The requirement that a claimed account-period must not receive additional withdrawal shares is supported by the project's own development history.

Commit `a2def47716389330c0ebc1f2006fca5c588c9469` is titled:

`Fix rescuing withdrawal to a user that already claimed them`

It added an `isWithdrawClaimed` check for the rescue destination and the regression test:

`reverts if rescuer tries to rescue pending withdrawals to a user who has already claimed that period`

This establishes that a claimed `(period, account)` is a closed state that must not receive additional `withdrawSharesOf` entries. However, the fix only protects the rescue write path. `_requestWithdraw()` also increases `withdrawSharesOf[period][receiver]` but omits the same barrier.

Sentora M-02 previously described a similar terminal state, but through a different trigger: changing the period duration could remap timestamps to a previously claimed historical period. That trigger was fixed through append-only period configurations.

This report does not modify the period configuration. Its independent trigger is:

`future-period rescue -> source key closed early -> authorized unblock -> public withdrawal writes into the closed key`

Therefore, this is not a reproduction of the period-duration-remapping issue. It is a separate cross-entry-point omission in the current rescue/unblock state machine.

## Vulnerability Details

A normal withdrawal requested during period `P` is assigned to `P+1`:

```solidity
uint256 period = currentPeriod() + 1;
```

The rescue function can move that pending withdrawal before `P+1` matures and permanently close the source key:

```solidity
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

After the source is unblocked, `_requestWithdraw()` can still select the same `P+1` key:

```solidity
uint256 period = currentPeriod() + 1;

uint256 sharesWithdraw = _convertToSharesTotals(
    assets,
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);

withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;

_update(owner, address(0), shares);
```

There is no check for:

```solidity
isWithdrawClaimed[period][receiver]
```

Consequently, the transaction succeeds and burns real shares even though the destination account-period is already closed.

When the period becomes claimable, `claimWithdraw()` checks the stale terminal flag before reading the newly added withdrawal shares:

```solidity
if (isWithdrawClaimed[period][sender]) {
    revert AlreadyClaimedPeriod(period);
}

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender],
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
```

A second rescue cannot recover the newly added shares because `rescueWithdrawFromBlocklisted()` performs the same source-side `AlreadyClaimedPeriod` check.

## Failure Sequence

1. During period `P`, a user calls `withdraw(amount, user, user)`, creating a real pending withdrawal for `P+1`.
2. `BLOCKLIST_ROLE` adds the account to the blocklist.
3. `RESCUER_ROLE` rescues the non-zero `P+1` pending withdrawal to a non-blocklisted recipient.
4. The Vault sets `isWithdrawClaimed[P+1][user] = true` while the current period is still `P`.
5. `BLOCKLIST_ROLE` removes the account from the blocklist. No rollover or cooldown is enforced.
6. The user deposits fresh assets and requests a self-receiver withdrawal before period `P` ends.
7. `_requestWithdraw()` accepts the request, burns the fresh shares, and writes the entitlement into the already-closed `(P+1, user)` key.
8. After maturity, `claimWithdraw(P+1)` permanently reverts with `AlreadyClaimedPeriod`.
9. Re-blocklisting the account and attempting another rescue also reverts with `AlreadyClaimedPeriod`.

The PoC additionally rescues all remaining old shares before the account is unblocked. The source balance is therefore zero before it deposits 5,000 units of entirely fresh assets. The full fresh deposit is then withdrawn into the closed key and becomes permanently unreachable, excluding any explanation based on old balances or dust.

## Impact Details

Funds written into the affected account-period are permanently frozen:

* The corresponding user shares have already been burned.
* The assets are included in `pendingWithdrawAssets` and excluded from active Vault assets.
* `claimWithdraw()` permanently rejects the account-period.
* A second rescue permanently rejects the same account-period.
* There is no public cancellation, claimed-flag reset, or entitlement-migration path.

The amount is not limited to dust. The PoC demonstrates the permanent freezing of an entire 5,000-unit fresh deposit after all old withdrawals and old shares have already been rescued.

The scope is specifically one rescued `source x period` key:

* Using a different, unclaimed receiver during the same period remains safe.
* Waiting for one rollover causes a new request to target the next period and remains safe.
* However, once funds have been written into the closed key, no existing path can recover them.

This matches the selected in-scope impact: `Permanent freezing of funds`.

The report does not claim attacker profit, protocol-wide insolvency, or the ability of an arbitrary external account to freeze the entire Vault.

## References

* Vulnerable request path: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L994-L1027>
* Rescue path: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L845-L875>
* Claim path: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L788-L810>
* Unblock path: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L560-L562>
* Historical claimed-destination fix: <https://github.com/immunefi-team/audit-comp-firelight/commit/a2def47716389330c0ebc1f2006fca5c588c9469>

## Proof of Concept

## Proof of Concept

### Environment and execution

1. Check out the competition repository at branch `v1_audit_ready` (tested against commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`).
2. Install the repository dependencies with `npm install`.
3. Save the following test as `test/poc/RescuedAccountPeriodReopen.test.js`.
4. Run:

```bash
npx hardhat test test/poc/RescuedAccountPeriodReopen.test.js
```

### Complete PoC

```javascript
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')

const { deployVault } = require('../setup/fixtures')

describe('PoC: rescued account-period can receive new withdrawals that can never be claimed', function () {
  async function fixture () {
    const depositLimit = ethers.parseUnits('100000', 6)
    const ctx = await deployVault({ initial_deposit_limit: depositLimit })
    const [user, rescueRecipient, alternateReceiver] = ctx.users

    await ctx.utils.mintAndApprove(depositLimit, user)
    await ctx.firelight_vault.connect(user).deposit(
      ethers.parseUnits('6000', 6),
      user.address
    )

    return { ...ctx, user, rescueRecipient, alternateReceiver }
  }

  async function rescueCurrentTarget (ctx) {
    const targetPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n
    await ctx.firelight_vault
      .connect(ctx.user)
      .withdraw(ethers.parseUnits('2000', 6), ctx.user.address, ctx.user.address)
    await ctx.firelight_vault.connect(ctx.blocklister).addToBlocklist(ctx.user.address)
    await ctx.firelight_vault
      .connect(ctx.rescuer)
      .rescueWithdrawFromBlocklisted(
        ctx.user.address,
        ctx.rescueRecipient.address,
        [targetPeriod]
      )
    return targetPeriod
  }

  it('permanently locks a later request made after an honest rescue and unblock', async function () {
    const {
      firelight_vault: vault,
      token_contract: asset,
      blocklister,
      rescuer,
      user,
      rescueRecipient,
      config,
    } = await loadFixture(fixture)

    const firstRequest = ethers.parseUnits('2000', 6)
    const laterRequest = ethers.parseUnits('1000', 6)
    const requestWindowPeriod = await vault.currentPeriod()
    const targetPeriod = requestWindowPeriod + 1n

    // The user creates an ordinary delayed withdrawal.
    await vault.connect(user).withdraw(firstRequest, user.address, user.address)
    expect(await vault.withdrawalsOf(targetPeriod, user.address)).to.equal(firstRequest)
    expect(targetPeriod).to.equal((await vault.currentPeriod()) + 1n)

    // Compliance roles perform the documented blocklist rescue flow.
    await vault.connect(blocklister).addToBlocklist(user.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      user.address,
      rescueRecipient.address,
      [targetPeriod]
    )

    // The rescued item is a non-zero P+1 pending withdrawal. No rollover or
    // invalid period data is needed.
    expect(await vault.currentPeriod()).to.equal(requestWindowPeriod)
    expect(targetPeriod).to.equal((await vault.currentPeriod()) + 1n)
    expect(await vault.withdrawalsOf(targetPeriod, rescueRecipient.address)).to.equal(firstRequest)

    // Rescue closes the source account-period even though it is still writable.
    expect(await vault.isWithdrawClaimed(targetPeriod, user.address)).to.equal(true)
    expect(await vault.withdrawSharesOf(targetPeriod, user.address)).to.equal(0n)

    // The compliance restriction is lifted within the same request window.
    await vault.connect(blocklister).removeFromBlocklist(user.address)
    expect(await vault.isBlocklisted(user.address)).to.equal(false)

    // withdraw() accepts a new request to the already-closed account-period.
    await expect(vault.connect(user).withdraw(laterRequest, user.address, user.address))
      .to.emit(vault, 'WithdrawRequest')
    expect(await vault.withdrawSharesOf(targetPeriod, user.address)).to.be.greaterThan(0n)
    expect(await vault.withdrawalsOf(targetPeriod, user.address)).to.equal(laterRequest)

    await time.increase(BigInt(config.period_configuration_duration) * 2n)
    await vault.connect(rescueRecipient).claimWithdraw(targetPeriod)

    // The new request is unreachable because claim checks the old terminal flag.
    await expect(vault.connect(user).claimWithdraw(targetPeriod))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(targetPeriod)

    expect(await vault.withdrawalsOf(targetPeriod, user.address)).to.equal(laterRequest)
    expect(await vault.pendingWithdrawAssets()).to.equal(laterRequest)
    expect(await asset.balanceOf(await vault.getAddress())).to.be.greaterThanOrEqual(laterRequest)

    // A second rescue cannot recover the newly locked request either.
    await vault.connect(blocklister).addToBlocklist(user.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        user.address,
        rescueRecipient.address,
        [targetPeriod]
      )
    ).to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(targetPeriod)
  })

  it('locks an entirely new deposit even after both the old withdrawal and all old shares were rescued', async function () {
    const {
      firelight_vault: vault,
      token_contract: asset,
      blocklister,
      rescuer,
      user,
      rescueRecipient,
      config,
    } = await loadFixture(fixture)

    const oldRequest = ethers.parseUnits('2000', 6)
    const freshDeposit = ethers.parseUnits('5000', 6)
    const requestWindowPeriod = await vault.currentPeriod()
    const targetPeriod = requestWindowPeriod + 1n

    await vault.connect(user).withdraw(oldRequest, user.address, user.address)
    await vault.connect(blocklister).addToBlocklist(user.address)

    // Move both the old pending withdrawal and every remaining active share.
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      user.address,
      rescueRecipient.address,
      [targetPeriod]
    )
    await vault.connect(rescuer).rescueSharesFromBlocklisted(
      user.address,
      rescueRecipient.address
    )
    expect(await vault.currentPeriod()).to.equal(requestWindowPeriod)
    expect(targetPeriod).to.equal((await vault.currentPeriod()) + 1n)
    expect(await vault.balanceOf(user.address)).to.equal(0n)
    expect(await vault.isWithdrawClaimed(targetPeriod, user.address)).to.equal(true)

    // The account returns with an entirely fresh deposit.
    await vault.connect(blocklister).removeFromBlocklist(user.address)
    expect(await vault.isBlocklisted(user.address)).to.equal(false)
    await vault.connect(user).deposit(freshDeposit, user.address)
    expect(await vault.balanceOf(user.address)).to.equal(freshDeposit)
    expect(await vault.maxWithdraw(user.address)).to.equal(freshDeposit)

    await vault.connect(user).withdraw(freshDeposit, user.address, user.address)
    expect(await vault.withdrawalsOf(targetPeriod, user.address)).to.equal(freshDeposit)
    expect(await vault.balanceOf(user.address)).to.equal(0n)

    await time.increase(BigInt(config.period_configuration_duration) * 2n)
    await vault.connect(rescueRecipient).claimWithdraw(targetPeriod)

    await expect(vault.connect(user).claimWithdraw(targetPeriod))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(targetPeriod)

    // The entire fresh deposit remains in the Vault and cannot be reached.
    expect(await vault.pendingWithdrawAssets()).to.equal(freshDeposit)
    expect(await asset.balanceOf(await vault.getAddress())).to.be.greaterThanOrEqual(freshDeposit)

    await vault.connect(blocklister).addToBlocklist(user.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        user.address,
        rescueRecipient.address,
        [targetPeriod]
      )
    ).to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(targetPeriod)
  })

  describe('triage boundaries', function () {
    it('does not freeze a same-period request sent to a different receiver', async function () {
      const ctx = await loadFixture(fixture)
      const poisonedPeriod = await rescueCurrentTarget(ctx)
      await ctx.firelight_vault.connect(ctx.blocklister).removeFromBlocklist(ctx.user.address)

      const freshAmount = ethers.parseUnits('1000', 6)
      await ctx.firelight_vault
        .connect(ctx.user)
        .withdraw(freshAmount, ctx.alternateReceiver.address, ctx.user.address)

      expect(
        await ctx.firelight_vault.isWithdrawClaimed(
          poisonedPeriod,
          ctx.alternateReceiver.address
        )
      ).to.equal(false)

      await time.increase(BigInt(ctx.config.period_configuration_duration) * 2n)
      await expect(
        ctx.firelight_vault.connect(ctx.alternateReceiver).claimWithdraw(poisonedPeriod)
      ).to.emit(ctx.firelight_vault, 'CompleteWithdraw')
    })

    it('does not freeze the source after one period rollover because new requests target a new key', async function () {
      const ctx = await loadFixture(fixture)
      const poisonedPeriod = await rescueCurrentTarget(ctx)

      await time.increase(BigInt(ctx.config.period_configuration_duration))
      await ctx.firelight_vault.connect(ctx.blocklister).removeFromBlocklist(ctx.user.address)

      const safePeriod = (await ctx.firelight_vault.currentPeriod()) + 1n
      expect(safePeriod).to.equal(poisonedPeriod + 1n)

      const freshAmount = ethers.parseUnits('1000', 6)
      await ctx.firelight_vault
        .connect(ctx.user)
        .withdraw(freshAmount, ctx.user.address, ctx.user.address)

      expect(await ctx.firelight_vault.isWithdrawClaimed(safePeriod, ctx.user.address)).to.equal(false)

      await time.increase(BigInt(ctx.config.period_configuration_duration) * 2n)
      await expect(ctx.firelight_vault.connect(ctx.user).claimWithdraw(safePeriod))
        .to.emit(ctx.firelight_vault, 'CompleteWithdraw')
        .withArgs(ctx.user.address, freshAmount, safePeriod)
    })
  })
})
```

### Expected output

```
PoC: rescued account-period can receive new withdrawals that can never be claimed
  ✔ permanently locks a later request made after an honest rescue and unblock
  ✔ locks an entirely new deposit even after both the old withdrawal and all old shares were rescued
  triage boundaries
    ✔ does not freeze a same-period request sent to a different receiver
    ✔ does not freeze the source after one period rollover because new requests target a new key

4 passing
```

### What the PoC proves

* The rescued withdrawal is a real, non-zero request for `currentPeriod() + 1`; no invalid period or rollover is used.
* Rescue permanently closes the source key while that key is still the target of public withdrawal requests.
* After unblock, `_requestWithdraw()` accepts a new request and burns the user's shares into the closed key.
* `claimWithdraw()` and a second rescue both reject the newly accrued withdrawal with `AlreadyClaimedPeriod`.
* Even after every old share and old withdrawal has been rescued, an entirely fresh 5,000-unit deposit can be frozen in full.
* The two boundary tests confirm that the affected state is precisely one rescued `source x period` key rather than the whole account or Vault.
