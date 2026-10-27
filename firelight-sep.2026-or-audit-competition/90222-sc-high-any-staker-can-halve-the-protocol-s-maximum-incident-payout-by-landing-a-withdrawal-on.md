# 90222 sc high any staker can halve the protocol s maximum incident payout by landing a withdrawal on the exact period start timestamp

**Submitted on Aug 23rd 2026 at 20:00:37 UTC by @drak3p14159 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90222
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

`FirelightVault.payout()` caps an incident payment at `totalAssetsAt(capturePeriodStart)`, a value read from a timestamp-keyed checkpoint trace whose lookup returns the last entry with key **≤** the query. Any unprivileged staker who lands a withdrawal in a block whose timestamp is exactly `capturePeriodStart` writes a new, lower checkpoint at that same key, so the cap reads the **post-withdrawal** total — while those same withdrawn assets remain fully exposed to the incident and are explicitly added back into the payable set by the very next lines. In production this lets any staker unilaterally suppress the protocol's maximum payout for the period: the A/B-controlled PoC supplied in the Proof of Concept field measures an approved claim underpaid by 50% (2,000 → 1,000) while the attacker retains 500 in slashable assets that the control run shows would otherwise have been taken, and the suppression scales with the attacker's share of the vault.

## Vulnerability Details

The payout is bounded by the vault's assets at the start of the covered period:

```solidity
// contracts/core/FirelightVault.sol:909-927
// Cap by the active assets committed at the start of the covered period.
// This is an exposure cap, not a segregated asset bucket.
@>  uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
    uint256 currentActiveAssets = totalAssets();
    uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
    ...
    uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
@>  paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

That bound resolves through a checkpoint lookup that accepts any entry at or before the queried key:

```solidity
// contracts/core/FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
@>      return _traceTotalAssets.upperLookupRecent(timestamp);   // key <= timestamp
}
```

And every withdrawal pushes a new checkpoint at the current block timestamp, so the key the cap reads is writable by an ordinary user action:

```solidity
// contracts/core/FirelightVault.sol:739-741 (redeem) and 773-775 (withdraw)
    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
