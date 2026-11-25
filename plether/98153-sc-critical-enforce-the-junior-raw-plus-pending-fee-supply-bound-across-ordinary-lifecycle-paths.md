# \#98153 [SC-Critical] Enforce the Junior raw-plus-pending-fee supply bound across ordinary lifecycle paths

**Submitted on Nov 5th 2026 at 14:29:49 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98153
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Classification and current-state boundary

This is one conditional **Security Best Practices Insight** about the optional Junior
maintenance fee. The release parameters at frozen commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8` list
`juniorMaintenanceFeeAprBps = 0`, disabled at deployment. `SECURITY.md` likewise says
the release deploys at `0 bps`, and the retained v2.0.0 Arbitrum Sepolia observation
measured `0 bps`. Pending maintenance-fee shares are therefore zero in that reviewed
state. The programme also says this test deployment's configuration is not the
production configuration.

I am not reporting a current Critical, High, Medium, or Low vulnerability, selecting a
vulnerability impact, or claiming that current production funds are at risk. The
programme says: "Issues reachable only through feature gates, flags, configurations or
networks that are not active on the listed deployment are out of scope." Its out-of-scope list also contains
"Best practice recommendations, except when submitted as Insights under the
Insight Reporting section." Its Insight rule says runnable code is not required and the
PoC must explain when the Insight is valuable. Immunefi defines a Security Best
Practices Insight as an improvement that strengthens security design or implementation
against potential threats even without an immediate or known vulnerability.
The programme states both provisions separately and does not specify which controls this
conditional preventive Insight. This report is submitted under the Security Best
Practices Insight route and does not claim that the inactive configuration is a
vulnerability or select any vulnerability impact.

## Missing joint supply invariant

At the frozen source commit,
`JuniorMaintenanceFeeMathLib.pendingFeeShares()` computes:

```solidity
uint256 retainedRay = retentionRay(aprBps, chargeableHours);
if (rawSupply == 0 || retainedRay == RAY) {
    return 0;
}

return Math.mulDiv(rawSupply, RAY - retainedRay, retainedRay, Math.Rounding.Floor);
```

In the retained 100 bps state that pending amount fits in `uint256`.
`TrancheVault.accruedTotalSupply()` then performs a separate checked addition:

```solidity
rawAndPendingSupply = totalSupply();
if (!IS_SENIOR) {
    rawAndPendingSupply += pendingMaintenanceFeeShares();
}
```

No lifecycle check enforces the joint domain:

```text
rawSupply + pendingFeeShares(rawSupply, rate, elapsedHours)
    + fixed downstream headroom <= type(uint256).max
