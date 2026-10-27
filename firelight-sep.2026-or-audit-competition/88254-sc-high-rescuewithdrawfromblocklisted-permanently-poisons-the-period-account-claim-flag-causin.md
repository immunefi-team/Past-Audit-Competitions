# 88254 sc high rescuewithdrawfromblocklisted permanently poisons the period account claim flag causing unrecoverable freezing of any later withdrawal the account makes into the same still&#x20;

**Submitted on Aug 12th 2026 at 12:36:17 UTC by @Oxb4b for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88254
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault` uses a single boolean, `isWithdrawClaimed[period][account]`, both to mark a withdrawal bucket as claimed by `claimWithdraw` and to mark a rescued balance as "consumed" by `rescueWithdrawFromBlocklisted`. The flag is never cleared once set. Because `rescueWithdrawFromBlocklisted` is designed to act on the *currently open* bucket (`currentPeriod() + 1`), and blocklisting is fully reversible, a legitimate compliance workflow — blocklist → rescue → un-blocklist, all within one period — leaves the flag permanently `true` for the account on that period. Any withdrawal the account subsequently requests in the same still-open period is credited to that poisoned bucket and becomes permanently unclaimable, with no recovery path on-chain, including via a second rescue. The underlying assets remain physically in the vault but are permanently excluded from `totalAssets()` through `pendingWithdrawAssets`, harming the victim and diluting backing for every other staker.

***

## Vulnerability Details

**Affected contract/functions:** `FirelightVault.sol` — `rescueWithdrawFromBlocklisted()`, `_requestWithdraw()`, `claimWithdraw()`, `removeFromBlocklist()`.

**Root cause:**

`isWithdrawClaimed[period][account]` has exactly two writers, and no function ever resets it to `false`:

* `claimWithdraw(period)` sets `isWithdrawClaimed[period][sender] = true` after paying out (line 806).
* `rescueWithdrawFromBlocklisted(from, to, periods)` sets `isWithdrawClaimed[periods[i]][from] = true` (line 870) after moving `withdrawSharesOf[p][from]` to `withdrawSharesOf[p][to]` and zeroing the source.

Both writers treat the flag as if it permanently describes the `(period, account)` pair. In reality it is meant to describe a *specific balance at a specific moment* — whether that balance has been consumed. The two concepts diverge because:

1. `_requestWithdraw` always credits the *currently open* bucket, `period = currentPeriod() + 1` (line 1008). This bucket is not yet claimable (`claimWithdraw` requires `period < currentPeriod()`, line 791), so it can still receive new deposits for the remainder of the current period.
2. `addToBlocklist` / `removeFromBlocklist` (lines 550–563) are fully reversible and carry no time lock, so an account can return to good standing while the just-rescued bucket is still open.
3. `_requestWithdraw` never reads `isWithdrawClaimed` before crediting `withdrawSharesOf[period][receiver]` — it has no way of knowing the bucket was already flagged for that account.

Once the flag is `true` for `(p, account)`, there is no path to reverse it:

* `claimWithdraw` reverts immediately on the flag check (line 794), regardless of the account's actual non-zero balance in `withdrawSharesOf[p][account]`.
* `rescueWithdrawFromBlocklisted` guards on the *same* flag (`if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);`, line 864), so re-blocklisting the victim to attempt a second rescue also reverts.
* No admin sweep or alternate write path to `withdrawSharesOf` or `isWithdrawClaimed` exists.

The loss is compounded on the accounting side: the second withdrawal burns the victim's shares (`_update(owner, address(0), shares)`, line 1025) and increments `pendingWithdrawAssets += assets` (line 1019), but that increment is never reversed since the claim can never succeed. Since `totalAssets() = super.totalAssets() - pendingWithdrawAssets` (line 428), the corresponding tokens stay in the vault's ERC-20 balance but are permanently excluded from share value for every remaining staker, not just the victim.

***

## Impact and Likelihood Details

**Impact: Critical — Permanent freezing of funds** (per `scope.md`'s in-scope Critical category).

* **Individual user:** shares are burned with no corresponding claimable asset entry; the withdrawal is unrecoverable through any documented function.
* **Protocol-wide:** `pendingWithdrawAssets` is permanently inflated by the frozen amount, so `totalAssets()` under-reports real backing forever. This dilutes share value/redemption rate for all other stakers and corrupts any downstream logic relying on `totalAssets()` (e.g. capital-adequacy or exposure calculations).
* No on-chain remediation exists; recovery requires a contract upgrade.

**Likelihood: Medium**

* Requires a blocklist → rescue → un-blocklist sequence to complete within a single period, followed by a new withdrawal from the same account before the period rolls over.
* This is not a contrived attacker sequence — it is the natural shape of a routine compliance workflow. Sanctions/KYC screening hits are blocklisted, rescued to protect the position, then cleared once verified as a false positive, and `removeFromBlocklist` exists specifically to support that reversal. The contract gives operators no indication that the account is now poisoned for the remainder of that period.
* The exposure window is up to one full period (e.g. 24h at `SMALLEST_PERIOD_DURATION`), which is a realistic amount of time for a compliance review-and-clear cycle to complete, and for the affected user to attempt a normal withdrawal in the interim.

**In-scope justification:** All privileged actions in the attack path (`BLOCKLIST_ROLE`'s `addToBlocklist`/`removeFromBlocklist`, `RESCUER_ROLE`'s `rescueWithdrawFromBlocklisted`) are used entirely within their documented authority. The loss is triggered by an unprivileged victim performing the single most ordinary action available — a normal `withdraw()` call. Per `scope.md`, reports requiring privileged roles to act within granted permissions are out of scope *unless* the report demonstrates the role can exceed its intended authority or bypass an intended protocol constraint. This report demonstrates the latter: the intended constraint that a burned withdrawal request always becomes claimable after its delay is bypassed as a side effect of correct rescue usage. This is a core state-machine defect, not a governance/trusted-role-misuse report.

***

## References

* `FirelightVault.sol::rescueWithdrawFromBlocklisted()` — lines 845–876, flag write at line 870, guards at lines 864–865.
* `FirelightVault.sol::_requestWithdraw()` — bucket credit at lines 1008–1019.
* `FirelightVault.sol::claimWithdraw()` — claim-flag guard at line 794, period guard at line 791.
* `FirelightVault.sol::removeFromBlocklist()` / `addToBlocklist()` — lines 550–563.
* `FirelightVault.sol::totalAssets()` — line 428 (`super.totalAssets() - pendingWithdrawAssets`).
* `scope.md` — "Critical: Permanent freezing of funds"; privileged-actor scope carve-out ("...unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint").
* Proof of Concept below (companion regression test appended to `rescue.js` suite).

## Proof of Concept

## Proof of Concept (clear scenarios)

Scenario walkthrough, with `C = currentPeriod()` and the open bucket `p = C + 1`:

1. **Setup.** Alice deposits into the vault and calls `withdraw(FIRST_WITHDRAW_AMOUNT, alice, alice)` while inside period `C`. This credits `withdrawSharesOf[p][alice] = FIRST_WITHDRAW_AMOUNT`-equivalent, sets `withdrawAssets[p] += FIRST_WITHDRAW_AMOUNT`, `pendingWithdrawAssets += FIRST_WITHDRAW_AMOUNT`, and burns her shares. `isWithdrawClaimed[p][alice]` is `false`.
2. **Compliance action.** `BLOCKLIST_ROLE` calls `addToBlocklist(alice)` — a legitimate, documented compliance action.
3. **Rescue.** `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, treasury, [p])`. Result: `withdrawSharesOf[p][alice] = 0`, `withdrawSharesOf[p][treasury] += FIRST_WITHDRAW_AMOUNT`-equivalent, and **`isWithdrawClaimed[p][alice] = true`**.
4. **Resolution.** The screening hit is cleared. `BLOCKLIST_ROLE` calls `removeFromBlocklist(alice)`. All actions so far are within documented role authority.
5. **Ordinary trigger.** Still inside period `C`, Alice deposits again and calls `withdraw(SECOND_WITHDRAW_AMOUNT, alice, alice)`. `_requestWithdraw` computes `period = currentPeriod() + 1 = p` — the same bucket. It credits `withdrawSharesOf[p][alice] = SECOND_WITHDRAW_AMOUNT`-equivalent, adds to `withdrawAssets[p]` and `pendingWithdrawAssets`, and burns Alice's shares. No check consults `isWithdrawClaimed`.
6. **Loss realized.** Once period `p` becomes claimable (`currentPeriod() > p`), Alice calls `claimWithdraw(p)`. The function reverts `AlreadyClaimedPeriod(p)` because `isWithdrawClaimed[p][alice]` is still `true` from step 3, despite her non-zero, never-claimed balance from step 5.
7. **Recovery proven impossible.** Operators re-blocklist Alice and attempt `rescueWithdrawFromBlocklisted(alice, treasury, [p])` to move the stranded balance out. This also reverts with `AlreadyClaimedPeriod(p)`, since the rescue guard reads the same permanently-set flag. `withdrawSharesOf[p][alice]` remains stuck forever, and `pendingWithdrawAssets` is never decremented for that amount.

This exact sequence is implemented and verified end-to-end in the accompanying Hardhat test provided below (run against the `rescue.js` fixtures), which asserts at each step:

* `isWithdrawClaimed(p, alice) == false` immediately after the first withdrawal (step 1).
* `withdrawSharesOf(p, alice) == 0` and `isWithdrawClaimed(p, alice) == true` immediately after rescue (step 3).
* `withdrawSharesOf(p, alice) == SECOND_WITHDRAW_AMOUNT` after the second withdrawal, with the flag still `true` (step 5).
* `claimWithdraw(p)` reverts with `AlreadyClaimedPeriod(p)` once claimable, and `pendingWithdrawAssets` is unchanged by the failed claim (step 6).
* A second `rescueWithdrawFromBlocklisted` attempt also reverts with `AlreadyClaimedPeriod(p)`, and Alice's balance remains stranded (step 7).

Every step uses only public, documented entry points (`deposit`, `withdraw`, `addToBlocklist`, `removeFromBlocklist`, `rescueWithdrawFromBlocklisted`, `claimWithdraw`) with roles used exactly as intended — no privilege escalation, no reentrancy, no unusual timing beyond staying within a single period.

***

## POC Testcase

add the below testcase to the existing test file `test/rescue.js` and run

```js
describe('Rescue test - poisoned isWithdrawClaimed flag blocks later legitimate withdrawals', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    FIRST_WITHDRAW_AMOUNT = ethers.parseUnits('1000', DECIMALS),
    SECOND_WITHDRAW_AMOUNT = ethers.parseUnits('500', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  let alice, treasury, poisonedPeriod

  before(async () => {
    ; ({ token_contract, blocklister, rescuer, firelight_vault, users, utils, config } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
    ))

    alice = users[0]
    treasury = users[1]

    await utils.mintAndApprove(DEPOSIT_LIMIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT_AMOUNT, alice.address)
  })

  it('step 1: alice requests a withdrawal that lands in the still-open bucket P', async () => {
    const currentPeriod = await firelight_vault.currentPeriod()
    poisonedPeriod = currentPeriod + 1n

    const withdrawTrx = firelight_vault.connect(alice).withdraw(FIRST_WITHDRAW_AMOUNT, alice.address, alice.address)
    await expect(withdrawTrx).to.emit(firelight_vault, 'WithdrawRequest').withArgs(
      alice.address, alice.address, alice.address, poisonedPeriod, FIRST_WITHDRAW_AMOUNT, FIRST_WITHDRAW_AMOUNT
    )

    expect(await firelight_vault.isWithdrawClaimed(poisonedPeriod, alice.address)).to.eq(false)
  })

  it('step 2: blocklister blocklists alice (legitimate compliance action)', async () => {
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.eq(true)
  })

  it('step 3: rescuer rescues the pending withdrawal for period P to treasury, poisoning the flag', async () => {
    const rescueTrx = firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      alice.address, treasury.address, [poisonedPeriod]
    )
    await expect(rescueTrx).to.emit(firelight_vault, 'WithdrawRescuedFromBlocklisted').withArgs(
      alice.address, treasury.address, [poisonedPeriod], [FIRST_WITHDRAW_AMOUNT]
    )

    expect(await firelight_vault.withdrawSharesOf(poisonedPeriod, alice.address)).to.eq(0)
    expect(await firelight_vault.isWithdrawClaimed(poisonedPeriod, alice.address)).to.eq(true)
  })

  it('step 4: blocklister removes alice from blocklist (false positive cleared)', async () => {
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.eq(false)
  })

  it('step 5: still inside the same open bucket, alice performs an ordinary second withdrawal', async () => {
    expect(await firelight_vault.currentPeriod() + 1n).to.eq(poisonedPeriod)

    const withdrawTrx = firelight_vault.connect(alice).withdraw(SECOND_WITHDRAW_AMOUNT, alice.address, alice.address)
    await expect(withdrawTrx).to.emit(firelight_vault, 'WithdrawRequest').withArgs(
      alice.address, alice.address, alice.address, poisonedPeriod, SECOND_WITHDRAW_AMOUNT, SECOND_WITHDRAW_AMOUNT
    )

    expect(await firelight_vault.withdrawSharesOf(poisonedPeriod, alice.address)).to.eq(SECOND_WITHDRAW_AMOUNT)
    expect(await firelight_vault.isWithdrawClaimed(poisonedPeriod, alice.address)).to.eq(true)
  })

  it('step 6: once period P is claimable, alice can never claim her legitimate new withdrawal', async () => {
    while ((await firelight_vault.currentPeriod()) <= poisonedPeriod) {
      await time.increaseTo((await firelight_vault.currentPeriodEnd()) + 1n)
    }

    const pendingBefore = await firelight_vault.pendingWithdrawAssets()

    const claimAttempt = firelight_vault.connect(alice).claimWithdraw(poisonedPeriod)
    await expect(claimAttempt).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(poisonedPeriod)

    expect(await firelight_vault.pendingWithdrawAssets()).to.eq(pendingBefore)
    expect(await firelight_vault.withdrawSharesOf(poisonedPeriod, alice.address)).to.eq(SECOND_WITHDRAW_AMOUNT)
  })

  it('step 7: recovery via re-blocklist + rescue is also impossible, proving permanent loss', async () => {
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)

    const rescueAttempt = firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      alice.address, treasury.address, [poisonedPeriod]
    )
    await expect(rescueAttempt).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(poisonedPeriod)

    expect(await firelight_vault.withdrawSharesOf(poisonedPeriod, alice.address)).to.eq(SECOND_WITHDRAW_AMOUNT)
  })
})
```
