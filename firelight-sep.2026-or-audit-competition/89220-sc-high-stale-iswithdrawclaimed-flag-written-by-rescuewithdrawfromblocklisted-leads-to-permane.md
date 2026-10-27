# 89220 sc high stale iswithdrawclaimed flag written by rescuewithdrawfromblocklisted leads to permanent freezing of funds capital deposited after the compliance action has ended is destroyed&#x20;

**Submitted on Aug 17th 2026 at 21:53:42 UTC by @Volga0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89220
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### ═══ Brief/Intro ═══

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal claim to a destination, and writes `isWithdrawClaimed[period][from] = true` (`FirelightVault.sol:870`) against the account the claim was taken **from**. `isWithdrawClaimed` has five sites in the contract — three reads (`:794`, `:864`, `:865`) and two writes, `:806` and `:870`, **both to `true`**. No line anywhere in `FirelightVault.sol` assigns `false` to it, so nothing ever clears it.

Once the account is rehabilitated, any withdrawal it requests that lands on the same period burns its shares (`:1025`) and records a claim (`:1017`), and `claimWithdraw` then reverts `AlreadyClaimedPeriod` (`:794`) — permanently. Measured: **1 000 000 000 000 of capital deposited after the compliance action had ended, on an account holding nothing, destroyed with no recovery path for anyone** — and the same flag destroys 500 000 000 000 of the original position by the same route.

**The remediation is one deleted line, and the project's own suite already agrees with it.** Delete `FirelightVault.sol:870` and run `npx hardhat test`: **448 passing, 0 failing, unchanged** — not one of the project's own tests asserts that a rescue marks the rescued account as having claimed. The runtime loses 21 bytes on a contract with 248 left under EIP-170, and `aderyn` returns an identical finding set. That check needs no Foundry, no Gist, and no acceptance of anything below it.

**The bound.** A withdrawal request lands on `currentPeriod() + 1` (`:1008`) and the period counter only advances, so the poisoned period is reachable only while it is still the landing period: request, blocklist, rescue, rehabilitation and the new request must all fall inside that one window. Firelight's staker documentation puts a period at *"currently 1 day"*. What the bound limits is how much can be caught — not what happens to it. Everything requested inside that window is burned and unclaimable **for ever**, by anyone.

### ═══ Vulnerability Details ═══

### The flag's own documentation says what it should mean

`FirelightVaultStorage.sol:70-71`, at the declaration site:

```solidity
/// @notice Indicates whether an account has claimed their withdrawal for a given period.
mapping(uint256 period => mapping(address account => bool value)) public isWithdrawClaimed;
```

The subject of *"has claimed"* is **the account**. The read site says the same thing in its own NatSpec, `claimWithdraw` (`:781-784`):

```solidity
* @notice Claims a pending withdrawal for a given period.
* Transfers the corresponding assets to the caller if not already claimed.
* Reverts if the withdrawal has already been claimed or if no withdrawal amount is available
```

**And the project's own rescue suite makes it three.** In `test/rescue.js`, the "already claimed" state is always produced by a real claim: `users[2]` calls `claimWithdraw(1)` at `:114` and the contract emits `CompleteWithdraw`. That is the state then fed to both guards — *"reverts if rescuer tries to rescue pending withdrawals **to a user who has already claimed that period**"* (`:133`) and *"… **if one or more periods were already claimed**"* (`:152`). **The suite never produces that state with a rescue.** It also never rehabilitates: zero occurrences of `removeFromBlocklist` in the file, against three of `addToBlocklist`. It runs the rescue to completion at `:157`, follows the destination as it collects at `:167-181`, and never asks what the flag does to the account it was written against.

After a rescue, that account has claimed nothing — the assets went to the destination, and the destination can and does collect them (measured below). Yet the account carries the flag for ever, and the contract reads it back as if it had been paid.

### Details

**Where it is written.** `rescueWithdrawFromBlocklisted` (`:845-876`, `onlyRole(RESCUER_ROLE)`, `onlyBlocklisted(from)`, `notBlocklisted(to)`) runs this loop body:

```solidity
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // :864
if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);   // :865
if (_withdrawOf == 0)                    revert NoWithdrawalAmount(periods[i]);     // :866

withdrawSharesOf[periods[i]][to]   += _withdrawOf;                                  // :868
withdrawSharesOf[periods[i]][from]  = 0;                                            // :869
isWithdrawClaimed[periods[i]][from] = true;                                         // :870
```

`:869` already empties the account's claim, and `:866` already refuses a second rescue on an empty one. `:870` adds nothing to the rescue it completes — and it is the line that outlives it.

**Where it bites.** `claimWithdraw` (`:794`):

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

Between the two, `redeem` has already burned the shares at `:1025` and recorded the claim at `:1017`. The order is fatal: the burn is unconditional and precedes any check of the flag.

**Why the account comes back at all.** Rehabilitation is the normal end of a compliance hold — `removeFromBlocklist` exists precisely for it — and the returning account has every reason to withdraw. Note that the sibling guards `notBlocklisted(receiver)` on `redeem` (`:727`) and `withdraw` (`:765`) exists because of OpenZeppelin **M-03**, *"Missing Blacklist Check Can Lock Funds"*: the protocol has already been told once that funds routed to a blocklisted party become irretrievable through `claimWithdraw`. That remediation guarded the *receiver*. The flag written at `:870` reaches the same outcome through the *sender*.