```

The domain depends jointly on rate, elapsed fee hours, and raw supply. It matters to
ordinary lifecycle paths whenever a nonzero fee is active: Junior mint and burn paths
checkpoint the fee before changing supply,
the checkpoint mints fee shares through `super._update`, configuration finalization
checkpoints an old nonzero rate before installing the staged rate, share-pricing paths
add `VIRTUAL_SHARES = 1000`, and `HousePool` reads accrued Junior supply during whole-Pool
settlement.

The retained demonstration activated the allowed rate first and then grew raw supply
while the fee was active. Source analysis also shows the reverse order is unsafe: if a
future activation starts from raw supply outside the proposed rate/horizon domain,
ordinary checkpointing, supply changes, Pool settlement, and the owner-only zero-rate
finalizer can reach the same raw-plus-pending overflow condition.

## Exact recommendation

The owner-only rate change remains a trusted action and is not the finding. The
recommendation is to derive one exact joint-domain bound and enforce it in the ordinary
downstream supply and settlement consumers for every accepted state:

1. Derive the exact maximum raw supply from the configured rate and maximum supported
   catch-up horizon, reserving all fixed downstream headroom, including
   `VIRTUAL_SHARES`; use that shared bound in items 2 through 5.
2. After any due fee checkpoint and before every ordinary Junior supply increase, use a
   subtraction-first check such as `sharesToMint <= maxSafeRaw - totalSupply()`; do not
   first evaluate `totalSupply() + sharesToMint`.
3. Apply the same invariant to the checkpoint's own fee mint. Specify a bounded policy
   for excess accrued fees rather than allowing the checkpoint itself to revert.
4. During whole-Pool settlement, reject or defer an unsafe Junior deposit epoch through
   a non-reverting path instead of allowing that epoch to veto unrelated settlement.
5. Provide a documented zero-rate recovery that can exit the unsafe domain without
   first materializing an unrepresentable old-rate checkpoint, explicitly specifying
   whether exceptional accrued fee is capped or waived.
6. Add invariant/property coverage across Pool losses, asynchronous deposits,
   rate/hour boundaries, cross-tranche settlement, and zero-rate recovery.

An exact construction for the bound and its rounding, overflow-free whenever `retainedRay` is at least 10^24 (a retained fraction of 0.001), is supplied in the
PoC field.

## Preconditions, overlap, and limitations

- A trusted Pool owner must legitimately activate a nonzero fee through the documented
  proposal, 72-hour delay, and owner-only finalization lifecycle. A staged zero rate is
  subject to the same delay. No compromised role, malicious configuration, or bypass is
  claimed.
- The retained conditional demonstration uses ordinary protocol entrypoints after
  activation and modeled contract-accepted Pyth values and timestamps. It does not show
  that an ordinary user controls Pyth or that the modeled sequence is likely live.
- The retained raw state was deliberately constructed and is approximately
  `0.99997099 * 2^256`; it is not presented as a likely live supply.
- The evidence does not establish a current Arbitrum One deployment, current Pool
  population, or current funds exposed through this condition.
- The specifications already cap one fee checkpoint at 8,760 charged hours, forgive older
  backlog, and keep elapsed hours pending after a reverted settlement. None of these
  bounds the joint domain: at 100 bps the retained state is unsafe at hour 26, inside the
  cap, and a persistent revert keeps the hours pending.
- PR #72 (merged) added the fee formula, the vault integration, the specification text and
  the library test `test_MaxReachableSupplyCanAddTheFullYearFeeWithoutOverflow`, which
  checks one rate and full-year point and does not reserve the 1000-share pricing
  headroom. PR #127 (merged) raised the nominal APR cap from 1,000 to 10,000 bps and kept
  the max-uint-supply quote test at 1,000 bps. Those tests exercise the fee library at
  one point each; they do not enforce the joint domain across lifecycle transitions. The
  specifications describe the formula and deliberately require old-rate-first
  finalization. The known zero-share maintenance-fee
  limitation and Cecuro LOW-27 concern the opposite small-supply rounding condition,
  not this near-maximum raw-plus-pending overflow and recovery domain.
- The programme invalidates a report whose root cause was publicly disclosed in listed
  repositories before submission. Public items were last reviewed through 2026-11-04;
  private and concurrent submissions cannot be checked from public sources.
- The programme rewards only the best report of a given Insight; duplicate Insight
  submissions are invalid and not rewarded. No acceptance or reward is predicted.

The recommendation is valuable if the dormant setting may change, because it defines the
invariant ordinary paths must preserve whenever the fee is active. It does not describe a
deployment whose Junior fee is already nonzero and makes no current vulnerability claim.


## Proof of Concept

## Proof of Concept

## Conditions under which this Insight is valuable

This preventive recommendation is valuable if Plether may ever enable a nonzero Junior
maintenance fee. Its value requires:

1. a trusted Pool owner legitimately activates a nonzero rate through the documented
   proposal, 72-hour delay, and owner-only finalization lifecycle;
2. accepted Pool/Vault transitions, including ordinary deposit settlement, raise raw
   Junior share supply above the rate-and-time-specific safe bound while positive Junior
   principal remains;
3. any necessary external-market condition occurs; the retained demonstration uses
   modeled Pyth values and does not claim ordinary-user control or live likelihood;
4. enough complete fee hours elapse for the effective supply to exceed the supported
   domain; and
5. an ordinary path reads or checkpoints Junior effective supply, changes Junior
   supply, or settles the Pool.

Condition 1 is false in the reviewed release and retained deployment state: both are
`0 bps`. This report therefore claims no current vulnerability, severity, selected
vulnerability impact, or funds at risk.

## Deterministic arithmetic and exact guard

For fixed `retainedRay`, the implementation computes:

```text
pending = floor(raw * (RAY - retainedRay) / retainedRay)
effective = raw + pending = floor(raw * RAY / retainedRay)
```

If downstream pricing adds fixed headroom `H`, define:

```text
L = type(uint256).max - H
```

The exact largest jointly safe raw supply is:

```text
maxSafeRaw = floor(((L + 1) * retainedRay - 1) / RAY)
```

For TrancheVault pricing, `H = VIRTUAL_SHARES = 1000`. An implementation can avoid the
wide intermediate by computing:

```solidity
uint256 limit = type(uint256).max - VIRTUAL_SHARES;
uint256 q = Math.mulDiv(limit, retainedRay, RAY, Math.Rounding.Floor);
uint256 maxSafeRaw = q;
if (q != type(uint256).max
    && Math.mulDiv(q + 1, RAY, retainedRay, Math.Rounding.Floor) <= limit) {
    maxSafeRaw = q + 1;
}
```

After any due checkpoint, admit a supply increase only with subtraction-first arithmetic, with `maxSafeRaw` computed from the retention for the configured rate over the maximum supported catch-up horizon (8760 complete hours):

```solidity
if (totalSupply() > maxSafeRaw || sharesToMint > maxSafeRaw - totalSupply()) {
    // reject or defer this epoch without reverting whole-Pool settlement
}
```

At 100 bps and 26 complete hours, the exact-integer values are:

```text
retainedRay:
999970320058220522902660768

