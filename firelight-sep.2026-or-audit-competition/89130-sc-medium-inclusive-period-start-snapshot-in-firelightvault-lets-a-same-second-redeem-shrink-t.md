# 89130 sc medium inclusive period start snapshot in firelightvault lets a same second redeem shrink the payout cap and sold cover capacity leading to griefing of cover claimants

**Submitted on Aug 17th 2026 at 11:47:53 UTC by @AvrahamAdvisory for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89130
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

At the exact second a period begins, `FirelightVault` treats one and the same transaction as both *inside the new period* and *part of that period's opening balance*. Because `Checkpoints.push` overwrites in place when the key is unchanged, a permissionless `redeem` landing in that second retroactively lowers the period-start snapshot — the single value that both caps insurance payouts and sizes how much cover may be sold. In production this means cover can be sold against 1000e18 of capacity while payouts for that same period are capped at 500e18, so approved claimants receive half of what the protocol sold them, with the assets still sitting in the vault. The same shortfall also arises **accidentally**, with no attacker at all, whenever an ordinary LP exits in a period's first block after a matching commitment has been posted.

## Vulnerability Details

### The two colliding conventions

At timestamp `pStart` (the first second of period `P`), the vault applies two contradictory rules to the same transaction:

* **`currentPeriod()` already returns `P`.** So `_requestWithdraw` files the withdrawal into `withdrawAssets[P+1]` — a bucket that `payout` explicitly treats as *still exposed and payable* (`FirelightVault.sol:916`).
* **`totalAssetsAt(capturePeriodStart)` is an inclusive lookup.** `Checkpoints.upperLookupRecent` is documented as returning "the value in the last checkpoint with a key **lower or equal** than the search key", so the transaction's *post*-state becomes period `P`'s opening snapshot.

One transaction is therefore simultaneously "inside period P" and "part of period P's opening balance".

### Why the write survives

`Checkpoints.push` replaces the stored value in place when the key equals the last key (`contracts/core/lib/Checkpoints.sol:207-208`):

```solidity
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();
uint256 len = self._values.length;
uint256 lastValue = latest(self);
if (exists && key == lastKey) {
    self._values[len - 1] = value;      // in-place overwrite, no new entry
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

No new checkpoint is appended, so **the last writer in that one second wins** and the earlier value is unrecoverable.

### The two consumers of that snapshot

```solidity
// FirelightVault.sol:911 — caps insurance payouts
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));

