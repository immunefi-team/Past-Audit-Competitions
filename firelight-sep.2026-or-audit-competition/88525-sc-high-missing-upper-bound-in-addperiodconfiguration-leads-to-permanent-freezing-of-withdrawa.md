# 88525 sc high missing upper bound in addperiodconfiguration leads to permanent freezing of withdrawals

**Submitted on Aug 13th 2026 at 16:22:41 UTC by @Griffinf for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88525
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Severity

Claimed Critical, "Permanent freezing of funds". Fallback High, "Temporary freezing of funds for at least 24 hours".

`addPeriodConfiguration` accepts any uint48 duration above the one day floor. A pending schedule cannot be cancelled or replaced, and once active the earliest correction epoch the contract accepts is two full durations away. So one transaction from PERIOD\_CONFIGURATION\_UPDATE\_ROLE freezes every withdrawal booked into or after the giant period for an attacker-chosen number of years. Our second test goes further: a duration near the uint48 maximum is also accepted, and the period-end arithmetic then overflows. No representable uint48 timestamp can advance the vault beyond the oversized period, and the contract's own configuration path cannot install a replacement; our test shows claims and corrections failing identically one hundred years later. Affected withdrawals therefore cannot become claimable under the current implementation, and normal withdrawal progression is permanently prevented once the oversized configuration becomes active. Recovery requires an implementation upgrade through the separate ProxyAdmin governance path, and the competition rules say emergency actions are not automatic grounds for a severity downgrade. If triage still applies a one-level reduction for the privileged starting position, the demonstrated ten year freeze alone sits far inside the High wording.

This is a role-exceeds finding under the competition's privileged-actor exception. The Macro trust model lists PERIOD\_CONFIGURATION\_UPDATE\_ROLE among the vault's lower-impact operational levers, attributed for period cadence. The missing bound and the irreversible scheduling turn that cadence authority into the power to freeze affected staker principal indefinitely, with no corrective action on the vault's own role surface.

## The bug

Duration validation is a lower bound and a granularity check only (FirelightVault.sol lines 1084 to 1086):

```solidity
if (newDuration < SMALLEST_PERIOD_DURATION || newDuration % SMALLEST_PERIOD_DURATION != 0)
    revert InvalidPeriodConfigurationDuration();
```

Withdrawals are period-bucketed and claimable only after the bucket's period fully ends (lines 791, 1008), so stretching one period stretches every claim horizon behind it. The guards close the escape routes: minimum notice is one period boundary but the role picks the scheduling moment (line 1093); a pending schedule cannot be replaced (lines 1090 to 1092, CurrentPeriodConfigurationNotLast) and there is no removal function; after activation the earliest corrective epoch is nextPeriodEnd(), two giant durations out (lines 360 to 363). Deposits and share transfers keep working throughout, so capital can enter but not leave.

The overflow arm makes it unrecoverable. Nothing computes epoch plus duration at schedule time, so a near-maximum duration is accepted; from the last short period onward nextPeriodEnd() overflows checked uint48 arithmetic and panics, and since addPeriodConfiguration calls it during validation, corrections are blocked by the no-cancel guard while the schedule is pending and by the panic once it is active. There is no window between the two. currentPeriodEnd() panics too, which also bricks cover sales (CoverOrderAllocator line 346). The same missing bound is open in the shrink direction down to one day, which can squeeze incident payout windows below what assessment needs.

## Impact

This contract is the V2 upgrade implementation for the Flare proxy at 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3: about 57.4 million FXRP across 4,651 holders, 1.14 million already in pending withdrawal buckets. One transaction from a role their own trust model calls low-impact halts withdrawal progression for that principal, for years in the finite case or permanently with the oversized configuration. The deployed Phase 1 vault already contains the identical entry point and validation, so the upgrade carries the defect forward rather than introducing it.

## Prior art

None of the three published known issues touches period configuration, and the Macro report does not flag this; it did fix the same missing-upper-bound class on two other privileged parameters (the settlement grace period cap and the capital adequacy ceiling). The deployed Phase 1 vault, included in this repository for upgrade review, carries the same setter with the same missing bound, and the V2 implementation under audit preserves it unchanged; no published audit finding or competition report flags the unbounded duration, the no-cancel semantics, or the overflow brick.

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/c1347660438c1e9648eab550db277388>

## Proof of Concept

The ProxyAdmin and DEFAULT\_ADMIN impersonation below is used only to reproduce the planned V2 deployment and normal role assignment on the local fork. The vulnerability itself begins after PERIOD\_CONFIGURATION\_UPDATE\_ROLE has been legitimately assigned; exploitation requires only one call within that role's ordinary period-configuration interface and does not require ProxyAdmin or DEFAULT\_ADMIN\_ROLE authority.

Attached: poc\_f2-1.js. Place at test/poc\_f2-1.js in a checkout of the commit above, then:

```
npm install
npx hardhat test test/poc_f2-1.js
```

The tests run on a Flare mainnet fork pinned to block 67321080 (the default public RPC is used; override with the FLARE\_RPC environment variable). Each test upgrades the real deployed proxy at 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 to the exact V2 implementation in this repository, on the fork only, through the protocol's own authorized actors: the ProxyAdmin owner performs the planned upgrade with initializeV2 and, as the vault's DEFAULT\_ADMIN, grants the period configuration role. All real storage is preserved through the upgrade, including 57.36 million FXRP of vault assets and the live one-day period configuration.

Test one schedules a 3650 day period with the minimum notice the contract accepts, proves the pending schedule cannot be replaced, freezes a real holder's withdrawal of about 898,900 FXRP at activation and a full year in, proves correction is rejected even at the giant period's own end, and proves the holder can claim only after the full ten years. Test two schedules the largest one-day-granular uint48 duration, proves the transaction is accepted, then proves nextPeriodEnd() and currentPeriodEnd() panic on checked overflow, the real holder's claim reverts, corrections fail under both guard regimes, and the state is unchanged one hundred years later. Both pass in about twenty seconds:

```
2 passing (19s)
```