### Why this is in scope, in the programme's own words

**The role exceeds the permission it was granted.** That is this report's ground, stated first so it is not mistaken for the other one. The rescue power itself is legitimate and is not questioned here, and nothing below argues that a missing check should be added. The argument is that a line which *is* there writes an effect the role's mandate does not cover, and that the function it belongs to does not need it.

The sequence requires `RESCUER_ROLE` and `BLOCKLIST_ROLE`, each calling a function it holds the key for. The Privileged actors rule therefore applies, and so does its exception:

> *"Any bug report that requires a privileged role holder to act within the permissions explicitly granted to that role is out of scope, **unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint**."*

**And the same section does not leave it at an exception — the very next bullet states it affirmatively:**

> *"There are no actors whose involvement is out of scope when they exceed their attributed privileges. Reports where a privileged role, admin, or operator **can exceed the privileges attributed to them remain in scope**."*

That is this report exactly. Here is the privilege, in the contract's own words (`:839-844`):

```solidity
* @notice Rescues pending withdrawals from a blocklisted address. Requires RESCUER_ROLE.
* @param from The blocklisted address.
* @param to The address to transfer the shares to. Must not be blocklisted.
* @param periods An array of periods to rescue.
```

Four lines, and every one is about moving a claim to a destination. **None of them marks the account the claim was taken from.** `:870` writes a permanent bar on every future withdrawal on that period: an effect the grant never mentions, that the rescue does not need (`:869` has already completed it), that the role itself cannot lift (`:864` refuses it), and that disqualifies an account which — by both documented sites above — *has not claimed*.

**The clearest measure of the excess is that it escapes the role itself.** Once `:870` is written, `RESCUER_ROLE` can no longer act on that account for that period: a second rescue reverts `AlreadyClaimedPeriod` at `:864`, which reads the flag the role has just written — measured in `test_afterTheBite_noCallReachesTheStrandedAsset`, alongside four other callers who are refused too. An effect that its own author cannot undo, and that no other role can undo either, is not the exercise of a granted power. It is a consequence the grant does not contain.

**And the consequence reaches what the permission never could.** Capital deposited **after** the account was cleared — money that did not exist when the permission was exercised, over which no role was ever granted anything — is destroyed by the same flag. Measured: **1 000 000 000 000**, in `test_freshCapitalAfterRehabilitationIsDestroyed`. The confiscation power 0xMacro A-1 describes reaches the position she held at the moment of the rescue, and it ends when the rescue returns. This does not.

**And nothing in the project's own expectations relies on the line.** With `:870` deleted, the repository's Hardhat suite passes **448 of 448, unchanged** — not one test asserts that a rescue marks the rescued account as having claimed. Measured under *Recommendation*.

The Scope tab's first *Public Disclosure* bullet carries the same exception in its own words — *"unless they demonstrate that the role can exceed its intended **authority** or bypass an intended protocol constraint"* — and it is answered the same way. That bullet also enumerates the acts it covers — *"create and settle cover orders, submit matching commitments, assess incidents, approve or reject assessments, configure risk parameters, operate the first-loss buffer, and execute or coordinate payouts"*. Blocklisting and rescuing are not among them.

### ═══ Impact Details ═══

### Money that was never subject to the compliance action

**1 000 000 000 000 of capital that no compliance action ever touched, destroyed with nothing recoverable by anyone.** `test_freshCapitalAfterRehabilitationIsDestroyed`. Both rescue functions are used, as a compliance officer would: after `rescueWithdrawFromBlocklisted` **and** `rescueSharesFromBlocklisted` the account holds **zero shares** — asserted — and the flag is still set. She is rehabilitated, which is asserted too, and then deposits money that has never been near a blocklist:

```
FRESH  shares minted after rehabilitation 1000000000000
FRESH  shares left after redeem           0
FRESH  claim recorded on the period       1000000000000
```

`claimWithdraw` reverts. The whole of that deposit is what is stranded — asserted against the amount minted, not merely logged. Nothing of the original position remains, no claim exists, and the compliance action is over; the flag is a property of the **account**, and it destroys money the action never reached.

### And when she is not the one sending the transaction

The scene above is hers: she deposits, she redeems, and she could in principle have read the flag first. This one removes even that opportunity.

`test_spenderFiresFirst_principalDestroyedInATransactionSheDoesNotSend` — **500 000 000 000 shares burned in a transaction the holder never sends.** Her shares are spent by an approved integrator holding a standing allowance, granted before the poisoning and asserted in the suite. The integrator redeems to her own address — `redeem(shares, holder, holder)` — and the claim is credited to the *receiver* (`:1017`), so it lands on her and meets her flag.

**She is not without levers, and this report does not pretend otherwise.** `approve` is not overridden in `FirelightVault`, so she could set the allowance to zero at any moment before the integrator's call, and `_spendAllowance` (`:1022`) runs *before* the burn (`:1025`) — a zero allowance reverts the whole transaction and nothing is destroyed. What she has no way of obtaining is the one fact that would tell her to do it. That is the subject of *What nothing tells anyone*, below. Measured:

| Step                                                                               | Result                                                                                                       |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| *setup* — a standing allowance to the integrator, granted **before** the poisoning | asserted in the suite, not narrated: the allowance is checked to be already in place once the rescue has run |
| the integrator calls `redeem(rest, victim, victim)`                                | **500 000 000 000 shares burned**                                                                            |
| she revokes the allowance afterwards                                               | **worthless — the shares are already gone**                                                                  |
| she calls `claimWithdraw(p)`                                                       | reverts `AlreadyClaimedPeriod`, for ever                                                                     |

**Control** (`test_negativeControl_spenderPathPaysWithoutTheRescue`): the identical integrator path, on a period that was never rescued, pays **1 000 000 000 000**. The loss is caused by the flag, not by the path.

### Why "permanent" is the exact word

The contract has exactly two asset outflows — `claimWithdraw` (`:808`) and `payout` (`:962`) — and `test_afterTheBite_noCallReachesTheStrandedAsset` closes both, sweeping every caller that could reach them. Gross bucket **1 000 000 000 000**, of which **500 000 000 000** remains after the rescue destination has legitimately taken its own:

| Caller                                | Outcome                                                               |
| ------------------------------------- | --------------------------------------------------------------------- |
| the owner                             | `AlreadyClaimedPeriod`                                                |
| the rescue destination, a second time | `AlreadyClaimedPeriod`                                                |
| any third account                     | `NoWithdrawalAmount`                                                  |
| `RESCUER_ROLE`                        | `AlreadyClaimedPeriod` — refused by the very flag it wrote, at `:864` |
| `PAYOUT_ROLE`                         | `InvalidCapturePeriod` — the payout window has closed                 |

And the protocol's most natural repair is refused too (`test_protocolCannotHandTheClaimBack`): handing the claim **back** to its rehabilitated owner reverts at `:865`, which reads the flag `:870` wrote. On that branch the destination is still paid **500 000 000 000** — the money exists, it simply cannot reach the person it belongs to.

**And the freeze is in the accounting, not only in the call surface.** `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets` (`:428`), and `pendingWithdrawAssets` falls in exactly two places in the whole contract: `claimWithdraw` (`:805`), which now reverts for ever, and `payout` (`:960`), whose window has closed — `_isPeriodInPayoutWindow` admits only the capture period and the next (`:1131`). The stranded assets therefore stay counted as pending permanently: they sit in the vault's own ERC-20 balance, are excluded from `totalAssets()` for ever, and are **not** redistributed to the remaining stakers, whose share price is unchanged. **Nobody gains what she loses.** The contract has no sweep, skim or token-rescue function.

One route is left, and the programme has already ruled on it. The vault sits behind an ERC1967 proxy — `initializeV2()` is a `reinitializer(2)` documented as *"Intended to be called atomically during upgrade via `upgradeToAndCall`"* (`:262-264`) — so the assets could be released only by shipping a new implementation that clears the flag, which is to say by fixing this bug. The programme's own rule: *"Emergency actions may be used to fix or mitigate an issue, but they are not intended as an automatic reason to downgrade an otherwise valid report."* Inside the code that ships at the audited commit, there is no route at all.

Assessed against the programme's Impacts in Scope, not CVSS.

**Impact claimed — Critical tier, verbatim:&#x20;*****"Permanent freezing of funds"*****.** The assets are queued in the period bucket, no caller can reach them, and no role can release them. Measured: 500 000 000 000 stranded after every legitimate claim has been paid.

**Difficulty of exploitation — none, and the discipline that would avoid it is one nothing names.** No attacker is required, and nothing is timed, raced or front-run. `:870` is unconditional in the loop body, so every correctly conducted rescue arms the flag. Whether the arming becomes a loss then turns on a single timing choice — rehabilitating before the period rolls — that no guard, no event, no revert and no line of documentation identifies as consequential.

### What nothing tells anyone

Four parties can prevent this loss. Not one of them is told to.

* **Neither rescue event indexes anything, while both blocklist events index the account.** `WithdrawRescuedFromBlocklisted(address from, address to, uint256[] periods, uint256[] rescuedShares)` (`:119`) and `SharesRescuedFromBlocklisted(address from, address to, uint256 rescuedShares)` (`:110`) declare no `indexed` parameter between them, while `AddedToBlocklist(address indexed account)` (`:129`) and `RemovedFromBlocklist(address indexed account)` (`:135`) — the two a blocklist operator actually watches — both do. **The operator cannot filter either rescue by account.**
* **And it is not the same operator.** `BLOCKLIST_ROLE` is held at `0x189993f3B53284c7e5C02b058EFC6618F78D0cEA` and `RESCUER_ROLE` at `0xC115757266b902d4b1284eA24DBbE557C15E963E`, each 1-of-1. The party who would have to defer rehabilitation is not the party who armed the flag.
* **The flag's own documentation states the opposite of its effect.** `FirelightVaultStorage.sol:70`: *"Indicates whether an account has claimed their withdrawal for a given period."* Anyone who did read `true` would read it as *she has already been paid* — benign — rather than as a live trap.
* **The getter built for the question returns nothing.** `withdrawalsOf(p, her)` (`:466`) reads `withdrawSharesOf[p][her]`, which `:869` has just zeroed. It returns 0, and its own NatSpec presents that figure as complete — *"whether claimed or not"*.
* **`removeFromBlocklist` (`:560-563`) reads no flag and reverts on no condition.** The premature call succeeds silently, and emits only `RemovedFromBlocklist(account)`, which carries no period.