// CoverOrderAllocator.sol:711 — sizes sellable cover
$.vault.totalAssetsAt($.vault.currentPeriodStart()),
```

A permissionless `redeem` at `pStart` therefore moves both the payout ceiling and the capacity figure downward *after* they have been read and acted upon.

### This contradicts the codebase's own intent

`VaultRewardDistributor` spends a dedicated privileged call specifically to keep this snapshot accurate (`VaultRewardDistributor.sol:148-150`, repeated at `:172-174`):

> "Checkpoint atomically so historical `totalAssetsAt` lookups (**period-start snapshots**) include the forwarded assets. Requires `CHECKPOINT_ROLE` on the vault."

And two of the project's own tests pin the snapshot as stable against activity inside the period:

* `test/coverOrderAllocator/commitAllocation.js:230` — *"uses `totalAssetsAt(currentPeriodStart)` — mid-period deposits do NOT inflate capacity"*. The inflate direction is tested and closed; the **deflate** direction at the boundary second is not.
* `test/FirelightVault/checkpoint.js:55-56` — asserts `totalAssetsAt(periodStart)` still returns the pre-donation figure, commenting *"history does not"* change.

So the snapshot is intended to be immutable once written. A same-second `redeem` rewrites it — the untested half of an invariant the project deliberately tests.

## Impact Details

### Impact A — cover sold against capacity that no longer exists

Ordering within the first block of a period:

{% stepper %}
{% step %}

## `commitAllocation` reads 1000e18 of capacity

`commitAllocation` (`ALLOCATOR_ROLE`, behaving entirely honestly) reads capacity from `totalAssetsAt(currentPeriodStart())` = **1000e18** and commits.
{% endstep %}

{% step %}

## A permissionless redeem overwrites the checkpoint

An ordinary LP's permissionless `redeem`, ordered after it **in the same block**, overwrites that same checkpoint key down to **500e18**.
{% endstep %}

{% step %}

## Cover settles against frozen capacity

Cover settles against the frozen `commit.totalAvailableCapacity`. `_settleCoverOrder` reads only `commit.totalAvailableCapacity` (`:507`) and `commit.totalDeclaredAllocated` (`:525`); it never re-derives capacity.
{% endstep %}
{% endstepper %}

**Result: 1000e18 of cover sold, payouts capped at 500e18 — a 500e18 shortfall.**

| scenario                                        | capacity at commit | cover sold | payout cap |    paid |  shortfall |
| ----------------------------------------------- | -----------------: | ---------: | ---------: | ------: | ---------: |
| **Backrun** (commit, then redeem, same second)  |            1000e18 |    1000e18 |     500e18 |  500e18 | **500e18** |
| Control A (redeem *before* commit, same second) |             500e18 |     500e18 |     500e18 |       — |          0 |
| Control B (identical redeem, **+1 second**)     |            1000e18 |    1000e18 |    1000e18 | 1000e18 |          0 |

The protocol's own validator confirms the commitment is dead on arrival: calling `recommitAllocation` afterwards with the **identical** numbers reverts `MatchingCapacityOverflow(1000e18, 500e18)`. The contract rejects the commitment it accepted one transaction earlier.

### Impact B — claimants underpaid by a boundary-second exit

With no allocator involvement at all:

| scenario               | payout cap | paid to claimants | redeemer recovers |
| ---------------------- | ---------: | ----------------: | ----------------: |
| no redeem              |    1000e18 |       **1000e18** |                 0 |
| redeem **at** `pStart` |     500e18 |        **500e18** |            250e18 |
| redeem `pStart − 1`    |     500e18 |            500e18 |            500e18 |
| redeem `pStart + 1`    |    1000e18 |           1000e18 |                 0 |

After the boundary redeem the vault holds **1000e18 that it itself classifies as exposed** (500e18 active plus 500e18 in `withdrawAssets[P+1]`, which `payout:916` counts as payable) yet caps payment at 500e18. It refuses to pay half of what its own exposure accounting committed to.

### What this is not — stated plainly

* **Not insolvency.** The vault is *over*-collateralised relative to what it disburses: 1000e18 of assets remain against a 500e18 cap. `test_ShortfallIsAccountingNotCollateral` measures this directly. The shortfall is an accounting artifact, not a funding gap.
* **Not theft, and not profitable.** Exiting one second *earlier* pays the actor 500e18 instead of 250e18 **and** produces a correctly-sized capacity with zero shortfall. The boundary action is strictly worse for the actor on every axis — it costs them 250e18 to create the gap.
* **Not freezing.** The unpaid 500e18 stays with LPs and remains claimable normally.

What it *is*: cover buyers paid premiums for cover the vault structurally refuses to honour, and a correct privileged commitment silently invalidated one transaction after it passed validation. That is why the claimed severity is **Medium — Griefing (no profit motive for an attacker, but damage to the users or the protocol)**, with **Low — Contract fails to deliver promised returns, but doesn't lose value** as the secondary framing for Impact A.

### Why the privileged-actor exclusion does not apply

The program excludes reports that "require a privileged role holder to act within the permissions explicitly granted to that role". Addressing that directly:

* The privileged actors here — `ALLOCATOR_ROLE` calling `commitAllocation`, `PAYOUT_ROLE` executing a payout — are assumed to behave **entirely honestly and within their permissions**. Nothing depends on a role misbehaving, exceeding its grant, or being compromised.
* The action that causes the harm is `redeem`, which is **permissionless**.
* Impact B requires no allocator involvement at all.

The privileged calls here are the operations being *harmed*, not the attack vector.

### Preconditions, stated honestly

1. For Impact A, `commitAllocation` must land in a block whose timestamp equals `pStart` exactly. Nothing forces this — `CommitTooCloseToPeriodEnd` only bars *late* commits — and orders for period `P` are all created during `P−1`, so an automated matcher firing at the rollover is plausible. But **one second of jitter removes the bug**, and I could not verify Firelight's operational timing, so I am not claiming it.
2. The redeem must be ordered after the commit in that block (a backrun bundle, or probabilistically via fee ordering).
3. An incident must land in that period for the shortfall to be realised.

Impact B needs only precondition 3.

{% hint style="info" %}
**Amplifier:** with `settlementGracePeriod` at `0` — the post-deployment default, which the project's own test asserts and exercises (`test/coverOrderAllocator/commitAllocation.js:1516-1531`) — a commit and `batchSettleCoverOrder` can land in the same block, removing the operator's window to `recommitAllocation`/`cancelCommitAllocation` after seeing the failure. That configuration is the project's choice and is not itself claimed as a defect here.
{% endhint %}

### Relationship to declared known issues

Checked against all three declared known issues and the out-of-scope list; none cover this:

* **"Aggregate claims are not capped on-chain against total cover sold"** is the opposite direction — it accepts that *claims* may exceed *cover*. This report is about *cover sold* exceeding the *payout cap*.
* **"Severe collateral drawdowns can reduce coverage capacity and may cause shortfalls or undercollateralization"** accepts shortfalls arising from a **decline in the staked asset's USD value**. Here there is no price movement at all: the vault holds the full 1000e18 throughout.
* The first-loss-buffer known issues concern that wallet's funding, approvals and peg — none of which appear in this report.

## Recommended fix

Read the period-opening snapshot **exclusive** of the boundary second, matching the convention that `currentPeriod()` and the withdrawal-bucket assignment already use:

```solidity
// FirelightVault.payout
uint256 cap = totalAssetsAt(capturePeriodStart - 1);