@>  _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
// _logTrace -> _traceTotalAssets.push(Time.timestamp(), _totalAssets)
```

The two reads are inconsistent by construction — and that inconsistency, not the choice of snapshot convention, is the defect. The **cap** is taken *after* a boundary withdrawal has reduced it, while `activePayableAmount` *adds the very same assets back* through `withdrawAssets[capturePeriod + 1]` — the bucket the design deliberately keeps slashable precisely so that "exposed assets could \[not] escape by withdrawing first" (the code's own comment at `:918-920`). The protocol therefore treats those assets as exposed for the purpose of what it may pay out of, and as absent for the purpose of how much it may pay at all.

No privileged role is involved at any step. The attacker needs only an ordinary staked position and a withdrawal transaction landing in one specific block.

**Reachability of the exact-timestamp landing.** The attack needs a withdrawal to execute in a block whose timestamp equals `capturePeriodStart` exactly. That is a repeated low-cost attempt, not a coincidence:

* `capturePeriodStart` is deterministic and publicly readable on-chain (`currentPeriodStart()` / `currentPeriodEnd()`), so the attacker knows the target second well in advance.
* Periods are long — 604,800 seconds in the configuration the PoC exercises — so a missed attempt simply waits for the next boundary. There is no limit on attempts.
* The attempt is an ordinary `withdraw` the staker is entitled to make anyway, so missing forfeits nothing and changes no position.
* It can be automated by routing the withdrawal through a contract that reverts unless `block.timestamp == currentPeriodStart()`. A miss then costs only the gas of an early revert and leaves the staked position untouched.

The PoC uses `setNextBlockTimestamp` to place the block deterministically, which is the ordinary way to reproduce a timing condition in a test suite; it is not an assumption that the attacker holds any special privilege. Under Immunefi's Feasibility Limitation Standards, attack difficulty is not by itself grounds for downgrade absent the attacker's risk extremely outweighing the reward — and here a failed attempt risks nothing beyond gas.

## Impact Details

A staker halves the protocol's maximum payout for the period and retains slashable value.

Measured by the PoC, with a control isolating the single variable (withdrawal timing):

|                       | control: withdrawal +60s | attack: withdrawal at exact period start |
| --------------------- | ------------------------ | ---------------------------------------- |
| payout cap            | 2,000                    | **1,000**                                |
| paid to the claimant  | 2,000                    | **1,000**                                |
| attacker retains      | 0                        | **500**                                  |
| honest staker retains | 0                        | 500                                      |

**The approved claim is underpaid by 50% while the vault holds the assets**, and the attacker keeps 500 that the control shows would otherwise have been slashed. Scaling: an attacker holding a larger fraction of the vault suppresses the cap proportionally; if every active staker withdraws at the boundary the cap approaches zero and the vault pays nothing at all on a valid claim.

**Who bears the loss.** The table shows the honest staker also retaining 500 in the attack run, so it is worth stating plainly who is harmed. It is not another staker: it is the **cover beneficiary**, whose approved and assessed claim is paid at half its value while the vault demonstrably holds the assets. The attacker's gain and the claimant's loss are separate measurements — 500 retained and 1,000 underpaid — and each is shown against the control.

**The defect is in the cap, not in any external reserve.** `totalAssetsAt(capturePeriodStart)` returns a value that a boundary write has already lowered, before any downstream component sees it. That follows from the two reads shown above — `upperLookupRecent` resolves key **≤** timestamp, and the withdrawal pushes its checkpoint at exactly that key — so it holds on every execution, independent of the state of any reserve the protocol maintains outside the vault. The first-loss buffer is a downstream top-up toward the claimant: it does not correct the ceiling, and it does not return the assets the attacker retains in `withdrawAssets[capturePeriod + 1]` to the payable set. Those assets are slashed in the control run and retained in the attack run, and no external payment path reaches them — which is the *Direct theft of any user funds* mapping, and it does not depend on any reserve being empty.

This distinction matters against the out-of-scope list, which excludes *"the first-loss buffer being an external, unescrowed wallet; and reports based only on the first-loss-buffer token depegging without a protocol bug causing or mishandling that depeg."* **This report is neither of those.** It does not rest on the buffer's custody model, and it does not concern a depeg. It identifies a defect in the vault's own checkpoint accounting, inside `FirelightVault.payout`, reachable by an unprivileged user with no privileged role involved at any step. The exclusion covers the buffer's *existence* as an external wallet; it does not convert that wallet into a mitigation for an independent defect elsewhere in the system. Reading it that way would also invert it — the exclusion records that the buffer is unescrowed and outside the protocol's control, whereas closing this report on the buffer would require assuming it is funded and available at the moment of the incident.

This defeats the exposure cap on which the protocol's solvency model rests, and it is reachable by an **unprivileged user** — the programme states that *bypassing an intended protocol constraint* is in scope.

**Impacts selected — both from the programme's in-scope list, either sufficient on its own.** ***Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield***: an unprivileged attacker manipulates the payout so that the claimant incurs a measured loss during the transfer while the attacker profits in retained slashable assets — the 1,000 and 500 figures above, both A/B controlled. ***Protocol insolvency***: because the cap is suppressible by any staker, cover written against period-start capacity exceeds what the protocol can be made to pay on a valid claim. Both are Critical under Severity Classification System v2.3.

The runnable PoC that produces these figures, the ablation confirming the boundary read is their cause, and the reason a local test against the project's own fixtures is the appropriate form here (rather than a mainnet fork) are all supplied in the **Proof of Concept** field of this submission.

## Distinctness from the adjacent capacity defect — verified by cross-ablation

There is a separate, nearby weakness in `CoverOrderAllocator._computeAvailableCapacity` concerning a missing prior-period reserve, and a triager may reasonably wonder whether this report is a symptom of it. **It is not**, and that was verified by execution rather than asserted: applying the capacity-reserve fix and re-running this PoC unchanged leaves both measured consequences intact.

| applied fix                                       | this PoC                                                          | verdict                     |
| ------------------------------------------------- | ----------------------------------------------------------------- | --------------------------- |
| the capacity-reserve fix in `CoverOrderAllocator` | claimant still underpaid **1000**, attacker still retains **500** | **this defect survives it** |

Different contract, different read, different fix — and a single remediation cannot close both. If this report is considered alongside any capacity-side finding, that is the operative test of whether the two describe one defect or two.

## Recommended Fix

**(a) One immutable opening-exposure snapshot, shared by the allocator and the payout — preferred.** Materialise the period's opening exposure once, and have both `_computeAvailableCapacity` and `payout` read that same stored value. A historical figure that a commitment has already been priced against must not remain overwriteable by later activity at the same key.

**(b) Smaller mitigation: use the strict pre-boundary value consistently in BOTH components.** Read the exposure at `capturePeriodStart - 1` so a checkpoint written *at* the period start — which belongs to the new period, not the old one — cannot lower it. ⚠ Apply this in `_computeAvailableCapacity` as well as in `payout`: the allocator reads `totalAssetsAt(currentPeriodStart())` and is exposed to the same boundary write. Fixing only the payout leaves the capacity path wrong.

**(c) Defer the withdrawal checkpoint to the period the request belongs to.** A request made at time T is booked to bucket `period(T)+1`; pushing its asset-reducing checkpoint at T lets it retroactively affect a period it is still exposed to.

⛔ **A fix that was considered and REJECTED, recorded because it is the intuitive one:** adding the still-slashable withdrawal buckets back into the cap (`totalAssetsAt(capturePeriodStart) + withdrawAssets[cp+1] + withdrawAssets[cp+2]`). This does make the measured shortfall disappear, but it is **not safe in production**: ordinary withdrawals are already represented in the opening snapshot, so adding whole buckets **double-counts exposure** and can create room to charge post-start deposits. Do not adopt it.

## References

All links pinned to the in-scope tag `v1_audit_ready` (HEAD `42f9ea5`).

* **The payout cap** — `FirelightVault.payout()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L909-L927>
* **The boundary-tolerant read** — `FirelightVault.totalAssetsAt()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L456-L458>
* **Checkpoint written at the current timestamp** — `redeem()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L739-L741>
* **Same write on the other path** — `withdraw()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L773-L775>
* **Lookup semantics (`key <= timestamp`)** — `Checkpoints.upperLookupRecent`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Second site requiring fix (b)** — `CoverOrderAllocator._computeAvailableCapacity`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **PoC** — `_audit_poc_boundary_cap_evasion.js`, run against the project's own fixtures (`test/setup/fixtures.js`); step-by-step explanation, full source and run instructions are in the Proof of Concept field of this submission.
* **Severity mapping** — Immunefi Vulnerability Severity Classification System v2.3, Smart Contracts → Critical: *Direct theft of any user funds…* and *Protocol insolvency*. <https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/>

## Link to Proof of Concept

<https://gist.github.com/drak3p14159/2a1fdc059fe771c88db463ad4bb6c699>

## Proof of Concept

## Step-by-step explanation of the attack and its impact

**Preconditions.** A vault with a one-week period. Two stakers each deposit 1,000, so the vault holds 2,000 at the start of the covered period. The attacker is one of them — an ordinary staker, no privileged role.

1. The attacker reads `currentPeriodEnd()`, which is public. This is the exact second at which the next `capturePeriodStart` falls, known in advance.
2. The attacker submits an ordinary `withdraw(1000)` and lands it in the block whose timestamp is **exactly** that boundary second.
3. That withdrawal calls `_logTrace`, which pushes a checkpoint `(key = capturePeriodStart, totalAssets = 1000)` — a *new, lower* value written at the very key the payout cap will read.
4. An incident covering that period is declared and a claim of 2,000 is approved.
5. `payout()` computes its ceiling as `totalAssetsAt(capturePeriodStart)`. That resolves through `upperLookupRecent`, which returns the last checkpoint with key **≤** the query — and the attacker's checkpoint sits exactly on the key. It returns the **post-withdrawal 1,000**, not the 2,000 the vault actually held when the period opened.
6. `paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount))` therefore binds at 1,000. The claimant is paid **1,000 against an approved 2,000**.
7. Meanwhile the attacker's withdrawn 1,000 sits in `withdrawAssets[capturePeriod + 1]`, which the same function explicitly *adds back* into the payable set — the protocol still considers those assets exposed. Because the ceiling was suppressed, only part of them is taken: the attacker **retains 500**.

**The control that isolates it.** The test then runs the identical scenario a second time, changing exactly one thing: the withdrawal lands 60 seconds later instead of on the boundary. The checkpoint then falls *after* the key, the cap reads the full 2,000, the claimant is paid **2,000**, and the attacker retains **0**. Nothing else differs between the two runs.

**Impact.** The claimant is underpaid by 1,000 — 50% of an assessed claim — while the vault demonstrably holds the assets, and the attacker keeps 500 that the control proves would otherwise have been slashed. Both figures scale with the attacker's share of the vault; if every active staker withdraws on the boundary the ceiling approaches zero and a valid claim pays nothing.

## Why this is a local test against the project's own suite, and not a mainnet fork

Immunefi's PoC guidelines require a runnable PoC and permit the project's existing test suite where *"forking the mainnet state is not feasible"*. Forking is not feasible for this defect: the payout path requires Phase 2 contracts that are **not yet deployed** on Flare, so there is no on-chain state to fork. This PoC therefore runs against the in-scope source at commit `42f9ea5` (tag `v1_audit_ready`) using the project's **own unmodified test fixtures** (`test/setup/fixtures.js`), deploys the real `FirelightVault`, and measures real token balance changes. No mainnet or public testnet is touched, no live deployment is interacted with at any point, and no contract under `contracts/` is modified.

## How to run

```
git clone https://github.com/immunefi-team/audit-comp-firelight
cd audit-comp-firelight
git checkout 42f9ea5                      # tag v1_audit_ready
npm ci
# save the source below as test/_audit_poc_boundary_cap_evasion.js
#   (it must live in test/ — it imports the project's own ./setup/fixtures.js)
npx hardhat test test/_audit_poc_boundary_cap_evasion.js     # run from the repo root
```

## The test — complete source

It runs the identical scenario twice and moves **only the withdrawal timestamp**, so the difference between the two runs is attributable to nothing else.

```javascript
// PoC — a staker who lands a withdrawal on the EXACT period-start timestamp overwrites the
// historical totalAssets checkpoint that the payout cap reads, unilaterally reducing the maximum
// the vault can pay for that period's incident — and keeping assets that would otherwise be slashed.
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')
const DEC = 6
const U = (n) => ethers.parseUnits(String(n), DEC)
const f = (x) => ethers.formatUnits(x, DEC)