maxSafeRaw with 1000-share pricing headroom:
115788652534849107862278920397747535067303554773259679316828397679206758662857

effective(maxSafeRaw):
115792089237316195423570985008687907853269984665640564039457584007913129638935

effective(maxSafeRaw + 1):
115792089237316195423570985008687907853269984665640564039457584007913129638936
```

`effective(maxSafeRaw)` equals `type(uint256).max - 1000`; the upper neighbour is unsafe. For
retained fractions below one half, `pendingFeeShares` can revert inside `Math.mulDiv`
before the later addition. Proving that the pending amount alone fits is insufficient.

## Retained conditional evidence

The retained local-fork run started from the measured zero-fee v2.0.0 Arbitrum Sepolia
state at block `323664360`, verified all 30 listed deployment runtime hashes, and replaced
external Pyth/ArbSys dependencies only inside loopback Anvil. It legitimately activated
an allowed `100 bps` rate, then used public protocol entrypoints and a deliberately
constructed modeled market path. The resulting raw supply was about
`0.99997099 * 2^256`; this demonstrates reachability of the arithmetic domain, not live
likelihood.

At the first unsafe complete fee hour, hour 26, the run recorded the raw supply, pending fee shares, and remaining margin below; the last value is the raw supply minus the `maxSafeRaw` above:

```text
raw Junior supply:
115788730297860575040627627497612678923408193484103067030503761236470677184077

pending fee shares:
3436704775157717622590734455871726693446532516736282470989810730797138471

remaining uint256 margin (type(uint256).max minus raw Junior supply):
3358939455620382943357511075228929861791181537497008953822771442452455858

raw supply above pricing-safe bound:
77763011467178348707099865143856104638710843387713675363557263918521220
```

Here the pending fee shares exceed that margin, so raw plus pending exceeds
type(uint256).max. In that conditional state:

- Junior accrued-supply reads and whole-Pool settlement revert with a
  `Panic(uint256)` selector `0x4e487b71`; the retained output records the selector, not
  the encoded panic code;
- a distinct positive Senior redemption remains pending and unfunded;
- its cancellation reverts with `TrancheVault__RedeemCancellationUnavailable()` under
  the ordinary rule that a matured, funded, or refundable request cannot be cancelled;
  the Pool-only funding/refund paths cannot complete because settlement reverts; and
- finalizing a staged zero rate checkpoints the old rate and reverts. The 72-hour delay
  ended 46.3 hours after the first unsafe hour, leaving the `100 bps` rate and staged
  proposal in place.

Formula controls on the same raw supply remain safe at 100 bps after one complete hour
and 25 bps after 72 hours. The deployed Junior vault also read accrued supply
successfully at 100 bps after 25 complete hours. This isolates the joint
rate/time/raw-supply domain rather than treating a large raw supply alone as the issue.
A standalone compiled guard control rejected the 26-hour state; it was not a patched
Vault. No patched-target run is claimed or required for this Insight.

## Source and applicability boundary

The load-bearing source is the frozen competition tree, principally:

- `packages/perps/src/libraries/JuniorMaintenanceFeeMathLib.sol` for retained-fraction
  and pending-share arithmetic;
- `packages/perps/src/TrancheVault.sol` for checkpointing, effective-supply addition,
  fixed pricing headroom, supply mutation, and old-rate-first finalization; and
- `packages/perps/src/HousePool.sol` for whole-Pool consumption and deposit settlement.

The v2.0.0 release was built from commit
`1209ca8a5b94a4fb6323c7c4585846ed93db3932`; the programme states that the in-scope
`packages/perps/src` is identical at the frozen commit, and these three files are
byte-identical at both commits.

The programme's Insight route does not require runnable code; it requires this field to
state when the recommendation is valuable. The measurements above come from retained
local-fork evidence and were not regenerated for this report. Public items were last reviewed
through 2026-11-04; private and concurrent submissions cannot be checked from public
sources.

