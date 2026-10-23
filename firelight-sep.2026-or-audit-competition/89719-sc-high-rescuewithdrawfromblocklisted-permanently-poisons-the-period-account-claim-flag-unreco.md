# 89719 sc high rescuewithdrawfromblocklisted permanently poisons the period account claim flag unrecoverably freezing any later withdrawal into the same still open bucket

**Submitted on Aug 20th 2026 at 18:39:32 UTC by @Oxb4b for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89719
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted()` sets `isWithdrawClaimed[period][from] = true` when it moves a blocklisted account's pending withdrawal to a rescue recipient. This mapping has no writer that ever clears it back to `false`, and the rescue function does not restrict itself to closed periods — it can act on the currently open bucket (`currentPeriod() + 1`), which is still accepting new withdrawal requests via `_requestWithdraw`.

Because `addToBlocklist`/`removeFromBlocklist` are freely reversible, a rescued account can be un-blocklisted and immediately submit a brand-new, unrelated withdrawal that lands in the same already-poisoned bucket. That new withdrawal can never be claimed: `claimWithdraw` reverts on the stale flag, and `rescueWithdrawFromBlocklisted` cannot be used to recover it either, since its own guard rejects accounts whose flag is already set. The result is a permanent, unrecoverable freeze of legitimately deposited/withdrawn funds, reachable through an entirely routine compliance workflow plus one ordinary, unprivileged withdrawal.

***

## Vulnerability Details

**Affected contract:** `FirelightVault.sol` **Affected functions:** `rescueWithdrawFromBlocklisted()` (flag write), `_requestWithdraw()` (bucket credit), `claimWithdraw()` (claim guard), `removeFromBlocklist()`, `totalAssets()`

**Root cause**

`isWithdrawClaimed[period][account]` has exactly two writers, both setting it to `true`, and no writer ever resets it to `false`:

1. `claimWithdraw()` sets it after paying out the account's balance for a *closed* period (`period < currentPeriod()`).
2. `rescueWithdrawFromBlocklisted()` sets it on the `from` side after transferring `withdrawSharesOf[period][from]` to `withdrawSharesOf[period][to]` and zeroing the source — with **no restriction that `period` must already be closed**.

`_requestWithdraw()` always credits the *currently open* bucket, `period = currentPeriod() + 1`. That is precisely the bucket a rescue can legally target, since nothing in `rescueWithdrawFromBlocklisted` requires the period to have ended.

The flag is meant to encode "this specific balance was already consumed," a fact about one balance at one moment. Instead it is keyed permanently to `(period, account)`, treating it as if that account can never again hold a legitimate balance in that period. Since blocklist status is reversible, an account can be rescued, cleared of the flag's underlying balance, un-blocklisted, and then legitimately request a *new* withdrawal into the same period — which the flag now blocks forever.

**Why there is no recovery path**

* `claimWithdraw()` reverts with `AlreadyClaimedPeriod` because the flag is `true`, regardless of the fact that the current balance in `withdrawSharesOf[period][account]` is a fresh, unrelated request.
* `rescueWithdrawFromBlocklisted()` cannot be used a second time to move the stranded balance out either: its own precondition `if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);` rejects exactly the accounts whose flag is already set — including the victim, even if re-blocklisted for another rescue attempt.
* No admin sweep function exists. No other function writes to `withdrawSharesOf[period][account]` for that period.

**Confirmed unique root cause**

This is the only way to reach a state where `isWithdrawClaimed[p][a] == true` while `withdrawSharesOf[p][a] > 0`: `claimWithdraw` can only set the flag for `p < currentPeriod()` (a closed period), and `_requestWithdraw` only ever credits `currentPeriod() + 1` (the open period) — so an account can never re-request into a period it already self-claimed. Only the rescue path can set the flag on a period that is still open for new requests, which is the precondition for this bug.

**Secondary consequence — protocol-wide accounting leak**

The frozen withdrawal's shares are burned via `_update(owner, address(0), shares)` and `pendingWithdrawAssets += assets` is incremented in `_requestWithdraw`, but is never decremented for the frozen entry (since `claimWithdraw` — the only decrementer — can never succeed for it). Since `totalAssets() = super.totalAssets() - pendingWithdrawAssets`, the frozen tokens remain physically in the vault but are permanently excluded from share value and from `activePayableAmount` in `payout()`, depressing share price and future claim-paying capacity for every remaining staker.

***

## Impact and Likelihood Details

**Impact: High (Permanent freezing of funds)**

* The individual account loses access to a legitimately-requested withdrawal permanently — shares are burned, but the corresponding assets can never be claimed by any address.
* Because `_requestWithdraw` credits `withdrawSharesOf[period][receiver]` (not the transaction sender/owner), any staker can accidentally or maliciously burn a third party's shares into a poisoned bucket by naming a previously-rescued (now un-blocklisted) address as `receiver` — extending the blast radius beyond the originally poisoned account.
* Every remaining staker is affected indirectly: the stuck assets inflate `pendingWithdrawAssets` forever, permanently understating `totalAssets()`, which lowers the share price and shrinks the exposure cap available to `payout()` for future incident claims.
* No on-chain remediation exists; recovery requires a contract upgrade.

**Likelihood: Medium**

* Requires the sequence blocklist → rescue (on the open bucket) → un-blocklist to complete within a single vault period, followed by any withdrawal crediting that account in the same period.
* Every step up to the trigger is a normal, documented use of `BLOCKLIST_ROLE` and `RESCUER_ROLE` — blocklisting a sanctioned/flagged address, rescuing its pending withdrawal, and later reversing a false positive via `removeFromBlocklist` (which exists precisely for this purpose).
* The triggering action is the single most ordinary behavior a returning user performs: a normal `withdraw()` call. No timing race, no special privilege, and no unusual behavior is required.
* The contract emits no signal that a `(period, account)` pair is poisoned; `isWithdrawClaimed` is a public mapping but nothing in the withdrawal path checks or warns against it.

**Scope justification**

`BLOCKLIST_ROLE` and `RESCUER_ROLE` act entirely within their documented authority at every step. The loss is caused by a state-machine defect in the contract's claim-flag lifecycle, triggered by an unprivileged user performing a routine withdrawal. This bypasses the intended protocol constraint that a burned, delayed withdrawal request is always claimable after its delay — the exception scope explicitly preserves for reports demonstrating a role "bypass\[ing] an intended protocol constraint." This is not a "trusted role acts maliciously/incorrectly" report; every privileged action is correct and expected.

***

## References

* `rescueWithdrawFromBlocklisted()` — flag write and guards
* `_requestWithdraw()` — open-bucket credit logic (`period = currentPeriod() + 1`)
* `claimWithdraw()` — claim guard reading `isWithdrawClaimed`
* `removeFromBlocklist()` / `addToBlocklist()` — reversible blocklist state
* `totalAssets()` — `super.totalAssets() - pendingWithdrawAssets`
* `payout()` — `activePayableAmount` computation excluding frozen `pendingWithdrawAssets`
* scope — "Permanent freezing of funds" (Critical, in-scope); privileged-actor exception for protocol-constraint bypass
* Test file: `test/rescue.js` (existing suite exercises rescue in isolation and never re-requests into a rescued bucket)

## Proof of Concept

## Proof of Concept

```javascript