async function scenario(useBoundaryTrick) {
  const ctx = await loadFixture(deployVault.bind(null, {
    decimals: DEC, initial_deposit_limit: U(1000000).toString(), period_configuration_duration: 604800 }))
  const { firelight_vault: v, token_contract: tok, users, utils, payout_signer, payout_receiver } = ctx
  const [alice, bob] = users
  for (const u of [alice, bob]) await utils.mintAndApprove(U(10000), u)
  await v.connect(alice).deposit(U(1000), alice.address)   // attacker
  await v.connect(bob).deposit(U(1000), bob.address)       // honest staker

  const boundary = await v.currentPeriodEnd()
  if (useBoundaryTrick) {
    await time.setNextBlockTimestamp(boundary)             // land EXACTLY on the period start
    await v.connect(alice).withdraw(U(1000), alice.address, alice.address)
  } else {
    await time.setNextBlockTimestamp(boundary + 60n)       // one minute later — same intent, no trick
    await v.connect(alice).withdraw(U(1000), alice.address, alice.address)
  }

  const P = await v.currentPeriod()
  const start = await v.currentPeriodStart()
  const cap = await v.totalAssetsAt(start)
  const before = await tok.balanceOf(payout_receiver.address)
  await v.connect(payout_signer).payout(payout_receiver.address, U(2000), start)   // full loss
  const paid = (await tok.balanceOf(payout_receiver.address)) - before

  // what each party can still get out
  const aliceBucket = await v.withdrawalsOf(P + 1n, alice.address)
  const bobLive = await v.maxWithdraw(bob.address)
  return { cap, paid, aliceBucket, bobLive, P }
}