### Six objections, pre-registered — and what each one is worth

**In the leading scene she does send the transaction, and she could in principle have looked.** That is the objection to answer rather than dodge. She has been fully confiscated, cleared, and holds nothing; she then deposits new money. The flag she would have to consult is `isWithdrawClaimed`, whose own documentation says `true` means *she has claimed her withdrawal* — she has claimed nothing and holds no claim at all. Nothing in her position, in any event she can filter, or in any getter she would think to call suggests the question is worth asking. The second scene removes even the opportunity: there the destroying call is sent by the integrator.

**Every lever here is preventive; not one is curative.** Measured (`test_conceded_deferralAfterTheBurnRecoversNothing`): once the burn has happened, freezing the account again and waiting a full period leaves 500 000 000 000 recorded on the armed period and `claimWithdraw` reverting for ever. No role, and no combination of roles, recovers a single unit.

**The operator can prevent it entirely, by deferring rehabilitation past the period roll.** Measured (`test_conceded_deferringRehabilitationPreventsTheLossButRepairsNothing`): the flag is armed either way, but her later request then lands on period 2 rather than the armed period 1, and she is paid 500 000 000 000 in full. The deferral is not free — the account stays frozen for up to a full period, 1 day at the `SMALLEST_PERIOD_DURATION` floor (`FirelightVaultStorage.sol:40`, enforced at `:1085`) and 30 days at the cycle the glossary describes, with its capital still backing coverage and exposed to any payout in that window, and any separate matured claim frozen with it. **And it repairs nothing.** The flag is armed either way — the test name says so — and once the burn has happened the same deferral recovers not one unit (`test_conceded_deferralAfterTheBurnRecoversNothing`).

**The programme's known-issues clause does not reach it either.** That clause excludes *"cases where operational mitigations or procedures **have been implemented** to reduce potential risk"*. None has been. The deferral above is not a procedure the project operates; it is one this report derived from `:1008` and `:1093`, and nothing in the code, the events or the documentation names it — which is the subject of *What nothing tells anyone*.

**The holder holds two levers of her own, and two that are commonly credited to her are not hers.** Hers: she can read the public flag on `isWithdrawClaimed(currentPeriod() + 1, herself)` and revoke the standing allowance — `_spendAllowance` (`:1022`) runs before the burn (`:1025`), so a zero allowance reverts the whole transaction; and she can pre-empt the integrator by redeeming to a fresh receiver herself, since the claim is credited to the *receiver* (`:1017`) and read against the *caller* (`:794`), so a clean address is paid. That second one is derived from those two lines, not measured by this suite. **Not hers**: naming the receiver and choosing the moment belong to whoever sends the transaction, and in the scenario above that is the integrator, not her. All of them require her to know that `:870` fired.

**A third party can stop the burn by keeping her blocklisted for ever.** Measured (`test_conceded_reblocklistingStopsTheBurnButKeepsHerFrozen`): `redeem` reverts `BlocklistedAddress`, not one share is burned, 500 000 000 000 preserved — and unreachable for as long as the freeze that preserves them lasts. That is a choice between two forms of the same freeze.

**The window is one period — stated here rather than left to be found, and the shortest published value is the one taken.** A request lands on `currentPeriod() + 1` (`:1008`) and the counter only advances, so request, blocklist, rescue, rehabilitation and the new request must all fall inside the same window; the PoC asserts exactly that (*"precondition: still the same landing period"*). The project's own material does not settle the length: the Hardhat harness uses 604 800 s, the glossary says *"a fixed 30-day cycle"*, and the staker documentation says *"Periods are currently 1 day long"*. Read it at one day. What the bound limits is how much can be caught, not what happens to it — whatever is requested inside that window is burned and unclaimable for ever, by anyone, and the fresh-capital test shows the flag still bites on money deposited after the rehabilitation.

### Relationship to the prior audits and the published known issues

**OpenZeppelin / Sentora, 2025-07-21, M-02 —&#x20;*****"Period Duration Cannot Be Changed"*****.** This finding describes the identical final state, verbatim:

> *"their shares would be burned during the call to `_requestWithdraw`, but the subsequent call to `claimWithdraw` would revert due to the `isWithdrawClaimed` check."*

**OZ recorded it Resolved, and at the audited commit that route is closed in the code.** M-02 described a `periodDuration` change that made an old request's period current again; `_addPeriodConfiguration` now refuses that structurally — `if (newEpoch < nextPeriodEnd() || …)` (`FirelightVault.sol:1093`). **The same end state is reachable by a second route that closure does not touch**, and that OZ did not consider: the rescue writing the flag against an account that claimed nothing. Same failure mode, different and still-open door.