//add this testcase to the existing test file `test/rescue.js` and run

describe('rescue poisons isWithdrawClaimed for a still-open bucket, freezing later withdrawals', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    FIRST_WITHDRAW_AMOUNT = ethers.parseUnits('2000', DECIMALS),
    SECOND_WITHDRAW_AMOUNT = ethers.parseUnits('1000', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  let token_contract, blocklister, rescuer, firelight_vault, users, utils, config

  before(async () => {
    ({ token_contract, blocklister, rescuer, firelight_vault, users, utils, config } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
    ))

    await utils.mintAndApprove(DEPOSIT_LIMIT, users[0])
    await firelight_vault.connect(users[0]).deposit(DEPOSIT_AMOUNT, users[0].address)
  })

  it('permanently freezes a later, unrelated withdrawal into the same rescued-then-reopened bucket', async () => {
    const alice = users[0]
    const treasury = users[1]

    const startPeriod = await firelight_vault.currentPeriod()
    const poisonedPeriod = startPeriod + 1n

    // Alice withdraws into bucket poisonedPeriod = currentPeriod() + 1
    await firelight_vault.connect(alice).withdraw(FIRST_WITHDRAW_AMOUNT, alice.address, alice.address)
    expect(await firelight_vault.withdrawalsOf(poisonedPeriod, alice.address)).to.eq(FIRST_WITHDRAW_AMOUNT)

    // Blocklist Alice (compliance action)
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)

    // Rescue moves her pending withdrawal to treasury and sets isWithdrawClaimed[poisonedPeriod][alice] = true
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, treasury.address, [poisonedPeriod])
    expect(await firelight_vault.withdrawalsOf(poisonedPeriod, alice.address)).to.eq(0)

    // Un-blocklist Alice (false positive cleared)
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.eq(false)

    // Still in same period; still inside period = currentPeriod() + 1
    expect(await firelight_vault.currentPeriod()).to.eq(startPeriod)

    // Alice withdraws again into the same poisoned bucket (no check consults isWithdrawClaimed on request)
    await firelight_vault.connect(alice).withdraw(SECOND_WITHDRAW_AMOUNT, alice.address, alice.address)
    expect(await firelight_vault.withdrawalsOf(poisonedPeriod, alice.address)).to.eq(SECOND_WITHDRAW_AMOUNT)

    // Advance past the period to make it claimable
    await time.increase(config.period_configuration_duration)
    await time.increase(config.period_configuration_duration)
    expect((await firelight_vault.currentPeriod()) > poisonedPeriod).to.eq(true)

    // Attempt to claim: reverts on stale isWithdrawClaimed[poisonedPeriod][alice] = true flag
    await expect(firelight_vault.connect(alice).claimWithdraw(poisonedPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(poisonedPeriod)

    // Recovery attempt: re-rescue also blocked by the same flag
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, treasury.address, [poisonedPeriod])
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(poisonedPeriod)
  })
})


