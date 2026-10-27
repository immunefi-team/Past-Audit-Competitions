# 88619 sc high stale iswithdrawclaimed flag set by rescuewithdrawfromblocklisted permanently freezes a later legitimate withdrawal by the same account burning the user s shares with no recove&#x20;

**Submitted on Aug 14th 2026 at 07:26:19 UTC by @enowk for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88619
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/IncidentManager.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` as a "this account is no longer entitled" marker, but `claimWithdraw` reads that same flag as "this account has already been paid". The flag is keyed by `(period, account)` and is never cleared, while `_requestWithdraw` neither consults nor resets it. If a rescued account is later removed from the blocklist — a routine, intended compliance action — and makes a **new, entirely legitimate** withdrawal request that lands in the same period bucket, its shares are burned on request and `claimWithdraw(period)` reverts `AlreadyClaimedPeriod` forever. The privileged rescue path cannot recover the funds either, because it reverts on the same flag. On mainnet this permanently freezes the user's assets with their shares already destroyed, and permanently inflates `pendingWithdrawAssets`, which suppresses `totalAssets()` — and therefore cover capacity and the payout exposure cap — for the remaining life of the vault.

## Vulnerability Details

### The flag carries two incompatible meanings

`isWithdrawClaimed` is declared once in [`FirelightVaultStorage.sol#L71`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVaultStorage.sol#L71):

```solidity
mapping(uint256 period => mapping(address account => bool value)) public isWithdrawClaimed;
```

It has two writers that disagree about what it means:

1. **`claimWithdraw`** — "this account has already been paid for this period."
2. **`rescueWithdrawFromBlocklisted`** — "this account's entitlement was moved elsewhere."

`claimWithdraw` ([`FirelightVault.sol#L788-L811`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L811)) treats any set flag as meaning (1):

```solidity
function claimWithdraw(uint256 period) external ... returns (uint256 assets) {
    if (period >= currentPeriod()) revert InvalidPeriod();

    address sender = _msgSender();
    if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // <-- hard block
    ...
    if (assets == 0) revert NoWithdrawalAmount(period);

    pendingWithdrawAssets -= assets;
    isWithdrawClaimed[period][sender] = true;
```

`rescueWithdrawFromBlocklisted` ([`FirelightVault.sol#L861-L873`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L861-L873)) writes it with meaning (2):

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // <-- blocks recovery
    if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;   // <-- poisons the (period, account) pair forever
}
```

### The flag write is redundant, so it buys no safety in exchange for the harm

Both protections the flag might be providing in the rescue path already exist independently:

* Double-rescue is already prevented by `if (_withdrawOf == 0) revert NoWithdrawalAmount(...)`, because the rescue zeroes `withdrawSharesOf[periods[i]][from]` on line 869.
* A post-rescue claim is already prevented by `claimWithdraw`'s own `if (assets == 0) revert NoWithdrawalAmount(period)`, since the rescued account's `withdrawSharesOf` is now zero.

The flag write on line 870 therefore adds nothing except a permanent, un-clearable poison on the `(period, account)` pair.

### `_requestWithdraw` never reconciles with the flag

[`FirelightVault.sol#L994-L1028`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028) accepts the new request unconditionally and burns the shares:

```solidity
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor);
withdrawAssets[period]              += assets;
withdrawShares[period]              += sharesWithdraw;
withdrawSharesOf[period][receiver]  += sharesWithdraw;   // booked into the poisoned bucket

pendingWithdrawAssets += assets;
...
_update(owner, address(0), shares);                      // shares burned, irreversibly
```

There is no read of `isWithdrawClaimed` anywhere in `_requestWithdraw`, and no path anywhere in the contract that ever sets the flag back to `false`. The user gets no revert and no warning; the transaction succeeds.

### Why only the rescue writer can reach this state

This is the crux, and it is what distinguishes this from the previously-disclosed issue discussed below.

A bucket is open for new requests only while `period == currentPeriod() + 1` (line 1008). `claimWithdraw` refuses to run unless `period < currentPeriod()` (line 791). Because period numbering is monotonic at this commit, **the `claimWithdraw` writer can never set the flag on a bucket that is still open for requests** — by the time it can write, the bucket has been closed to new requests for at least one full period. Reaching the freeze through `claimWithdraw` therefore requires the period counter to move *backwards*, which is exactly the (now-fixed) precondition of OpenZeppelin M-02.