// CoverOrderAllocator._computeAvailableCapacity
uint256 assets = vault.totalAssetsAt(vault.currentPeriodStart() - 1);
```

This makes the opening snapshot immutable once the period starts, so neither a commitment nor a payout cap can be moved by activity inside the period they measure. Both impacts share this one root cause and this one fix.

## References

In-scope code (`immunefi-team/audit-comp-firelight` @ `v1_audit_ready`):

* `contracts/core/FirelightVault.sol:911` — payout cap reads the period-start snapshot
* `contracts/core/FirelightVault.sol:916` — `withdrawAssets[capturePeriod + 1]` counted as payable
* `contracts/core/CoverOrderAllocator.sol:711` — capacity sized from the same snapshot
* `contracts/core/CoverOrderAllocator.sol:507`, `:525` — settlement reads only the frozen commitment
* `contracts/core/lib/Checkpoints.sol:207-208` — same-key in-place overwrite
* `contracts/core/VaultRewardDistributor.sol:148-150` — the stated intent that period-start snapshots be accurate

Project tests that pin the invariant this breaks:

* `test/coverOrderAllocator/commitAllocation.js:230`
* `test/FirelightVault/checkpoint.js:55-56`

## Link to Proof of Concept

<https://gist.github.com/shahar20201993-spec/ab6e1466cddbfd3962e88b095ff7986f>

## Proof of Concept

### Step-by-step — Impact A (same-block commit backrun)

**Setup.** A period `P` begins at timestamp `pStart`. The vault holds 1000e18 of total assets: 500e18 belonging to an ordinary LP ("Larry") and 500e18 to other depositors. Capacity is configured with `effectiveLeverage == minCAR == 12000`, so capacity reads 1:1 against collateral.

{% stepper %}
{% step %}

## Block N, tx 1 — the honest commitment

`ALLOCATOR_ROLE` calls `commitAllocation(period = P, root, totalAllocated = 1000e18, ...)`.

Internally `_computeAvailableCapacity` reads `vault.totalAssetsAt(vault.currentPeriodStart())`. Because `block.timestamp == pStart` and the lookup is inclusive, it returns **1000e18**. The commitment validates and is stored, freezing `commit.totalAvailableCapacity = 1000e18`.
{% endstep %}

{% step %}

## Block N, tx 2 — Larry exits, ordered after the commit

Larry calls `redeem(500e18 worth of shares, ...)` — a permissionless call, no role needed.

`_requestWithdraw` files it into `withdrawAssets[P+1]` (because `currentPeriod()` already returns `P`), then `_logTrace` pushes the new total-assets value at key `block.timestamp`. That key **equals `pStart`**, so `Checkpoints.push` takes the in-place branch (`Checkpoints.sol:207-208`) and **overwrites** the period-start snapshot from 1000e18 down to **500e18**. No new checkpoint is appended; the old value is gone.
{% endstep %}

{% step %}

## Settlement proceeds against the stale figure

`settleCoverOrder` / `batchSettleCoverOrder` reads only the frozen `commit.totalAvailableCapacity` (`:507`) and `commit.totalDeclaredAllocated` (`:525`). It never re-derives capacity. **1000e18 of cover is sold.**
{% endstep %}

{% step %}

## Proof the commitment is now invalid

Call `recommitAllocation` with the *identical* numbers. It reverts `MatchingCapacityOverflow(1000e18, 500e18)` — the protocol's own validator rejects the commitment it accepted one transaction earlier.
{% endstep %}

{% step %}

## An incident lands in period `P`, and the payout is capped

`payout(receiver, amount, captureTimestamp)` reads `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)` (`FirelightVault.sol:911`) → **500e18**, and computes:

```solidity
paidAmount = min(amount, min(assetsAtCapturePeriod, activePayableAmount));
```

Claimants are paid **500e18 against 1000e18 of sold cover — a 500e18 shortfall.**
{% endstep %}

{% step %}

## The assets are still there

At payout time the vault holds the full 1000e18 (500e18 active + 500e18 in `withdrawAssets[P+1]`, which `payout:916` itself counts as exposed and payable). The shortfall is an accounting artifact, not undercollateralization.
{% endstep %}
{% endstepper %}

**Controls that isolate the cause:**

* Order the redeem *before* the commit in the same block → capacity reads 500e18, cover sold 500e18, **shortfall 0**.
* Move the identical redeem **one second later** → snapshot untouched at 1000e18, cover sold 1000e18, paid 1000e18, **shortfall 0**.

Only the same-second ordering opens the gap.

### Step-by-step — Impact B (no allocator involved)

{% stepper %}
{% step %}

## Period starts

Period `P` begins at `pStart`. Vault holds 1000e18; Larry owns 500e18 of it.
{% endstep %}

{% step %}

## Larry redeems at the period boundary

Larry calls `redeem` in the block whose `block.timestamp == pStart`. As above, the period-start snapshot is overwritten 1000e18 → 500e18, while his withdrawal sits in `withdrawAssets[P+1]` — which `payout` classifies as still payable.
{% endstep %}

{% step %}

## An incident is approved

An incident captured in period `P` is approved for 1000e18.
{% endstep %}

{% step %}

## Payout is capped

`payout` caps at `min(1000e18, min(500e18, 1000e18)) = 500e18`.

**Claimants receive 500e18 instead of 1000e18.** Larry recovers 250e18.
{% endstep %}
{% endstepper %}

Note this is not profitable for Larry: redeeming one second *earlier* would have paid him 500e18 and produced no shortfall at all. The boundary action costs him 250e18, which is why this is classified as griefing rather than theft.

### Automated proof (Foundry) — 11 tests, all passing

Two suites run against the **unmodified** in-scope contracts (`FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, `Checkpoints`, `Decimals`, `PriceFeed`), each deployed behind `ERC1967Proxy` exactly as in production. Only external collaborators are mocked: the ERC-20s, the Chainlink-shaped price feed, and `IncidentManager` (`payout` invoked directly under `PAYOUT_ROLE`). The Merkle tree in the backrun suite is a genuine single-leaf `StandardMerkleTree` — not a verification bypass.

