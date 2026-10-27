# 89222 sc medium a withdrawal at the exact period start second halves the cover payout

**Submitted on Aug 17th 2026 at 21:58:41 UTC by @UptightChipmunk990 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89222
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`payout` uses two different conventions for the same instant, and they disagree on exactly one second.

The payable pool deliberately includes assets withdrawn *during* the capture period — the code says why at `:918-920`: *"otherwise exposed assets could escape by withdrawing first."* But the cap on that same payout is `totalAssetsAt(capturePeriodStart)`, an **inclusive** `<= key` lookup.

A withdrawal executed at exactly `capturePeriodStart` is therefore counted as inside the period for bucketing (so its assets stay payable) and as at-or-before the period start for the snapshot (so it rewrites the cap downward). One LP shifting their transaction by one second halves what the cover buyer is paid.

### Vulnerability Details

`contracts/core/FirelightVault.sol:911`:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
```

`totalAssetsAt` resolves through `Checkpoints.upperLookupRecent` (`contracts/core/lib/Checkpoints.sol:196`), which is an inclusive upper bound — it returns the value of the newest checkpoint with `key <= capturePeriodStart`.

`withdraw`/`redeem` both call `_logTrace(..., true)`, so a withdrawal pushes a checkpoint at `block.timestamp`. When `block.timestamp == capturePeriodStart`, that push lands on the same key as the period-start snapshot and **overwrites it** rather than appending after it.

Meanwhile `_requestWithdraw` (`:1008`) buckets the same withdrawal into `currentPeriod() + 1`, and `payout` at `:916` adds `withdrawAssets[capturePeriod + 1]` back into the payable pool precisely so those assets cannot escape.

So the assets are simultaneously counted as still-exposed (payable) and as never-having-been-there (snapshot). The cap wins, and the payout is capped at the post-withdrawal figure.

Period starts are day-aligned and publicly derivable from `currentPeriodEnd()`, so the target second is deterministic and known in advance.

### Impact Details

A cover buyer is under-paid, permanently, by the fraction of vault assets that one LP moves at that instant.

Measured in the PoC — Alice and Bob each deposit 1000e6, the assessor requests the full 2000e6:

|                                   | snapshot `totalAssetsAt(ps)` |  payable pool |          **paid** |
| --------------------------------- | ---------------------------: | ------------: | ----------------: |
| baseline, no boundary transaction |                2 000 000 000 | 2 000 000 000 | **2 000 000 000** |
| control — withdraw at `ps + 1`    |                2 000 000 000 | 2 000 000 000 | **2 000 000 000** |
| exploit — withdraw at `ps`        |            **1 000 000 000** | 2 000 000 000 | **1 000 000 000** |

The control is the important row: the identical withdrawal one second later pays in full. The only variable is the timestamp.

The shortfall is final. `IncidentManager.sol:423-434` calls `vault.payout` once and immediately sets `incident.status = CLOSED`, so nothing can top it up.

The same lookup also feeds `CoverOrderAllocator._computeAvailableCapacity` (`CoverOrderAllocator.sol:711`), so a boundary transaction additionally mis-states that period's sellable capacity.

**Preconditions.** A vault transaction must land in the block whose timestamp is exactly `currentPeriodStart()` of the capture period. On a \~2-second-block chain an LP can target it, and it is retryable every period at gas cost. It can also fire by accident, which is worth noting since the LP causing it need not intend anything.

**On severity.** I am claiming Medium. The impact is a large, irreversible under-payment of a real obligation, but it requires hitting an exact second and the LP who triggers it is not paid extra — they take their pro-rata share of the reduced payout, and the retained assets benefit all LPs including non-participants. That is value misdirected from the covered party to the vault rather than theft by the actor, which is what keeps me from arguing High.

One counter-argument I want to pre-empt: it could be read as intended, on the view that the cap is meant to be "last write at or before period start." But then `:916` should not add `withdrawAssets[capturePeriod + 1]` back into the payable pool. The code holds both positions simultaneously, and the control test shows a one-second shift changes the payout by 2×.

## Recommendation

Make the two conventions agree. The snapshot lookup should exclude writes made at the period-start instant itself, since the bucketing logic treats that instant as inside the period:

* use a strict `< capturePeriodStart` lookup for the exposure cap, or
* capture the period-start snapshot in a dedicated slot written once at period rollover, so a same-second user transaction cannot overwrite it.

The second is preferable — it removes the class rather than one boundary, and it also stabilises `CoverOrderAllocator._computeAvailableCapacity`, which reads the same value.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L911\\>
— assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart), the payout cap

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L916\\>
— withdrawAssets\[capturePeriod + 1] added back to the payable pool

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L918-L920\\>
— "otherwise exposed assets could escape by withdrawing first"

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L1008\\>
— \_requestWithdraw buckets into currentPeriod() + 1

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol#L196\\>
— upperLookupRecent, inclusive <= key

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol#L711\\>
— \_computeAvailableCapacity reads the same lookup

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/IncidentManager.sol#L423-L434\\>
— payout called once, incident set CLOSED

No published Known Issue or "Notes for researchers" item covers the period-start snapshot convention, the `Checkpoints` lookup boundary, or the payout exposure cap. The Firelight A-1 audit contains no finding in this area either — its only `payout()` finding is G-1, a gas note about scanning `periodConfigurations` twice.

## Link to Proof of Concept

<https://gist.github.com/Manimal777/f080190a84bed8be2db7c0c779c7b518>

## Proof of Concept

Place `Base.t.sol` and `BoundarySnapshot.t.sol` in `poc/` and run:

```shell
forge test --offline --match-contract BoundarySnapshot -vv
```

`Base.t.sol` deploys `FirelightVault` behind an ERC1967 proxy with the initializer arguments from `test/setup/fixtures.js`.

```
[PASS] test_baseline_fullClaimPaid()
[PASS] test_control_withdrawOneSecondLater_fullClaimPaid()
[PASS] test_exploit_withdrawAtBoundary_halvesPayout()
```

Sequence for the exploit case: two 1000e6 deposits in period 0; warp to exactly `currentPeriodEnd()` (which is period 1's start); Alice withdraws 1000e6 in that block; assert her assets are still in the payable bucket while the snapshot has dropped to 1000e6; the assessor requests 2000e6 and receives 1000e6.