**Two things follow.** First, this end state is not an accepted consequence of trusted operation: an auditor raised it, the project acted on it, and OpenZeppelin's own summary records the engagement as *9 issues, 8 resolved*, M-02 among them. Second, the severity assigned there belongs to *that* route, not to this one. OZ's route ran through a `periodDuration` setter which, in its own words, *"as of the current contract commit, does not exist"* — a speculative path through a function that had not been written. This route runs only through functions that ship at the audited commit, are documented, and are meant to be used.

**Two findings from the 2025 competition touch this area, and neither is this defect.** `#59027` — *"withdrawalsOf() view function does not account for already withdrawn funds"* — concerns a function declared `external view` (`FirelightVault.sol:466`), which therefore writes nothing and strands nothing; it is a reporting discrepancy, rated Insight. `#59288` is a **gas note on the `rescueWithdrawFromBlocklisted` loop itself**. The function that carries `:870` has already been examined at competition; the write, and the destruction it causes, appear in neither finding.

**0xMacro A-1 describes the rescue power, and this report does not contest it.** A-1 records that `BLOCKLIST_ROLE` with `RESCUER_ROLE` *"can blocklist any account and move its shares and pending withdrawals to an address of choice"*, and calls it *"the standard compliance rescue pattern … a targeted confiscation power over a staker's principal"* — accepted for the MVP.

**That power is not what is reported here, and it ends where `:870` does not.** The confiscation is over when the rescue returns. The flag it also writes is not: capital deposited *after* the account was cleared, which no compliance action ever touched, is destroyed by it — **1 000 000 000 000**, measured in `test_freshCapitalAfterRehabilitationIsDestroyed`. No published power reaches that, and A-1 does not discuss the mechanism that causes it: zero occurrences of `isWithdrawClaimed` and zero of `claimWithdraw` in the report, against 6 of `blocklist`.

**None of the eight published known issues names this mechanism.** Measured across the section: `blocklist` 0, `rescue` 0, `withdraw` 0. The two closest bullets are the permissioned-operations ones. Each closes with the same exception in its own wording — *"unless they demonstrate that the role can exceed its intended **authority** or bypass an intended protocol constraint"* for the permissioned-operations bullet, and *"unless they demonstrate a protocol-constraint bypass or **unauthorized privilege escalation**"* for the incident-assessment one. Both are answered on the privilege limb under *Why this is in scope*.

### ═══ Recommendation ═══

**This report does not ask for a new check.** It asks for the deletion of a line the function it belongs to does not need: `:869` has already emptied the claim, and `:866` already refuses a second rescue on the same period — both measured below. The remediation **removes** 21 bytes rather than adding a guard — on a contract that has only **248 bytes of EIP-170 headroom left** at the audited commit — and it restores the protocol's own repair path, handing the claim back to its owner, instead of only stopping the next burn.

**Delete `FirelightVault.sol:870`.**

```solidity
withdrawSharesOf[periods[i]][to]   += _withdrawOf;
withdrawSharesOf[periods[i]][from]  = 0;
- isWithdrawClaimed[periods[i]][from] = true;
```

The guard that genuinely protects the rescue is `:864`, on the account's **own** flag. Deleting `:870` means that flag is only ever set by an actual claim — which is exactly what its documentation at `FirelightVaultStorage.sol:70-71` says it means.

**Measured, on a copy of the repository at the target commit:**

| Check                                           | Unpatched                       | With `:870` deleted                                      |
| ----------------------------------------------- | ------------------------------- | -------------------------------------------------------- |
| this PoC                                        | 9 passed                        | **2 passed, 7 failed** — see below                       |
| the project's Hardhat suite, `npx hardhat test` | 448 passing                     | **448 passing, 0 failing**                               |
| static analysis, `aderyn`                       | 7 High / 24 Low                 | **identical set** — no type added, no file newly touched |
| `FirelightVault` runtime size                   | 24 328 bytes, 248 under EIP-170 | **24 307 bytes, 269 under — the fix frees 21 bytes**     |

**Verification after the fix — what must fail, and how.** Seven of the nine tests must fail, each in the way the fix predicts. **Four fail inside `vm.expectRevert`** with *"next call did not revert as expected"*, because `claimWithdraw` now pays instead of reverting. **Three fail on an assertion about the flag.** Two on `assertTrue(vault.isWithdrawClaimed(p, victim), "and the flag survives the position entirely")` — the defect asserted directly, and `test_freshCapitalAfterRehabilitationIsDestroyed` inherits it by opening with `test_fullConfiscationLeavesNoShares()`. The third is `test_conceded_deferringRehabilitationPreventsTheLossButRepairsNothing`, on *"the flag is armed either way"*. **No test fails on an amount.** The two that must keep passing are exactly the two that should: the negative control, and the third-party lever.

| Test                                                                    | Expected after the fix                                   |
| ----------------------------------------------------------------------- | -------------------------------------------------------- |
| `test_spenderFiresFirst_principalDestroyedInATransactionSheDoesNotSend` | **FAIL** — `next call did not revert as expected`        |
| `test_afterTheBite_noCallReachesTheStrandedAsset`                       | **FAIL** — idem                                          |
| `test_protocolCannotHandTheClaimBack`                                   | **FAIL** — idem                                          |
| `test_conceded_deferralAfterTheBurnRecoversNothing`                     | **FAIL** — idem                                          |
| `test_fullConfiscationLeavesNoShares`                                   | **FAIL** — `and the flag survives the position entirely` |
| `test_freshCapitalAfterRehabilitationIsDestroyed`                       | **FAIL** — idem                                          |
| `test_conceded_deferringRehabilitationPreventsTheLossButRepairsNothing` | **FAIL** — `the flag is armed either way`                |
| `test_negativeControl_spenderPathPaysWithoutTheRescue`                  | **PASS**                                                 |
| `test_conceded_reblocklistingStopsTheBurnButKeepsHerFrozen`             | **PASS**                                                 |

