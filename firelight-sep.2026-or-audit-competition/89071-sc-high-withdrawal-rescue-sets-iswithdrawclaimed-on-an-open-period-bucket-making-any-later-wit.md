# 89071 sc high withdrawal rescue sets iswithdrawclaimed on an open period bucket making any later withdrawal into that bucket permanently unclaimable

**Submitted on Aug 16th 2026 at 23:54:16 UTC by @xeQted for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89071
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted` writes `isWithdrawClaimed[period][from] = true` at `FirelightVault.sol:870` to mark the rescued shares as consumed. `claimWithdraw` reads that same mapping at `FirelightVault.sol:794` and reverts before computing any amount. The mapping has no clearing path anywhere in the contract. Once an account is rescued from period `p`, any withdrawal it is later credited into bucket `p` burns vault shares at `FirelightVault.sol:1017` and can never be claimed.

`rescueWithdrawFromBlocklisted` is the only function that can set the flag while the bucket still accepts credits, because the credit window (`currentPeriod() + 1 == p`, `FirelightVault.sol:1008`) and the claim window (`currentPeriod() > p`, `FirelightVault.sol:791`) are otherwise disjoint.

### Affected code

| File                                | Lines | Role                                            |
| ----------------------------------- | ----: | ----------------------------------------------- |
| `contracts/core/FirelightVault.sol` |   870 | Sets the flag on an open bucket                 |
| `contracts/core/FirelightVault.sol` |   794 | Reads the flag, reverts before computing assets |
| `contracts/core/FirelightVault.sol` |   864 | Same read blocks the only recovery route        |
| `contracts/core/FirelightVault.sol` |  1017 | Credits the bucket after the flag is set        |

## Root cause

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal shares to another address, then flags the source account:

```solidity
// contracts/core/FirelightVault.sol:861-873
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);  /
    if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);  // 865
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);                       /

    withdrawSharesOf[periods[i]][to] += _withdrawOf;                                   /
    withdrawSharesOf[periods[i]][from] = 0;                                            // 869
    isWithdrawClaimed[periods[i]][from] = true;                                        /
```

`claimWithdraw` treats the same mapping as "this account has been paid for this period":

```solidity
// contracts/core/FirelightVault.sol:791-803
if (period >= currentPeriod()) revert InvalidPeriod();                                //

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);            // 794

assets = _convertToAssetsTotals(                                                      // 796
    withdrawSharesOf[period][sender], withdrawShares[period], withdrawAssets[period], Ma
);