`rescueWithdrawFromBlocklisted` has **no period bound at all**. Its only constraint is `withdrawSharesOf[periods[i]][from] != 0`, and the one bucket that is guaranteed to be non-empty and simultaneously still open is `currentPeriod() + 1`. The rescue writer can therefore set the flag on a *live* bucket, which the claim writer structurally cannot. That is a distinct root cause with a distinct reachability precondition, and it is unaffected by the fix that closed M-02.

### Attack / failure sequence

All privileged roles behave exactly as designed throughout. No role acts maliciously or beyond its documented authority.

{% stepper %}
{% step %}

## Preconditions

Alice holds vault shares and requests a withdrawal, which is booked to period `P = currentPeriod() + 1`. She retains further shares (a partial withdrawal, the ordinary case).
{% endstep %}

{% step %}

## Blocklist Alice

`BLOCKLIST_ROLE` blocklists Alice — a routine compliance action (sanctions screening hit, suspected key compromise, exchange-flagged deposit).
{% endstep %}

{% step %}

## Rescue the pending withdrawal

`RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(Alice, Carol, [P])`. Alice's pending entitlement moves to Carol, and `isWithdrawClaimed[P][Alice]` is set to `true`. This is the intended compliance remedy working correctly.
{% endstep %}

{% step %}

## Clear Alice from the blocklist

Alice is cleared and `BLOCKLIST_ROLE` removes her from the blocklist — still inside period `P-1`, so `currentPeriod() + 1` is still `P`.
{% endstep %}

{% step %}

## Make a new withdrawal request

Alice calls `withdraw()` for her remaining shares. The `notBlocklisted(receiver)` guard added for OZ M-03 now passes, since she is cleared. Her shares are burned and the assets are booked to `withdrawSharesOf[P][Alice]`. **No revert, no warning.**
{% endstep %}

{% step %}

## Let period `P` mature

**`claimWithdraw(P)` reverts `AlreadyClaimedPeriod(P)` — permanently.**
{% endstep %}

{% step %}

## Attempt recovery

**No recovery exists.** Re-blocklisting Alice does not help: the rescue's own guard `if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(...)` (line 864) fires on the identical flag. The shares are already burned, and the assets remain counted in `withdrawAssets[P]` and `pendingWithdrawAssets` forever.
{% endstep %}
{% endstepper %}

Step 4 is the only step that needs a plausibility argument, and there are two ordinary operational narratives for it:

* **Key compromise.** Alice reports a compromised key. The vault blocklists the old address and rescues the pending withdrawal to her new address (`to` must be non-blocklisted, so rescuing back to Alice's own blocklisted address is impossible by construction). The old address is then cleared once the incident is closed.
* **False-positive screening.** An automated compliance hold is opened and actioned, then cleared on review. Because `to` cannot be the blocklisted address, the rescue necessarily routes funds to a third address even when the hold was later found to be unwarranted.

The whole sequence must fit inside one period. The docs state periods are **currently 1 day** and extend to **30-day cover periods at the feature-complete launch**, at which point a compliance hold that is opened, actioned and cleared within a single period is ordinary operations rather than an exotic prerequisite.

### Prior art: why this is not OpenZeppelin M-02

The OpenZeppelin Phase 1 audit (2025-07-21) reported **M-02 "Period Duration Cannot Be Changed"**, which reaches a similar-looking end state through a different mechanism. I raise it here proactively rather than leave it for triage.

M-02's body reads:

> "If a user had requested a withdrawal in a previous period and, after the `periodDuration` change, the recalculated `currentPeriod` still matches the earlier request period, the user may attempt to withdraw again. In such cases, their shares would be burned during the call to `_requestWithdraw`, but the subsequent call to `claimWithdraw` would revert due to the `isWithdrawClaimed` check."

Three reasons this report is a separate, still-live issue:

1. **OpenZeppelin diagnosed a period-configuration bug, not a flag bug.** Their recommendation is *"Consider keeping `periodDuration` constant throughout the lifetime of the vault to prevent inconsistencies such as the one discussed above."* They proposed no change to `isWithdrawClaimed`, `claimWithdraw`, or `_requestWithdraw`. The flag was the symptom carrier, not the diagnosis.
2. **Their fix is in place at this commit and is effective against their trigger.** M-02 is marked "Resolved in pull request #6 at commit 1433252". `_addPeriodConfiguration` now requires `newEpoch >= nextPeriodEnd()` plus epoch alignment, so period numbering is monotonic and `currentPeriod` can never rewind onto an earlier request period. **Test 3 of the attached PoC asserts this directly** — `addPeriodConfiguration(nextPeriodEnd() - 1, ...)` reverts `InvalidPeriodConfigurationEpoch`, and a legal future configuration never lowers the period number. OpenZeppelin's path is dead.
3. **The rescue path never touches period configuration at all,** and PR #6 did not modify `rescueWithdrawFromBlocklisted`. As shown in the reachability argument above, the rescue writer can poison a bucket that is still open for requests, which the claim writer structurally cannot. Test 1 of the PoC runs green against this exact commit.

For completeness, the following were also checked and none covers this issue:

* The 8 published Immunefi known issues — none concerns the blocklist or rescue paths.
* 0xMacro Phase 2 (`d0d3733`, 2026-07-29) — mentions the rescue only as a trust assumption ("BLOCKLIST\_ROLE with RESCUER\_ROLE … targeted confiscation power"), with no finding on it. Findings M-1, M-2, L-1…L-4, G-1 and I-1…I-11 are unrelated.
* OpenZeppelin M-03 ("Missing Blacklist Check Can Lock Funds") is the nearest neighbour, but concerns a blacklisted **receiver at request time**; its fix (the `notBlocklisted(receiver)` guard on `withdraw`/`redeem`) is present and is a *precondition* of this report, not a mitigation for it.
* Coinspect 2025-09-01 — frontend scope only.
* The Dec-2025 Immunefi Audit Competition summary reports **6 valid and 40 insight submissions from 329 reports, with zero findings at Medium severity or above** (top payout $2,546 of a $15,000 pool). The two published reports are unrelated (`_disableInitializers` and a `periodAtTimestamp` timestamp bug). A permanent freeze of user funds would not have been resolved as a Low.

### Recommended fix

Delete the redundant flag write on line 870:

```diff
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;
```

`withdrawSharesOf[period][from] = 0` already makes `claimWithdraw` revert `NoWithdrawalAmount`, and the `_withdrawOf == 0` guard already prevents a double rescue.

If the flag must be retained for accounting or event reasons, track rescues in a separate mapping (e.g. `isWithdrawRescued`) that `claimWithdraw` does **not** consult, and drop the `isWithdrawClaimed[...][from]` precondition on line 864 so that privileged recovery remains possible.

Either way, add a regression test asserting that an un-blocklisted account can still claim a withdrawal it requested *after* a rescue.

## Impact Details

**Selected in-scope impact: Permanent freezing of funds.**

**Direct loss, per occurrence.** The full asset amount of the affected user's post-rescue withdrawal request is permanently frozen. The loss is realised in two irreversible steps within the vault:

* The user's shares are burned by `_update(owner, address(0), shares)` at request time, so the claim on the vault is destroyed.
* The corresponding assets sit in `withdrawSharesOf[P][Alice]` and can never be paid out: `claimWithdraw` is blocked by the stale flag, and `rescueWithdrawFromBlocklisted` — the protocol's own designated remedy for stuck balances — reverts on the same flag.

In the attached PoC, a user who deposits 1,000 and has a 400 withdrawal rescued then loses the entire 300 of her next request. There is no cap on this: a user withdrawing her full remaining position in the affected period loses that entire position. Nothing in the contract bounds the amount, and the freeze applies per `(period, account)` pair, so every account the rescue path touches carries the latent poison for the remainder of that period.

**Secondary, permanent protocol-level harm.** `claimWithdraw` is the only path that decrements `pendingWithdrawAssets` for a user's own entitlement (line 805). Because the claim can never execute, the frozen amount stays in `pendingWithdrawAssets` for the life of the vault. Since

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() - pendingWithdrawAssets;
}
```

the frozen assets are subtracted from `totalAssets()` forever while physically remaining in the vault. They become dead capital: they can never be withdrawn by anyone, can never back cover, and are permanently excluded from both the cover-capacity computation in `CoverOrderAllocator._computeAvailableCapacity` (which reads `vault.totalAssetsAt(...)`) and the payout exposure cap in `payout` (which reads `totalAssetsAt(capturePeriodStart)`). Every occurrence permanently ratchets down the protocol's underwriting capacity. The PoC asserts this residue explicitly.

**Why this is "permanent" and not "temporary".** No sequence of calls on the deployed contract restores access. The one privileged function designed to unstick balances is blocked by the identical flag it set, and the flag has no clearing path anywhere in the contract. A proxy upgrade could of course introduce one, but that is true of essentially any state corruption behind an upgradeable proxy and is not an in-contract recovery path.

**Why this is not griefing.** There is no attacker and no profit motive — the rescue destination receives only the funds the rescue legitimately moved, and nobody gains from the freeze. The user's own funds are destroyed by a transaction the user sends herself, and every privileged role acts within its documented authority. The absence of a beneficiary does not reduce the loss; it means the loss is unrecoverable rather than recoverable from an attacker.

**Preconditions, stated honestly.** The sequence requires `BLOCKLIST_ROLE` and `RESCUER_ROLE` to perform their intended compliance operations and then clear the account within the same period, followed by a withdrawal the user initiates herself. It is gated by an operational sequence rather than by attacker capability, and the trusted-operator carve-out does not apply because no role exceeds its intended authority — this is the in-scope contract destroying in-scope user funds while all of its dependencies behave correctly.

**Severity note.** The program's in-scope impact table lists "Permanent freezing of funds" as Critical with no TVL qualifier, and this report is filed against that entry. For the program's own calibration: the blast radius is one user's withdrawal per occurrence rather than majority TVL, and protocol-owned funds are not put at risk, so a High rating would also be a defensible read.

## References

**Vulnerable code (commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, branch `v1_audit_ready`):**

* [`FirelightVault.sol#L870`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L870) — the flag write in `rescueWithdrawFromBlocklisted` (root cause)
* [`FirelightVault.sol#L864`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L864) — the rescue guard that blocks recovery on the same flag
* [`FirelightVault.sol#L794`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L794) — the `claimWithdraw` check that reads the flag with the other meaning
* [`FirelightVault.sol#L791`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L791) — `period >= currentPeriod()` bound that makes the claim writer unable to reach an open bucket
* [`FirelightVault.sol#L994-L1028`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028) — `_requestWithdraw`, which never consults or clears the flag
* [`FirelightVault.sol#L1008`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1008) — `period = currentPeriod() + 1`, defining the open bucket
* [`FirelightVault.sol#L427-L429`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L427-L429) — `totalAssets()` subtracting `pendingWithdrawAssets`
* [`FirelightVault.sol#L763-L765`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L763-L765) — the `notBlocklisted` guards on `withdraw` (the OZ M-03 fix, a precondition of step 5)
* [`FirelightVaultStorage.sol#L71`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVaultStorage.sol#L71) — the `isWithdrawClaimed` slot declaration

**Prior audits referenced:**

* OpenZeppelin, *Sentora Firelight Vault Audit*, 2025-07-21 — M-02 "Period Duration Cannot Be Changed" (resolved in PR #6 at commit `1433252`) and M-03 "Missing Blacklist Check Can Lock Funds" (resolved in PR #5 at commit `68a9dd2`). Published at Firelight's `/resources/audits-and-security`.
* 0xMacro, *Firelight A-1*, 2026-07-29 — the BLOCKLIST\_ROLE/RESCUER\_ROLE combination is documented there as a trust assumption only, with no finding on the rescue path.
* Immunefi, *Firelight Audit Competition Summary Report*, December 2025.

**Protocol documentation:**

* Core Concepts — Periods and Coverage Cycles: periods are currently 1 day and extend to 30-day cover periods at the feature-complete launch.
* For Stakers — Deployments and Withdrawals: the unstaking window is the remainder of the current period plus one full period.

## Proof of Concept

A runnable Hardhat test is provided below. It executes against the in-scope repository at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (branch `v1_audit_ready`) using the repository's own `test/setup/fixtures.js`, with no modifications to any contract.

### How to run

Save the file as `test/zz-audit-poc-rescue-freeze.js` in the `audit-comp-firelight` repository and run:

```bash
npx hardhat test test/zz-audit-poc-rescue-freeze.js
```

### Actual output

```
  AUDIT POC: rescue leaves a permanent claim block
      PERMANENTLY FROZEN: 300.0 assets, shares already burned
    ✔ freezes a later withdrawal in the same period, with no recovery path (13401ms)
    ✔ CONTROL: an identical user who was never rescued claims fine (163ms)
    ✔ CONFIRMS the OpenZeppelin M-02 trigger is dead — period numbering cannot rewind (117ms)


  3 passing (14s)
```

Test 1 demonstrates the vulnerability. Test 2 is a control proving the freeze is caused by the rescue's flag write rather than by the test setup. Test 3 asserts that OpenZeppelin M-02's trigger is closed at this commit, which is what makes the rescue trigger a distinct and still-live root cause.

### Step-by-step explanation

All amounts use the vault asset's 6 decimals. `P` denotes the withdrawal bucket `currentPeriod() + 1`.

{% stepper %}
{% step %}

## Alice requests a withdrawal of 400

`withdraw(400, alice, alice)` routes into `_requestWithdraw`, which books the request to `P = currentPeriod() + 1` and burns the corresponding shares.

*Assertion:* `withdrawSharesOf(P, alice) > 0`.
{% endstep %}

{% step %}

## Blocklist Alice

`addToBlocklist(alice)` — a routine compliance action, working exactly as designed.
{% endstep %}

{% step %}

## Rescue the pending withdrawal to Carol

`rescueWithdrawFromBlocklisted(alice, carol, [P])` moves Alice's entitlement to Carol. This is the intended compliance remedy and it succeeds.

*Assertions:* `isWithdrawClaimed(P, alice) == true` — the flag is now set even though Alice never claimed anything — and `withdrawSharesOf(P, alice) == 0`.
{% endstep %}

{% step %}

## Clear Alice while still inside period `P-1`

`removeFromBlocklist(alice)`.

*Assertion:* `currentPeriod() + 1 == P` — the same bucket is still the open one, so a new request will land in the poisoned bucket.
{% endstep %}

{% step %}

## Alice makes a new withdrawal request of 300

`withdraw(300, alice, alice)`. The `notBlocklisted(_msgSender() / owner / receiver)` guards all pass because she is cleared. The call **succeeds with no revert and no warning**.

*Assertions:* `balanceOf(alice)` strictly decreased — her shares were burned — and `withdrawalsOf(P, alice) > 0`, so the vault now owes her assets in bucket `P`.
{% endstep %}

{% step %}

## Let period `P` mature

Time is advanced past two period ends so that `currentPeriod() > P` and the bucket becomes claimable.
{% endstep %}

{% step %}

## The claim is permanently blocked

`claimWithdraw(P)` reverts `AlreadyClaimedPeriod(P)`, because `isWithdrawClaimed[P][alice]` is still set from the rescue. Alice's shares are already burned, so her assets are frozen.
{% endstep %}

{% step %}

## Attempt recovery

Re-blocklisting Alice and retrying the rescue does not help: `rescueWithdrawFromBlocklisted(alice, carol, [P])` reverts with the same `AlreadyClaimedPeriod` error, because line 864 guards on the identical flag. The protocol's own designated remedy for stuck balances is blocked by the flag it set itself.

*Assertion:* `withdrawalsOf(P, alice)` is unchanged — still owed, still unclaimable.
{% endstep %}

{% step %}

## Confirm protocol-level residue

The frozen assets remain counted in `pendingWithdrawAssets` forever, because `claimWithdraw` is the only path that would decrement them. Since `totalAssets()` subtracts `pendingWithdrawAssets`, the assets are permanently excluded from cover capacity and the payout exposure cap while still physically sitting in the vault.

*Assertion:* `pendingWithdrawAssets() >= owed`.
{% endstep %}
{% endstepper %}

**Result: 300 assets permanently frozen, shares already burned, no recovery path.**

### Test source

```javascript
// AUDIT POC — permanent freeze of a rescued account's future withdrawal.
//
// rescueWithdrawFromBlocklisted() sets isWithdrawClaimed[period][from] = true as a
// "no longer entitled" marker. That flag is per-(period, account) and is never cleared.
// If the account is later un-blocklisted and requests a NEW withdrawal that lands in the
// SAME period bucket, claimWithdraw(period) reverts AlreadyClaimedPeriod forever — and the
// rescue path cannot recover it either, because it reverts on the same flag.
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')

const DECIMALS = 6
const unit = (n) => ethers.parseUnits(String(n), DECIMALS)

describe('AUDIT POC: rescue leaves a permanent claim block', function () {
  this.timeout(300000)

  it('freezes a later withdrawal in the same period, with no recovery path', async () => {
    const { firelight_vault: vault, token_contract: token, utils, users,
            blocklister, rescuer, deployer } = await deployVault()

    const alice = users[0]
    const carol = users[1] // rescue destination
    await utils.mintAndApprove(unit(10000), alice)

    // --- Alice deposits and requests a withdrawal (lands in bucket P = current + 1).
    await vault.connect(alice).deposit(unit(1000), alice.address)
    const P = (await vault.currentPeriod()) + 1n
    await vault.connect(alice).withdraw(unit(400), alice.address, alice.address)
    expect(await vault.withdrawSharesOf(P, alice.address)).to.be.gt(0n)

    // --- Compliance flow: blocklist Alice, rescue her pending withdrawal to Carol.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, carol.address, [P])

    // The rescue marks Alice as "claimed" for period P even though she never claimed.
    expect(await vault.isWithdrawClaimed(P, alice.address),
      'rescue sets the claimed flag on the rescued account').to.equal(true)
    expect(await vault.withdrawSharesOf(P, alice.address)).to.equal(0n)

    // --- Alice is cleared and returns to normal use, still inside period P-1.
    await vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await vault.currentPeriod() + 1n, 'still the same target bucket').to.equal(P)

    // --- She makes a brand-new, entirely legitimate withdrawal request.
    const sharesBefore = await vault.balanceOf(alice.address)
    await vault.connect(alice).withdraw(unit(300), alice.address, alice.address)
    const sharesAfter = await vault.balanceOf(alice.address)

    // Her shares are burned and the assets are booked to her in bucket P.
    expect(sharesAfter, 'shares were burned').to.be.lt(sharesBefore)
    const owed = await vault.withdrawalsOf(P, alice.address)
    expect(owed, 'vault owes Alice for the new request').to.be.gt(0n)

    // --- Period P matures.
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.be.gt(P)

    // IMPACT 1: the claim is permanently blocked by the stale flag.
    await expect(vault.connect(alice).claimWithdraw(P))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')

    // IMPACT 2: the rescue path cannot recover it either — it reverts on the same flag,
    // so even re-blocklisting Alice does not let the rescuer move the funds out.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, carol.address, [P]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')

    // The assets remain counted as pending forever: burned shares, unclaimable assets.
    expect(await vault.withdrawalsOf(P, alice.address), 'still owed, still unclaimable').to.equal(owed)

    // IMPACT 3: the assets are still physically in the vault and still counted as pending,
    // permanently suppressing totalAssets() -> cover capacity and the payout exposure cap.
    expect(await vault.pendingWithdrawAssets(), 'frozen assets inflate pendingWithdrawAssets forever')
      .to.be.gte(owed)

    console.log(`      PERMANENTLY FROZEN: ${ethers.formatUnits(owed, DECIMALS)} assets, shares already burned`)
  })

  it('CONTROL: an identical user who was never rescued claims fine', async () => {
    // Proves the freeze is caused by the rescue's flag write, not by the test setup.
    const { firelight_vault: vault, utils, users } = await deployVault()
    const bob = users[2]
    await utils.mintAndApprove(unit(10000), bob)

    await vault.connect(bob).deposit(unit(1000), bob.address)
    const P = (await vault.currentPeriod()) + 1n
    await vault.connect(bob).withdraw(unit(400), bob.address, bob.address)
    // Same "two requests into the same bucket" shape as Alice, minus the rescue.
    await vault.connect(bob).withdraw(unit(300), bob.address, bob.address)

    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)

    await expect(vault.connect(bob).claimWithdraw(P), 'unrescued user must be able to claim')
      .to.not.be.reverted
    expect(await vault.isWithdrawClaimed(P, bob.address)).to.equal(true)
  })

  it('CONFIRMS the OpenZeppelin M-02 trigger is dead — period numbering cannot rewind', async () => {
    // OZ M-02 reached the same isWithdrawClaimed freeze via a periodDuration change.
    // PR #6 closed it: a new configuration must start at or after nextPeriodEnd(), so
    // currentPeriod is monotonic and can never fall back onto an earlier request period.
    // This is what makes the rescue trigger a distinct, still-live root cause.
    const { firelight_vault: vault, period_configuration_updater: pcu } = await deployVault()

    const before = await vault.currentPeriod()
    const nextEnd = await vault.nextPeriodEnd()

    // Any epoch before nextPeriodEnd() is rejected outright.
    await expect(vault.connect(pcu).addPeriodConfiguration(nextEnd - 1n, 7n * 86400n))
      .to.be.revertedWithCustomError(vault, 'InvalidPeriodConfigurationEpoch')

    // A legal future configuration never rewinds the current period number.
    await vault.connect(pcu).addPeriodConfiguration(nextEnd, 14n * 86400n)
    expect(await vault.currentPeriod(), 'period number must not rewind').to.be.gte(before)
  })
})
```