```
Ran 7 tests for test/PeriodStartSnapshotRedeem.t.sol
[PASS] test_1_SameKeyPushOverwritesAndIsIncluded()
[PASS] test_2_A_Baseline_NoRedeem()
[PASS] test_2_B_Attack_RedeemExactlyAtPeriodStart()
[PASS] test_2_C_Control_RedeemOneSecondEarlier()
[PASS] test_2_D_Control_RedeemOneSecondLater()
[PASS] test_3_Delta_OneSecondFlipsTheOutcome()
[PASS] test_4_CapUnderstatesPayableExposure()

Ran 4 tests for test/CommitBackrunStaleCapacity.t.sol
[PASS] test_Backrun_CommitThenRedeemSameSecond()
[PASS] test_ControlA_RedeemBeforeCommit_StaysConsistent()
[PASS] test_ControlB_RedeemOneSecondLater_NoShortfall()
[PASS] test_ShortfallIsAccountingNotCollateral()

11 tests passed, 0 failed
```

`_liveMaxCapacity` in the backrun suite reads the allocator's own `_computeAvailableCapacity` by decoding the `MatchingCapacityOverflow(uint256,uint256)` revert payload, so the 1000e18 → 500e18 transition is the contract's arithmetic rather than the test's.

### To reproduce

```bash
git clone --depth 1 --branch v1_audit_ready \
  https://github.com/immunefi-team/audit-comp-firelight.git firelight-poc
cd firelight-poc

git clone --depth 1 --branch v5.2.0 \
  https://github.com/OpenZeppelin/openzeppelin-contracts.git lib/openzeppelin-contracts
git clone --depth 1 --branch v5.4.0 \
  https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable.git lib/openzeppelin-contracts-upgradeable
git clone --depth 1 --branch v1.9.4 \
  https://github.com/foundry-rs/forge-std.git lib/forge-std

# copy the attached foundry.toml, remappings.txt and the two test files into place
forge test --match-path "test/{PeriodStartSnapshotRedeem,CommitBackrunStaleCapacity}.t.sol" -vv
```

`foundry.toml` points `src` at `contracts/`, pins `solc = 0.8.28`, and enables `optimizer` + `via_ir` (required — `CoverOrderAllocator` hits "stack too deep" without it).

Full details and a per-test breakdown are in the attached `REPRODUCE.md`.