if (assets == 0) revert NoWithdrawalAmount(period);                                   // 803
```

Line 794 executes before line 796, so a nonzero `withdrawSharesOf[period][sender]` is never read.

`isWithdrawClaimed` is the only boolean in `FirelightVaultStorage.sol` that is set true with no assignment back to false. `isBlocklisted` is cleared at `FirelightVault.sol:561`, `isPayoutAllowlisted` at `FirelightVault.sol:580`, and `hasActiveIncident` takes a caller-supplied value at Firelight

A later fix added the `to`-side check at line 865. No equivalent check exists for the `from`

## Impact

The victim's vault shares are burned at `FirelightVault.sol:1017` and the corresponding `as[p]` and `pendingWithdrawAssets` for the lifetime of the contract. No function returns them. `claimWithdraw` reverts at line 794 and `rescueWithdrawFromBlocklisted` reverts at line 864 on the same flag, so the two functions that move assets out of a withdrawal bucket are both closed. Recovery requires a contract upgrade that writes storage to clear the flag, not a logic patch.

The stranded assets stay subtracted from `totalAssets()` at `FirelightVault.sol:428` permaneot a solvency gap: `totalAssetsAt` feeds both the payout exposure cap and `CoverOrderAllocator._computeAvailableCapacity`, so it moves both together.

Sanity check against the Immunefi Vulnerability Severity Classification System v2.3, smart contract category: the impact is permanent and unrecoverable without an upgrade, which is Critical. The trigger requires two role-gated calls, stated below. A triager weighting likelihood rather than

## Attack scenario

No attacker participates. Each step is an operation the contract exposes for its stated ctim's own routine withdrawal. All steps occur inside one period, 1 week in the defaultconfiguration.

{% stepper %}
{% step %}

### Period p-1: Victim requests a withdrawal

Victim calls `withdraw`. `withdrawSharesOf[p][victim]` is credited at line 1017, vault shares are burned, assets are added to `withdrawAssets[p]` and `pendingWithdrawAssets`.

Cost: none, normal user action.
{% endstep %}

{% step %}

### Victim is blocklisted

Victim is blocklisted via `addToBlocklist` at line 550.

Precondition: `BLOCKLIST_ROLE`.
{% endstep %}

{% step %}

### Pending withdrawal is rescued

`rescueWithdrawFromBlocklisted(victim, X, [p])` moves the shares to `X` and sets `isWithdrt` line 870.

Precondition: `RESCUER_ROLE`.

`X` is an arbitrary address parameter, so thiscovers migrating a beneficial owner's pending withdrawal to a fresh compliant address.
{% endstep %}

{% step %}

### Victim is removed from the blocklist

Victim is cleared via `removeFromBlocklist` at line 560.

Precondition: `BLOCKLIST_ROLE`.

the sequence to continue: `withdraw` carries `notBlocklisted(receiver)` at line 765, as doesevery other path that can credit a bucket.
{% endstep %}

{% step %}

### Victim withdraws again into the same bucket

Still period p-1. Victim calls `withdraw` again. The call succeeds. Nothing in `_requestmed`, so no revert or event signals the pending loss.

Cost: none, normal user action.
{% endstep %}

{% step %}

### Claim fails permanently

Period p+1. `claimWithdraw(p)` reverts `AlreadyClaimedPeriod(p)` at line 794.

The victim discovers the loss one to two periods after the shares were burned.
{% endstep %}
{% endstepper %}

Period numbers cannot collide, so the same-period constraint in step 5 is required: `_addPeriodConfiguration` rejects an epoch earlier than `nextPeriodEnd()` at `FirelightVault.sol:1093` and accumulates `startingPeriod` at line 1096.

## Recommended fix

```diff
--- a/contracts/core/FirelightVault.sol
+++ b/contracts/core/FirelightVault.sol
@@ -868,7 +868,6 @@
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;
```

Line 869 already covers both jobs line 870 was doing. A second rescue of the same `(period, from)` reads `_withdrawOf == 0` and reverts `NoWithdrawalAmount` at line 866. A `claimWithdraw` by `from` computes assets from `withdrawSharesOf`, now zero, and reverts `NoWithdrawalAmount` at line 803.

Retain line 864. `claimWithdraw` sets `isWithdrawClaimed` at line 806 without zeroing `withdrt` has legitimately claimed period p still holds a nonzero `withdrawSharesOf[p][account]`. Line 864 is what stops a rescue from moving already-paid shares to `to` for a second payout.

If a consumed-marker is needed for off-chain accounting, add a separate `isWithdrawRescued[period][account]` that `claimWithdraw` does not read.

## Relationship to prior audit findings

The OpenZeppelin/Sentora review of 2025-07-21, finding M-02 "Period Duration Cannot Be Changed", reaches this same revert at line 794, but from a different cause: a `periodDuration` change that made two withdrawal requests land on the same period number. It was closed as resolved, and the frations are scheduled.

## Proof of Concept

Runs against the repository's stock Hardhat suite. No source modification. The test uses the repo's own `deployVault` fixture from `test/setup/fixtures.js`, which is what grants `RESCUER_ROLE` and `BLOCKLIST_ROLE`.

Save as `test/poc_rescue_flag_poison.js`:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

// PoC: rescueWithdrawFromBlocklisted permanently poisons isWithdrawClaimed[period][account].
// The flag is used as a "shares consumed" marker but is never cleared, so ANY later credit
// into the same withdrawal bucket for that account becomes permanently unclaimable.
describe('PoC / isWithdrawClaimed poisoning permanently freezes a re-withdrawal', function () {
  const DECIMALS = 6
  const DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS)
  const WITHDRAW_1 = ethers.parseUnits('2000', DECIMALS)
  const WITHDRAW_2 = ethers.parseUnits('1500', DECIMALS)
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  let token_contract, blocklister, rescuer, firelight_vault, users, utils, config
  let victim, rescueBeneficiary, bucket

  before(async () => {
    ({ token_contract, blocklister, rescuer, firelight_vault, users, utils, config } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
    ))
    victim = users[0]
    rescueBeneficiary = users[2]

    await utils.mintAndApprove(DEPOSIT_LIMIT, victim)
    await firelight_vault.connect(victim).deposit(DEPOSIT_AMOUNT, victim.address)
  })

  it('step 1: victim requests a withdrawal into bucket p', async () => {
    bucket = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_1, victim.address, victim.address)
    expect(await firelight_vault.withdrawSharesOf(bucket, victim.address)).to.be.gt(0n)
    console.log('        bucket period                 =', bucket.toString())
    console.log('        withdrawSharesOf[p][victim]   =', (await firelight_vault.withdrawSharesOf(bucket, victim.address)).toString())
  })

  it('step 2-3: victim is blocklisted, rescuer moves the pending withdrawal away', async () => {
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      victim.address, rescueBeneficiary.address, [bucket]
    )
    // shares moved out, and the flag is now permanently set for (bucket, victim)
    expect(await firelight_vault.withdrawSharesOf(bucket, victim.address)).to.eq(0n)
    expect(await firelight_vault.isWithdrawClaimed(bucket, victim.address)).to.eq(true)
    console.log('        after rescue: isWithdrawClaimed[p][victim] =', await firelight_vault.isWithdrawClaimed(bucket, victim.address))
  })

  it('step 4-5: victim is cleared from the blocklist and withdraws again into the SAME bucket', async () => {
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)
    expect(await firelight_vault.isBlocklisted(victim.address)).to.eq(false)

    // still the same period, so this lands in the same bucket p
    expect((await firelight_vault.currentPeriod()) + 1n).to.eq(bucket)
    await firelight_vault.connect(victim).withdraw(WITHDRAW_2, victim.address, victim.address)

    const owed = await firelight_vault.withdrawalsOf(bucket, victim.address)
    expect(await firelight_vault.withdrawSharesOf(bucket, victim.address)).to.be.gt(0n)
    expect(owed).to.be.gt(0n)
    console.log('        re-withdraw credited, assets owed to victim =', owed.toString())
  })

  it('step 6: claimWithdraw reverts forever, the funds are permanently frozen', async () => {
    await time.increase(config.period_configuration_duration * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(bucket)

    const owed = await firelight_vault.withdrawalsOf(bucket, victim.address)
    expect(owed).to.be.gt(0n)

    await expect(firelight_vault.connect(victim).claimWithdraw(bucket))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(bucket)

    console.log('        FROZEN: vault still owes victim', owed.toString(), 'but claimWithdraw reverts AlreadyClaimedPeriod')
  })

  it('step 7: even the rescue path cannot recover them, no in-protocol recovery exists', async () => {
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    // rescueWithdrawFromBlocklisted checks the very same poisoned flag (L864) and reverts
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        victim.address, rescueBeneficiary.address, [bucket]
      )
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(bucket)

    const owed = await firelight_vault.withdrawalsOf(bucket, victim.address)
    console.log('        rescue also reverts; assets permanently stranded =', owed.toString())
  })
})
```

