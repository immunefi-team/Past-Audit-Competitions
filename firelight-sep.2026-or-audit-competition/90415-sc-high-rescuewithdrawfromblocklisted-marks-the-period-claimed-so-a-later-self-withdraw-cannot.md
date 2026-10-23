# 90415 sc high rescuewithdrawfromblocklisted marks the period claimed so a later self withdraw cannot be claimed

**Submitted on Aug 24th 2026 at 21:48:19 UTC by @Dliteofficial for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90415
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted()` sets `isWithdrawClaimed[period][from]` and moves the pending bucket. `_requestWithdraw()` never reads that flag. After a routine unblock, a self-`withdraw()` into the same period burns shares that `claimWithdraw()` will never release.

## Finding description

[`claimWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788) keys eligibility on `msg.sender` and a per-period claimed flag.

```solidity
address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
// ...
isWithdrawClaimed[period][sender] = true;
IERC20(asset()).safeTransfer(sender, assets);
```

[`rescueWithdrawFromBlocklisted()`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845) moves `withdrawSharesOf` from the blocklisted account to custody and then marks `from` claimed. Rescue itself also reverts if `from` or `to` is already claimed. The first pending bucket is the one that is meant to move.

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

A later request still writes a new bucket with no claimed check. [`_requestWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994) always books into `currentPeriod() + 1`.

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

If `receiver` is the same address that rescue just marked claimed, the new shares sit under a key that can never pass `claimWithdraw()`. That is the default ERC-4626 call `withdraw(assets, me, me)`.

The path is the documented compliance flow. A user deposits and requests a withdrawal to self for period P+1. `BLOCKLIST_ROLE` lists them. `RESCUER_ROLE` rescues that pending bucket to custody. The investigation clears and they are unblocked while still in period P. They still hold live shares, because rescue does not take those. They withdraw the remainder to self. The request succeeds and burns the shares. After P+1 ends, their `claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod`. Custody claims only the rescued first bucket. The second amount stays in `pendingWithdrawAssets` and is excluded from `totalAssets()`. There is no claim path for that self-booking.

The freeze is same-window. If they wait until the next period, the new request targets P+2 and the old flag does not apply. A second `withdraw` with `receiver != from` is claimable by that other address. That choice must be made at request time. `claimWithdraw()` has no receiver argument, so a self-booking already on chain cannot be repaired. That is not the default UI.

This is not the rescuer stealing funds. Rescue of the first bucket works as specified. The missing bound is at request. A period already marked claimed for an address still accepts a new booking for that address.

## Impact Explanation

Critical. The second booking is permanently unclaimable for the default self-receiver. Shares are burned. FXRP remains in the vault but is excluded from `totalAssets()` via `pendingWithdrawAssets`. Remaining stakers see a depressed vault. In the fixture the user deposits 6000 FXRP, 2000 is rescued and claimed by custody, and the remaining 4000 is stranded. `totalAssets()` is then 0 while the vault still holds those 4000 FXRP. Emergency re-rescue is not a repair for a self-booking already written under a claimed key.

## Likelihood Explanation

High. Blocklist, rescue, and unblock are the intended compliance path, not an extra-role attack. Unblock often lands in the same period as the original request. The vault's public `withdraw()` books to self. A user who withdraws leftover shares the way ERC-4626 is normally used hits the freeze. `BLOCKLIST_ROLE` and `RESCUER_ROLE` stay inside their documented jobs. The unprivileged `withdraw()` / `claimWithdraw()` pair is what fails after those roles acted as designed.

## Proof of Concept

## Proof of Concept

```bash
npx hardhat test test/rescue.js --grep "Rescue withdraw re-book"
```

The test is `test/rescue.js`. It does the following.

1. User A deposits 6000 FXRP.
2. A withdraws 2000 to self for period P+1.
3. A is blocklisted. Rescue moves that 2000 pending bucket to B and sets `isWithdrawClaimed[P+1][A]`.
4. A is unblocked in the same period. A still holds 4000 live shares.
5. A withdraws the remaining 4000 to self. The request succeeds and burns the shares.
6. After P+1 ends, A's `claimWithdraw(P+1)` reverts `AlreadyClaimedPeriod`.
7. B claims only the rescued 2000. `pendingWithdrawAssets` stays 4000. The vault still holds that FXRP. `totalAssets()` is 0.

add to test/rescue.js

```javascript
describe('Rescue withdraw re-book after unblock PoC', function () {
  const DECIMALS = 6
  const DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS)
  const FIRST_WITHDRAW_AMOUNT = ethers.parseUnits('2000', DECIMALS)
  const SECOND_WITHDRAW_AMOUNT = DEPOSIT_AMOUNT - FIRST_WITHDRAW_AMOUNT // 4000 leftover live shares
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const fmt6 = (n) => ethers.formatUnits(n, DECIMALS)

  it('unblocked self-withdraw into a rescued period is permanently unclaimable', async () => {
    const { token_contract, blocklister, rescuer, firelight_vault, users, utils, config } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
    )

    const userA = users[0]
    const userB = users[1]

    await utils.mintAndApprove(DEPOSIT_LIMIT, userA)
    await firelight_vault.connect(userA).deposit(DEPOSIT_AMOUNT, userA.address)

    const periodP = await firelight_vault.currentPeriod()
    const periodP1 = periodP + 1n

    console.log('\n--- setup ---')
    console.log(`A deposits:             ${fmt6(DEPOSIT_AMOUNT)} FXRP`)
    console.log(`period:                 ${periodP}`)

    await firelight_vault.connect(userA).withdraw(FIRST_WITHDRAW_AMOUNT, userA.address, userA.address)
    expect(await firelight_vault.withdrawSharesOf(periodP1, userA.address)).to.eq(FIRST_WITHDRAW_AMOUNT)
    console.log('\n--- first withdraw ---')
    console.log(`A withdraws to self:    ${fmt6(FIRST_WITHDRAW_AMOUNT)} FXRP for period ${periodP1}`)
    console.log(`A live shares left:     ${fmt6(await firelight_vault.balanceOf(userA.address))}`)

    await firelight_vault.connect(blocklister).addToBlocklist(userA.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(userA.address, userB.address, [periodP1])

    expect(await firelight_vault.isWithdrawClaimed(periodP1, userA.address)).to.eq(true)
    expect(await firelight_vault.withdrawSharesOf(periodP1, userA.address)).to.eq(0n)
    expect(await firelight_vault.withdrawSharesOf(periodP1, userB.address)).to.eq(FIRST_WITHDRAW_AMOUNT)
    console.log('\n--- rescue ---')
    console.log(`A blocklisted, pending ${fmt6(FIRST_WITHDRAW_AMOUNT)} moved to B`)
    console.log(`isWithdrawClaimed[${periodP1}][A] = true`)

    await firelight_vault.connect(blocklister).removeFromBlocklist(userA.address)
    expect(await firelight_vault.currentPeriod()).to.eq(periodP)
    expect(await firelight_vault.balanceOf(userA.address)).to.eq(SECOND_WITHDRAW_AMOUNT)
    console.log('\n--- unblock ---')
    console.log(`A unblocked in period ${periodP}. live shares still ${fmt6(SECOND_WITHDRAW_AMOUNT)}`)

    await firelight_vault.connect(userA).withdraw(SECOND_WITHDRAW_AMOUNT, userA.address, userA.address)
    expect(await firelight_vault.withdrawalsOf(periodP1, userA.address)).to.eq(SECOND_WITHDRAW_AMOUNT)
    expect(await firelight_vault.balanceOf(userA.address)).to.eq(0n)
    console.log('\n--- second withdraw (the bug) ---')
    console.log(`A withdraws remaining ${fmt6(SECOND_WITHDRAW_AMOUNT)} to self for period ${periodP1}`)
    console.log('request succeeds and burns the shares')

    while ((await firelight_vault.currentPeriod()) <= periodP1) {
      await time.increase(config.period_configuration_duration)
    }

    await expect(firelight_vault.connect(userA).claimWithdraw(periodP1))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(periodP1)
    console.log('\n--- claim ---')
    console.log(`A claimWithdraw(${periodP1}) reverts AlreadyClaimedPeriod`)

    const bBalBefore = await token_contract.balanceOf(userB.address)
    await firelight_vault.connect(userB).claimWithdraw(periodP1)
    const bReceived = (await token_contract.balanceOf(userB.address)) - bBalBefore
    expect(bReceived).to.eq(FIRST_WITHDRAW_AMOUNT)

    const pending = await firelight_vault.pendingWithdrawAssets()
    const vaultHeld = await token_contract.balanceOf(firelight_vault.target)
    const liveTa = await firelight_vault.totalAssets()
    expect(pending).to.eq(SECOND_WITHDRAW_AMOUNT)
    expect(vaultHeld).to.be.gte(SECOND_WITHDRAW_AMOUNT)
    expect(liveTa).to.eq(0n)

    console.log(`B claims rescued bucket: ${fmt6(bReceived)} FXRP`)
    console.log(`pendingWithdrawAssets:  ${fmt6(pending)} FXRP  (A's second booking)`)
    console.log(`vault still holds:      ${fmt6(vaultHeld)} FXRP`)
    console.log(`totalAssets():          ${fmt6(liveTa)}  (pending is excluded)`)
    console.log(`A's ${fmt6(SECOND_WITHDRAW_AMOUNT)} FXRP is permanently unclaimable\n`)
  })
})
```