**A second option, and why it is worse — measured, not assumed.** Guarding the receiver against the flag inside `_requestWithdraw` looks equivalent. It is not:

| Remediation        | size                                        | this PoC       |
| ------------------ | ------------------------------------------- | -------------- |
| delete `:870`      | **24 307**, margin 269 — **frees 21 bytes** | 7 fail, 2 pass |
| guard the receiver | **24 383**, margin 193 — **costs 55 bytes** | 4 fail, 5 pass |

The five that still pass under the guard are the point. It converts the burn into a revert — all four failures are `AlreadyClaimedPeriod`, so no *new* position is destroyed — but `test_protocolCannotHandTheClaimBack` still passes, because `:865` still reads the flag and the assets already stranded by a rescue that has happened stay unreachable. Deleting `:870` lets the claim be handed back to its owner; guarding the receiver leaves that repair blocked and only stops the next burn.

### ═══ References ═══

* `contracts/core/FirelightVaultStorage.sol:70-71` — the flag and its NatSpec
* `contracts/core/FirelightVault.sol` — `:727` / `:765` (`notBlocklisted(receiver)`, origin OZ M-03) · `:794` (`claimWithdraw` reads the flag) · `:845-876` (`rescueWithdrawFromBlocklisted`) · `:864-866` (the three guards) · `:869` (the claim already zeroed) · **`:870` (the defect)** · `:1017` / `:1025` (claim recorded, shares burned) · `:1093` (what closed OZ M-02's route)
* `test/setup/fixtures.js:9` — the project's period duration, used by the PoC
* OpenZeppelin / Sentora, *Firelight Vault Audit*, 2025-07-21 — **M-02** (Resolved, PR #6) and **M-03** (Missing Blacklist Check Can Lock Funds)
* Immunefi 2025 competition — `#59027` (`withdrawalsOf()`, a `view`) and `#59288` (gas, the `rescueWithdrawFromBlocklisted` loop), both rated Insight, neither of them this defect
* `test/rescue.js` — the project's own rescue suite: `:114` (the claim that creates "already claimed"), `:133` and `:152` (the two guards), `:157`-`:181` (the rescue and the destination collecting), and no `removeFromBlocklist` anywhere in the file

## Link to Proof of Concept

<https://gist.github.com/volga0x/275bf9ef5f76bd18d7ba2115bc3f1b0e>

## Proof of Concept

## Run it — one command, from a fresh clone

```sh
git clone https://github.com/immunefi-team/audit-comp-firelight
cd audit-comp-firelight && git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62

# The project's postinstall patch adds SafeMath.sol to OZ 5.2.0, which contracts/fasset/ imports.
# A plain `npm ci --ignore-scripts` leaves the repository unable to compile at all.
npm ci --ignore-scripts && npx patch-package

# forge-std v1.16.2, pinned to the revision the tests were written and verified against (~2 s)
forge install foundry-rs/forge-std@bf647bd6046f2f7da30d0c2bf435e5c76a780c1b

# From the secret Gist linked with this report: foundry.toml and remappings.txt at the repository
# root, H02_PoisonedPeriod.t.sol in a new test-forge/ directory. Then:
forge test --match-contract H02_PoisonedPeriod -vv
```

**9 tests, 9 passed, 0 failed** on `forge 1.5.1-stable` — measured by running exactly these commands against a fresh `git clone` of the public repository, with nothing else added.

**The headline result is step 8.** The steps below trace one compliance lifecycle, but they are not one run: **step 5 and step 8 are separate tests**, and step 8 confiscates to zero the very shares step 5 burns. Read them as two outcomes of the same defect. The worse of the two is not the position that was under the hold, but the **1 000 000 000 000 of capital deposited after the hold ended**, on an account the protocol had already emptied and cleared, destroyed by a flag written before that money existed. Steps 1 to 4 establish how the flag gets there; steps 6 and 7, that nobody can lift it.

**Then delete `FirelightVault.sol:870` and re-run.** Seven of the nine must fail, each in the way the fix predicts: **four inside `vm.expectRevert`** with *"next call did not revert as expected"*, because `claimWithdraw` now pays, and **three on an assertion that the flag survives, or that it is armed at all** — `assertTrue(vault.isWithdrawClaimed(p, victim), "and the flag survives the position entirely")`. No test fails on an amount. The negative control and the third-party lever stay green.

## Step-by-step

All values below are `console.log` output from that suite, run against the production `FirelightVault` at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` on a local network. The vault is the real contract behind its ERC1967 proxy; only the ERC-20 is a mock.

Every call is made by a role that holds the key for it — and `:870` writes an effect that key does not cover. `rescueWithdrawFromBlocklisted` is granted the power to move *one* pending claim to a destination; `:869` has already completed that move; `:870` then adds a permanent bar on every future withdrawal, which the grant never mentions and the rescue does not need. **That is the ground on which this report is in scope**, and step 3 below measures it.

No attacker, no race, no front-running. `:870` is unconditional in the loop body, so every correctly conducted rescue arms the flag. Whether the arming becomes a loss then turns on a timing choice — rehabilitating before the period rolls — that nothing in the code, the events or the documentation identifies as consequential. Both are measured below, in the two `test_conceded_*` cases.

**Setup.** Two stakers deposit 1 000 000 fXRP each, so the vault is not a one-account world. Period duration **604 800 s**, the value the project's own harness uses (`test/setup/fixtures.js:9`).

**The bound this imposes, stated plainly.** A request lands on `currentPeriod() + 1` (`:1008`) and the counter only advances, so steps 1 to 5 below must all fall inside one period — the suite asserts it (*"precondition: still the same landing period"*). The project's material does not settle the length: harness 604 800 s, glossary *"a fixed 30-day cycle"*, staker documentation *"Periods are currently 1 day long"*. **Read it at one day.** The bound limits how much can be caught, not what happens to it: whatever is requested inside that window is burned and unclaimable for ever.

{% stepper %}
{% step %}

### The holder requests a withdrawal

`redeem(half, self, self)`. Her claim is recorded on the landing period `p = currentPeriod() + 1` in `withdrawSharesOf[p][her]`.
{% endstep %}

{% step %}

### She is blocklisted

`BLOCKLIST_ROLE` calls `addToBlocklist(her)` — a nominal compliance hold.
{% endstep %}

{% step %}

### The pending claim is rescued to a destination

This is where the defect is written.

`RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(her, destination, [p])`. Inside the loop body:

```solidity
withdrawSharesOf[p][destination] += _withdrawOf;   // :868  the claim moves
withdrawSharesOf[p][her]          = 0;             // :869  hers is emptied — the rescue is complete
isWithdrawClaimed[p][her]         = true;          // :870  and this is never cleared
```

`:869` has already emptied her claim and `:866` (`_withdrawOf == 0`) already refuses a second rescue on the same period. **`:870` adds nothing to the rescue it completes.** What it does add is a permanent mark on an account that, by the flag's own NatSpec at its declaration site (`FirelightVaultStorage.sol:70-71`, *"Indicates whether an account **has claimed** their withdrawal for a given period"*), **has claimed nothing** — the assets went to the destination.
{% endstep %}

{% step %}

### She is rehabilitated

`removeFromBlocklist(her)`. This is the normal end of a compliance hold, and the returning account has every reason to withdraw again.
{% endstep %}

{% step %}

### An approved integrator redeems for her

**500 000 000 000 shares are burned.** Her shares are spent by an approved integrator holding a standing allowance, granted before the poisoning and asserted in the suite. The integrator redeems to her own address — `redeem(shares, holder, holder)` — and the claim is credited to the *receiver* (`:1017`), so it lands on her and meets her flag.

She is not without levers: `approve` is not overridden, so she could zero the allowance before the integrator's call, and `_spendAllowance` (`:1022`) runs before the burn (`:1025`), so the whole transaction would revert. What she has no way of obtaining is the fact that would tell her to. The rescue event `WithdrawRescuedFromBlocklisted` (`:119`) indexes none of its parameters, unlike the two blocklist events (`:129`, `:135`); `withdrawalsOf(p, her)` (`:466`) returns 0 because `:869` has zeroed her claim; and the flag's own NatSpec says `true` means *she has claimed*.

| Step                                                                               | Measured                                                                                                     |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| *setup* — a standing allowance to the integrator, granted **before** the poisoning | asserted in the suite, not narrated: the allowance is checked to be already in place once the rescue has run |
| the integrator calls `redeem(rest, her, her)`                                      | **500 000 000 000 shares burned**                                                                            |
| she revokes the allowance afterwards                                               | **worthless — the shares are already gone**                                                                  |
| she calls `claimWithdraw(p)`                                                       | reverts `AlreadyClaimedPeriod`, for ever                                                                     |

```
PIVOT  shares burned                  500000000000
PIVOT  claim recorded on the period   500000000000
PIVOT  claimWithdraw reverts AlreadyClaimedPeriod, for ever
```

The order inside `redeem` is what makes it fatal: the burn at `:1025` and the claim record at `:1017` are unconditional and precede any reading of the flag; `claimWithdraw` (`:794`) only checks it afterwards, when there is nothing left to protect.
{% endstep %}

{% step %}

### Once the assets are stranded, no caller reaches them

Gross bucket on the period **1 000 000 000 000**, of which **500 000 000 000** remains after the rescue destination has legitimately taken its own:

| Caller                                | Outcome                                                               |
| ------------------------------------- | --------------------------------------------------------------------- |
| the owner                             | `AlreadyClaimedPeriod`                                                |
| the rescue destination, a second time | `AlreadyClaimedPeriod`                                                |
| any third account                     | `NoWithdrawalAmount`                                                  |
| `RESCUER_ROLE`                        | `AlreadyClaimedPeriod` — refused by the very flag it wrote, at `:864` |
| `PAYOUT_ROLE`                         | `InvalidCapturePeriod` — the payout window has closed                 |

```
STRANDED  withdrawAssets(p), gross bucket    1000000000000
STRANDED  pendingWithdrawAssets, all claims in 500000000000
STRANDED  owner / destination / third party / RESCUER_ROLE / PAYOUT_ROLE: all refused
```

**The role's own write locks the role out.** `RESCUER_ROLE` cannot act on that account for that period again: `:864` refuses it, reading the flag the role itself set at `:870`. An effect its own author cannot undo — and that no other role can undo either — is not the exercise of the power the role was granted.

**And nobody else gets the assets either.** `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets` (`:428`), and `pendingWithdrawAssets` falls in exactly two places in the whole contract: `claimWithdraw` (`:805`), which now reverts for ever, and `payout` (`:960`), whose window has closed. The stranded 500 000 000 000 stay counted as pending permanently — they sit in the vault's own ERC-20 balance, are excluded from `totalAssets()` for ever, and are **not** redistributed to the remaining stakers, whose share price is unchanged. Nobody gains what she loses, and the contract has no sweep, skim or token-rescue function.
{% endstep %}

{% step %}

### The protocol's most natural repair is refused by the same flag

Handing the claim **back** to its rehabilitated owner reverts at `:865`, which reads what `:870` wrote. The money is not lost on that branch — the destination can still be paid — it simply cannot reach the person it belongs to.

```
REPAIR  handing the claim back is refused at :865 - it reads the flag :870 wrote
REPAIR  the rescue destination is still paid  500000000000
```

{% endstep %}

{% step %}

### Money that was never subject to the compliance action is destroyed by it

Both rescue functions are used, as a compliance officer would. After `rescueWithdrawFromBlocklisted` **and** `rescueSharesFromBlocklisted`, the account holds **zero shares** — asserted — and the flag is still set. She is rehabilitated, asserted too, and deposits money that has never been near a blocklist:

```
CONFISCATION  shares left to the account 0
CONFISCATION  flag still set on it       1
FRESH  shares minted after rehabilitation 1000000000000
FRESH  shares left after redeem           0
FRESH  claim recorded on the period       1000000000000
```

`claimWithdraw` reverts. Nothing of the original position remains, and the new capital is burned anyway. **The flag is a property of the account, not of the claim it was written against.**
{% endstep %}
{% endstepper %}

## The two controls, and the three levers — all measured in the suite

**Negative control — the loss is caused by the flag, not by the path.** The identical integrator sequence on a period that was never rescued pays in full:

```
CONTROL  same path, no rescue: assets received 1000000000000
```

**Exhaustive caller sweep** — step 6 above is not a sample, and the contract bounds the question: it has exactly two asset outflows, `claimWithdraw` (`:808`) and `payout` (`:962`). Five distinct callers, including the two roles that could reach those outflows, each refused with its own named error.

**Three levers, measured rather than argued — every one of them a gate, none of them a repair.**

*A third party can stop the burn* by keeping the account blocklisted for ever. With `BLOCKLIST_ROLE` re-applied, `redeem` reverts `BlocklistedAddress` and not one share is burned:

```
CONCEDED  a third party can stop the burn, by freezing her instead
CONCEDED  shares preserved                500000000000
```

That is a choice between two forms of the same freeze, not a repair — and it is available to the role, never to the holder. It is in the test suite, not hidden from it.

## What must fail once `:870` is deleted

The remediation is a **deletion**, not an added guard: `:869` has already emptied the claim and `:866` already refuses a second rescue, both exercised by the suite.

`foundry.toml` is required and not optional: it replicates the compiler settings `hardhat.config.js` applies to the contracts under test, including both per-file overrides.

| Test                                                                    | After deleting `:870`                                    |
| ----------------------------------------------------------------------- | -------------------------------------------------------- |
| `test_spenderFiresFirst_principalDestroyedInATransactionSheDoesNotSend` | **FAIL** — `next call did not revert as expected`        |
| `test_afterTheBite_noCallReachesTheStrandedAsset`                       | **FAIL** — idem                                          |
| `test_protocolCannotHandTheClaimBack`                                   | **FAIL** — idem                                          |
| `test_conceded_deferralAfterTheBurnRecoversNothing`                     | **FAIL** — idem                                          |
| `test_fullConfiscationLeavesNoShares`                                   | **FAIL** — `and the flag survives the position entirely` |
| `test_freshCapitalAfterRehabilitationIsDestroyed`                       | **FAIL** — idem                                          |
| `test_conceded_deferringRehabilitationPreventsTheLossButRepairsNothing` | **FAIL** — `the flag is armed either way`                |
| `test_negativeControl_spenderPathPaysWithoutTheRescue`                  | **PASS**                                                 |
| `test_conceded_reblocklistingStopsTheBurnButKeepsHerFrozen`             | **PASS**                                                 |

Measured alongside: the project's own Hardhat suite stays at **448 passing, 0 failing** — **not one of its tests asserts that a rescue marks the rescued account as having claimed**; `aderyn` returns an identical finding set; and `FirelightVault` drops from **24 328 to 24 307 bytes**, returning 21 of the **248 bytes of EIP-170 headroom** the contract has left at the audited commit.