Run:

```bash
npm install
npx hardhat compile
npx hardhat test test/poc_matrix_rescue_flag_poison.js
```

Output:

```
  PoC / isWithdrawClaimed poisoning permanently freezes a re-withdrawal
        bucket period                 = 1
        withdrawSharesOf[p][victim]   = 2000000000
    ✔ step 1: victim requests a withdrawal into bucket p
        after rescue: isWithdrawClaimed[p][victim] = true
    ✔ step 2-3: victim is blocklisted, rescuer moves the pending withdrawal away
        re-withdraw credited, assets owed to victim = 1500000000
    ✔ step 4-5: victim is cleared from the blocklist and withdraws again into the SAME bucket
        FROZEN: vault still owes victim 1500000000 but claimWithdraw reverts AlreadyClaimedPeriod
    ✔ step 6: claimWithdraw reverts forever, the funds are permanently frozen
        rescue also reverts; assets permanently stranded = 1500000000
    ✔ step 7: even the rescue path cannot recover them, no in-protocol recovery exists

  5 passing (1s)
```

`1500000000` is 1,500 fXRP at 6 decimals. Steps 6 and 7 assert `revertedWithCustomError(..., 'AlreadyClaimedPeriod').withArgs(1)`, so both `claimWithdraw(1)` and `rescueWithdrawFromBlocklisted(victim, X, [1])` are proven closed against a balance the vault still reports as owed via `withdrawalsOf(1, victim)`.

Environment: Node v24.15.0, Hardhat 2.22.19, solc 0.8.28 and 0.8.29, evm target london, Linux x86\_64.