```

add the above testcase to the existing test file `test/rescue.js` and run

The PoC reproduces the exact attack path using only the vault's public functions, real Hardhat time-travel, and the project's existing test fixtures (`deployVault`, `utils.mintAndApprove`, `config.period_configuration_duration`) — no storage manipulation, no bypassed modifiers.

### Proof of Concept Scenario

1. **Setup.** Alice deposits 6,000 units. `startPeriod = currentPeriod()`; the open bucket is `poisonedPeriod = startPeriod + 1`.
2. **First withdrawal.** Alice calls `withdraw(2,000, alice, alice)`. This credits `withdrawSharesOf[poisonedPeriod][alice] = 2,000`.
3. **Compliance blocklist.** `BLOCKLIST_ROLE` calls `addToBlocklist(alice)` — a routine sanctions/KYC action.
4. **Rescue.** `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, treasury, [poisonedPeriod])`. This moves the 2,000 balance to `treasury` and sets `isWithdrawClaimed[poisonedPeriod][alice] = true`. `withdrawalsOf(poisonedPeriod, alice)` now reads 0.
5. **Resolution.** `BLOCKLIST_ROLE` calls `removeFromBlocklist(alice)` — the screening hit is cleared; `isBlocklisted[alice] == false`. Still inside `startPeriod`.
6. **Second, unrelated withdrawal — the trigger.** Still in the same period, Alice calls `withdraw(1,000, alice, alice)`. `_requestWithdraw` computes `period = currentPeriod() + 1 = poisonedPeriod` again and credits `withdrawSharesOf[poisonedPeriod][alice] = 1,000`. No code path checks `isWithdrawClaimed` on this request path.
7. **Time advances** two full periods so `currentPeriod() > poisonedPeriod`, making the period eligible for claims.
8. **Loss confirmed.** Alice calls `claimWithdraw(poisonedPeriod)`. This reverts with `AlreadyClaimedPeriod(poisonedPeriod)` — despite Alice holding a genuine, unclaimed 1,000-unit balance for that period.
9. **Recovery confirmed impossible.** Operators re-blocklist Alice and attempt `rescueWithdrawFromBlocklisted(alice, treasury, [poisonedPeriod])` to sweep the stranded balance out. This also reverts with `AlreadyClaimedPeriod(poisonedPeriod)`, because the rescue's own guard checks the same permanently-poisoned flag.

**Result:** The 1,000-unit withdrawal is frozen inside the vault indefinitely. It is excluded from `totalAssets()` via `pendingWithdrawAssets`, and no function in the contract can move it, decrement the counter, or restore claimability.