describe('PoC: period-boundary checkpoint overwrite evades the payout exposure cap', function () {
  this.timeout(300000)

  it('timing a withdrawal to the exact period start cuts the protocol payout and saves the staker', async () => {
    const ctl = await scenario(false)
    const atk = await scenario(true)

    console.log(`\n    CONTROL — withdrawal one minute after the period start:`)
    console.log(`      payout cap  = ${f(ctl.cap)}`)
    console.log(`      actually paid to claimant = ${f(ctl.paid)}`)
    console.log(`      attacker keeps (pending bucket) = ${f(ctl.aliceBucket)}`)
    console.log(`      honest staker keeps            = ${f(ctl.bobLive)}`)

    console.log(`\n    ATTACK — the same withdrawal landed on the EXACT period-start timestamp:`)
    console.log(`      payout cap  = ${f(atk.cap)}`)
    console.log(`      actually paid to claimant = ${f(atk.paid)}`)
    console.log(`      attacker keeps (pending bucket) = ${f(atk.aliceBucket)}`)
    console.log(`      honest staker keeps            = ${f(atk.bobLive)}`)

    const claimantLoss = ctl.paid - atk.paid
    const attackerGain = atk.aliceBucket - ctl.aliceBucket
    console.log(`\n    => claimant is underpaid by ${f(claimantLoss)}`)
    console.log(`       attacker keeps ${f(attackerGain)} more than in the control`)
    console.log(`       honest staker delta ${f(atk.bobLive - ctl.bobLive)}\n`)

    expect(atk.cap, 'the exposure cap is lowered by the boundary withdrawal').to.be.lt(ctl.cap)
    expect(atk.paid, 'the claimant is paid less').to.be.lt(ctl.paid)
    expect(attackerGain, 'the attacker retains value that would otherwise be slashed').to.be.gt(0n)
  })
})
```

## Output

```
CONTROL — withdrawal one minute after the period start:
  payout cap  = 2000.0
  actually paid to claimant = 2000.0
  attacker keeps (pending bucket) = 0.0

ATTACK — the same withdrawal landed on the EXACT period-start timestamp:
  payout cap  = 1000.0
  actually paid to claimant = 1000.0
  attacker keeps (pending bucket) = 500.0

=> claimant is underpaid by 1000.0
   attacker keeps 500.0 more than in the control
```

The claimant is underpaid by **1000** (50% of an approved 2000 claim) and the attacker retains **500** in assets that the control run shows would otherwise have been slashed.

## Cause verified by ablation, not merely asserted

Applying recommended fix (b) — reading the exposure strictly before the boundary — and re-running the **unchanged** test makes both measured consequences vanish:

```
                       baseline            with fix (b)
paid to claimant       1000.0              2000.0      (full)
attacker retains        500.0                 0.0
claimant underpaid     1000.0                 0.0
```

The test then fails, because it asserts the attack succeeds. The measured effect is therefore attributable to the boundary read and to nothing else.

One note for exactness: the line the test logs as `cap` is the raw `totalAssetsAt(start)` and still prints 1000 after the fix — it is an adjacent proxy for the effective cap, not the cap itself. The payout and retention figures are the operative measurements. The contract was restored after the ablation; `git diff` on `contracts/` is empty at `42f9ea5`.
