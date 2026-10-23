# 90493 sc high incorrect claim flag update in the rescuewithdrawfromblocklisted function leads to permanent freezing of restored users funds

**Submitted on Aug 25th 2026 at 09:48:45 UTC by @Oxhsn for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90493
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds for at least 24 hours

## Description

**Severity:** Medium (impact maps to *Permanent freezing of funds*; see Risk Breakdown for the severity-characterization analysis)

***

## Brief

`rescueWithdrawFromBlocklisted` moves a blocklisted user's pending withdrawal to a custodian address and marks the source slot with `isWithdrawClaimed[period][from] = true`. Everywhere else in the contract that flag means *"already paid out,"* so writing it during a rescue plants a lie in storage. If the user is later un-blocklisted, `removeFromBlocklist` is unconditional and cleans up no associated state, and requests a new withdrawal while the current period still resolves to the same target period, the fresh entitlement lands in the poisoned slot. Their claim then reverts forever, and the only administrative cleanup anyone would reach for (blocklist again, rescue again) is blocked by the very flag the rescue wrote.

Importantly, **every privileged action in this sequence is a normal, documented operation performed correctly**, blocklist, rescue, delist is the protocol's own sanctioned compliance playbook, executed with zero malice and zero authority abuse. The party harmed is an ordinary, fully unblocked user, and the loss completes through her own permissionless `withdraw` and `claimWithdraw` calls. This matters for scope: per Immunefi's published clarification of the privileged-action rule, the exclusion targets findings where privileged access *is the attack vector*; here the privileged events are organic protocol operations and the attack itself requires no privileged access at all.

***

## Details

### The actors, and what each one does wrong (answer: nothing)

| Actor                     | Privileged? | Action                                    | Within documented duties?                                 |
| ------------------------- | ----------- | ----------------------------------------- | --------------------------------------------------------- |
| Alice (victim)            | No          | deposits, withdraws twice, attempts claim | Ordinary user operations                                  |
| `BLOCKLIST_ROLE` operator | Yes         | blocklists Alice, later un-blocklists her | Standard compliance action and its standard reversal      |
| `RESCUER_ROLE` operator   | Yes         | rescues pending redemption to custody     | Standard playbook; attempted once more during remediation |
| Custody/beneficiary       | No          | claims rescued funds                      | Ordinary                                                  |

No operator misbehaves, gains authority, or bypasses any check. Every failure is deterministic contract logic downstream of correct usage.

### The state machine

The vault tracks withdrawal claims with one boolean per `(period, account)`:

```solidity
// FirelightVault.sol:794, claimWithdraw
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

Inside `claimWithdraw`, the bit is set only *after* assets have been transferred out (`:806`). Its meaning is therefore unambiguous: **this account consumed this payout**. The rescue path writes the identical bit for a different reason:

```solidity
// FirelightVault.sol:861-873, rescueWithdrawFromBlocklisted
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to]   += _withdrawOf;
withdrawSharesOf[periods[i]][from]  = 0;
isWithdrawClaimed[periods[i]][from] = true;   // ← "moved away" recorded as "consumed"
```

Three facts compose into permanent loss rather than a cosmetic inconsistency:

1. **One boolean, two meanings, never reconciled.** `claimWithdraw` writes *"paid"*; rescue writes *"relocated."* Downstream, only the first meaning is implemented.
2. **Blocking is reversible; rescue's side effect is not.** `removeFromBlocklist` (`FirelightVault.sol:560-563`) flips one storage bit and performs no cleanup of withdrawal slots or flags. Nothing anywhere clears `isWithdrawClaimed`.
3. **New requests ignore the flag entirely.** `_requestWithdraw` resolves the target period as `currentPeriod() + 1` (`FirelightVault.sol:1008`) and credits `withdrawSharesOf[period][receiver] += sharesWithdraw` (`:1017`) without ever reading `isWithdrawClaimed`. Because both requests resolve to the *same absolute period number* whenever they happen inside the same period, the fresh entitlement inherits the poisoned slot.

### The sequence (matches the attached PoC transaction-for-transaction)

Configuration used below and in the PoC: 1-week periods (configurable; the MVP ships 1-day, feature-complete docs describe \~30-day), fXRP 6-decimal asset.

1. Alice deposits 10,000 fXRP. Unremarkable.
2. **Period C.** Alice calls `withdraw(5,000 fXRP)` → queued for period C+1. Her remaining \~5,000 position stays as shares.
3. **Period C.** `BLOCKLIST_ROLE` blocklists Alice (e.g., a screening vendor flags her address). She can no longer transfer, deposit, or claim.
4. **Period C.** `RESCUER_ROLE` executes the standard playbook, settle her pending redemption to a custody address (`rescueWithdrawFromBlocklisted(Alice, Custody, [C+1])`). The moved funds are safe; Custody will claim them normally in step 7. Side effect under test: `isWithdrawClaimed[C+1][Alice] = true`, and note the rescue deliberately does **not** touch her share balance, sweeping shares is the separate, optional `rescueSharesFromBlocklisted`. Partial remediation (pending redemption only) is therefore the operationally natural choice, leaving a restored user holding legitimate shares.
5. **Still period C.** The listing proves erroneous (or sanctions change). `BLOCKLIST_ROLE` removes Alice. Restoration is unconditional: no flag reset, no slot cleanup, no documented constraint that she abstain until the next period.
6. **Still period C.** Alice, wanting out entirely, calls `withdraw(5,000 fXRP)` on her untouched balance. All modifiers pass (she is unblocked); her shares burn via `_update(owner, address(0), shares)` (`:1025`); the assets are booked into `withdrawAssets[C+1]` and `pendingWithdrawAssets`, excluded from earnable supply (`totalAssets()` subtracts pending, `:427-429`), and credited into slot `C+1` next to the stale flag.
7. Two periods pass; `currentPeriod()` reaches C+2, which is when `claimWithdraw(C+1)` becomes callable (`:791` requires `period < currentPeriod()`).
8. Custody claims the **rescued** funds successfully. Rescued money was never at risk, proving the loss is exclusively the *post-restoration* entitlement.
9. Alice calls `claimWithdraw(C+1)` for her own funds. The gate at `:794` reads the poisoned flag before any balance math: `AlreadyClaimedPeriod(C+1)`. Her burned shares' backing sits frozen.
10. Operations attempt the obvious remediation, blocklist Alice again, rescue her fresh entitlement to a clean address. It reverts at `:864`: the guard checks `isWithdrawClaimed[periods[i]][from]` first, and that flag **is** the damage. The repair tool is blinded by its own earlier write.

Net result: Alice's second 5,000 fXRP entitlement is stranded inside `pendingWithdrawAssets` indefinitely. There is no sweeper, no force-refund, no flag-reset entrypoint, recovery requires deploying new contract code.

***

## Impact

* **What is lost:** the entire post-restoration redemption of a restored user, permanently. The PoC strands 5,000 fXRP (5,000000000 units, logged and asserted); the amount is bounded only by the user's position, there is no cap tying the second entitlement to the first rescue.
* **Where it sits:** inside the vault's `pendingWithdrawAssets`, excluded from `totalAssets()`, owed to no reachable party. Vault-level accounting stays internally consistent, this is pure stranding, not insolvency, which is also why it is silent: the re-request emits normal events, and nothing signals trouble until the user independently attempts a claim days later.
* **Scale context:** approximately **60.16M FXRP** of real predeposit capital lives in the Phase-1 proxy (Macro's July 2026 on-chain verification) and migrates into exactly these withdrawal semantics. Every staker's exit transits `withdrawSharesOf` slots that this collision can poison, and sanctions-style block/delist/relist cycles are precisely the workflow this tooling was built for.
* **Window:** the re-request must land while `currentPeriod()` still equals the original request period, 24 hours at MVP's 1-day periods, up to \~30 days at the documented feature-complete cadence. Erroneous-listing corrections and sanctions lifts mid-period are routine operational events, not exotic states.
* **In-scope impact mapping:** *Permanent freezing of funds*. See Risk Breakdown for the honest severity-characterization discussion, including the temporary-freeze counterfactual.

***

## Risk Breakdown

### Admissibility, the privileged-action exclusion does not attach

Immunefi's clarified position on this program's privileged-actor rule draws the line at whether **privileged access is the attack vector**, i.e., whether an attacker must be, or must compromise, a privileged role for the harm to execute. Quoting the operative test: *"If the operational event happens on its own in the ordinary course of the protocol running, and an unprivileged user is the one who triggers the harm, then the attack does not require privileged access and the exclusion does not apply."*

That is precisely this shape:

* The blocklist/rescue/delist sequence is the documented compliance workflow executing normally, no role acts maliciously, exceeds its grant, or chooses an unsafe value;
* The harm is triggered and completed by an **unprivileged** party (Alice's own permissionless withdrawals and failed claim);
* The loss arises deterministically from contract state-transition logic (one bit, two meanings), not from any operator decision;

so the privileged event is *setup*, not exploit, and the report should be assessed on the merits of the bug. Note also that we do not need to invoke the "bypasses an intended protocol constraint" exception at all, per the same clarification, that exception addresses roles exceeding authority, whereas here the exclusion simply never attaches. (Independently, the collision does violate an intended constraint, every positive `withdrawSharesOf[period][account]` should have a reachable claim path, but that is reinforcement, not load-bearing.)

### Severity characterization, and why "an upgrade could fix it" is not an automatic downgrade

Candidly stated: because all contracts sit behind proxies, an upgrade adding a sweep/reset function could eventually recover stranded funds. Under Immunefi's stated standards this cuts both ways:

* Recoverability-through-upgrade can legitimately recharacterize impact as *temporary* rather than *permanent* freeze, that is the impact definition working, and we acknowledge it.
* But mitigation-based downgrades require the mitigation to be **feasible, conventional, and documented in advance**. No such emergency mechanism exists in these contracts, no operational runbook for this failure is documented anywhere in the program materials or prior audit, and the fix is not a conventional action, it is purpose-built new code addressing this specific defect. Under the feasibility-limitation standard, its mere availability cannot suppress severity.

Even under the least favorable characterization (temporary freeze pending bespoke remediation), the freeze duration is indefinite-until-intervention and comfortably satisfies *Temporary freezing of funds for at least 24 hours*. We submit this as Medium overall: high-tier impact class, likelihood tempered by the privileged-workflow precondition and the same-period timing window.

### Exploitation difficulty summary

* Requires `BLOCKLIST_ROLE` + `RESCUER_ROLE` performing their **documented** workflow, followed by a documented unconditional restore, all benign, none discretionary.
* Timing condition: re-request within the same period as the original request. Trivially satisfiable given period durations.
* Zero detection signals: the re-request succeeds and emits normal events; the revert surfaces only when the user attempts the claim.
* Zero recovery paths on-chain, including administrative (demonstrated, PoC step 9).

***

## Recommendation

Preferred, stop writing the flag in the rescue path. Relocation is not consumption, and removing the write keeps repeat rescues safe anyway: they already revert on the zero-balance check (`NoWithdrawalAmount`, `:866`) because `withdrawSharesOf[from]` is cleared first.

```solidity
withdrawSharesOf[periods[i]][to]   += _withdrawOf;
withdrawSharesOf[periods[i]][from]  = 0;
// remove: isWithdrawClaimed[periods[i]][from] = true;
```

If operators need to distinguish rescued slots historically, use a dedicated mapping (e.g., `isWithdrawRescued`) instead of overloading the claim flag.

Alternatives:

* Reset `isWithdrawClaimed[period][account] = false` inside `removeFromBlocklist(account)`, restoring a user should restore all user state (requires tracking open periods per account).
* Derive claim eligibility from live entitlement rather than a sticky bit: allow the claim iff `withdrawSharesOf[period][account] > 0 && !claimed[period][account]`, with `claimed` written only by `claimWithdraw`. Cleanest invariant, largest refactor.

Relying on documented operator discipline ("never delist within the same period") is not an acceptable mitigation: the constraint appears nowhere in the protocol documentation, `removeFromBlocklist` is specified as unconditional, and it leaves a silent, unrecoverable fund-loss trap armed against the restored user.

***

## References

* `contracts/core/FirelightVault.sol:845-876`, `rescueWithdrawFromBlocklisted` (poison write `:870`; self-blocking guard `:864`)
* `contracts/core/FirelightVault.sol:788-811`, `claimWithdraw` (flag gate `:794`; flag set after payout `:806`)
* `contracts/core/FirelightVault.sol:994-1028`, `_requestWithdraw` (target period `:1008`; unguarded re-credit `:1017`; share burn `:1025`)
* `contracts/core/FirelightVault.sol:560-563`, unconditional `removeFromBlocklist`
* `contracts/core/FirelightVault.sol:550-554` / `:569-582`, blocklist management surface
* `contracts/core/FirelightVault.sol:427-429`, `totalAssets()` excludes pending withdrawals (stranded funds outside earnable supply)

***

## Proof of Concept

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { deployVault } = require('./setup/fixtures')

const DEPOSIT = ethers.parseUnits('10000', 6)
const WITHDRAW = ethers.parseUnits('5000', 6)

describe('HS-004 PoC — rescue flag strands same-period re-requested withdrawal', function () {
  async function fixture () {
    return deployVault({}) // fXRP 6 decimals, 50k deposit limit, 1-week periods
  }

  it('rescued flag permanently strands a legitimate re-requested claim, with no recovery path', async function () {
    const { firelight_vault, rescuer, blocklister, users, utils, config } =
      await loadFixture(fixture)

    const alice = users[0]
    const beneficiary = users[1]
    const freshAddress = users[2]

    // Step 0. Fund and seat Alice (ordinary user).
    await utils.mintAndApprove(DEPOSIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)

    const periodC = await firelight_vault.currentPeriod()
    const claimPeriod = periodC + 1n

    // Step 1. Period C: Alice queues a withdrawal -> claimable at C+1.
    await firelight_vault.connect(alice).withdraw(WITHDRAW, alice.address, alice.address)
    expect(await firelight_vault.withdrawalsOf(claimPeriod, alice.address)).to.be.gt(0)

    // Step 2. BLOCKLIST_ROLE blocklists Alice (standard compliance action).
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)

    // Step 3. RESCUER_ROLE settles her pending redemption to custody (standard playbook).
    // Side effect under test: isWithdrawClaimed[C+1][Alice] = true.
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, beneficiary.address, [claimPeriod])
    expect(await firelight_vault.withdrawalsOf(claimPeriod, alice.address)).to.equal(0)

    // Step 5. Still period C: BLOCKLIST_ROLE restores Alice (unconditional, no cleanup).
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)

    // Step 6. Still period C: Alice exits her remaining position. _requestWithdraw
    // re-credits the poisoned C+1 slot without inspecting the flag.
    await firelight_vault.connect(alice).withdraw(WITHDRAW, alice.address, alice.address)
    expect(await firelight_vault.withdrawalsOf(claimPeriod, alice.address)).to.be.gt(0)

    // Step 7 (timing). Advance two periods so claimPeriod (C+1) matures:
    // claimWithdraw requires period < currentPeriod(), i.e. currentPeriod >= C+2.
    await time.increase(BigInt(config.period_configuration_duration) * 2n)

    // Step 7 (cont.). Custody claims the RESCUED funds successfully.
    await firelight_vault.connect(beneficiary).claimWithdraw(claimPeriod)

    // Step 8. Alice claims her own fresh entitlement -> PERMANENT revert.
    await expect(
      firelight_vault.connect(alice).claimWithdraw(claimPeriod)
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(claimPeriod)

    // Step 9. No administrative recovery: re-blocklist + re-rescue hits the SAME flag.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, freshAddress, [claimPeriod])
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(claimPeriod)
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)

    console.log('        FUNDS FROZEN (assets):', (await firelight_vault.withdrawalsOf(claimPeriod, alice.address)).toString())
    expect(await firelight_vault.withdrawalsOf(claimPeriod, alice.address)).to.be.gt(0)
  })
})

```
