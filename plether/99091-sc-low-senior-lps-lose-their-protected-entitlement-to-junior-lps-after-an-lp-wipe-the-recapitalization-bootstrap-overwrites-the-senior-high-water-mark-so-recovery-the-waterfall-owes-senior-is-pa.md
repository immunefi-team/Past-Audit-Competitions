# \#99091 [SC-Low] Senior LPs lose their protected entitlement to Junior LPs: after an LP wipe, the recapitalization bootstrap overwrites the Senior high-water mark, so recovery the waterfall owes Senior is paid to Junior

**Submitted on Nov 11th 2026 at 20:22:55 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99091
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The waterfall protects Senior through its high-water mark H. WHITEPAPER §4.1 states: "3. Loss does not reduce H. 4. Later revenue first restores S to H. 5. Revenue remaining after senior restoration accrues to junior." ACCOUNTING_SPEC.md:331-338 says the same, and :449-451 says claimant recapitalization exists to "restore an existing protected senior entitlement".

When the owner follows the documented terminal-deficit runbook (SECURITY.md:1500-1509: recapitalize, then clear degraded mode), the settlement that applies the recap bucket takes a different branch if LP equity is ≤ 0 at the settlement mark:

```solidity
// HousePoolPendingPreviewLib.sol:117-122 (applyClaimantRecapitalizationIntent)
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

This **assigns** H from the recap amount, discarding the Senior entitlement that losses were never supposed to reduce. Every later recovery above the new, smaller H goes to Junior. Junior is the first-loss tranche and was just wiped, yet it is paid before Senior's lost principal is restored. That is the reverse of the documented seniority.

The path: `HousePool._reconcile` (:1598) → `_applyPendingClaimantBucketsLive` (:1768-1770) →
`HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview` (:47) → `applyClaimantRecapitalizationIntent` (:117-122).

## Who triggers it, and who gains
- **Trigger:** the settlement that first applies the recap.
  - **Live market:** any keeper's `OrderRouter.settleLpEpoch(bytes[])` with a valid tick at which LP equity is ≤ 0.
  - **Frozen market:** **anyone**, via the permissionless `HousePool.settleLpEpoch(0, 0)` on the cached mark (HousePool.sol:809).
- **Gainers:** Junior holders. A Junior LP who settles at the wiped mark and redeems after recovery collects value owed to Senior.
- **The owner can't avoid it by choosing the recap amount:**
  - recap below the old H, and Senior loses (main test);
  - recap above it, and H is set ABOVE the old entitlement, so Junior loses instead (`test_RecapBootstrapAboveOldHwm`: H 3,875,508 vs 3,002,712).

  Either way, the split between tranches depends on whether a keeper's valid price puts equity just below or just above zero.

## Proof of concept (release params; Senior 3.0M, Junior 1.016M; production routes)
- **Market path:**
  1. A hedged book is opened at 35% of depth per leg.
  2. The basket moves to 1.90 and the winning shorts close and are paid; degraded mode latches.
  3. The basket moves to 0.30, at which point LP equity is ≤ 0 (deficit 432,148).
- **Owner runbook:** recapitalize 1,375,508 (= L − E + 100k) and clear degraded mode.
- **The only difference between branches** is the keeper's valid settlement price in the same epoch:

| | A: keeper settles at 0.30 | B: keeper settles at 0.55 |
|---|---|---|
| Senior H after the recap settle | **1,375,508** (old H 3,000,355) | 3,002,712 |
| After the basket returns to 1.00 (flat book): S / J / U | 1,375,508 / **1,097,581** / 432,148 | 2,907,404 / 0 / 0 |
| Senior LP (SEN1) value | **1,334,243** | 2,820,183 |
| Junior LP (JUN1) withdraws | **753,689 USDG** | 0 |

- **Net transfer:** Senior loses 1,531,896; Junior gains 1,097,581, and the rest goes to unassigned.
- **Frozen variant (`test_FrozenPermissionlessSettleOverwritesSeniorHwm`):** the owner recaps on Friday after the freeze. On Saturday, **any address** calls `pool.settleLpEpoch(0, 0)` on the cached mark, and H drops from 3,000,355 to 1,375,508. After the Sunday reopen at 1.00, Junior principal is 1,097,581, all created from Senior's entitlement. No keeper price choice is involved.

## Precondition
The branch needs LP equity ≤ 0 at the settlement that applies the recap. That is the terminal-deficit state the protocol plans for, with its own runbook (SECURITY.md:1500-1509) and recapitalization path. The PoC reaches it with a large two-way basket move on a hedged book. The finding is not that a wipe is likely: once the documented recovery runs, the protocol's own recovery accounting moves Senior principal to Junior and breaks the seniority it promises.

## Not a known or filed issue
- **Not #97760:** that one is a pending recap not reduced when its cash pays trader claims. Here there are no trader claims in any state (claims = 0 throughout).
- **Not #97968:** that one is a queued recap applied after same-pass revenue, which strands Junior revenue in unassigned. Here the cause is the HWM assignment in the bootstrap branch, and the effect is Junior taking Senior's restoration.
- **Not Cecuro MED-12:** MED-12 ("mixed recovery inflows can strand LP revenue in ownerless pool assets") is about revenue continuation after a recap. Its regression test (HousePoolSnapshotParity.t.sol:135-153) wipes a pool whose Senior HWM is only the 1,000 seed, then recaps 40,000. The recap is above the old HWM, so the overwrite never shows. A sibling test asserts "recapitalization preserves its recovery target" (:193), which is the behaviour this report shows failing when the existing HWM is larger than the recap.
- **Not in Known Limitations,** the AI scan, the pre-audit or Cecuro.

## Recommended fix
In the bootstrap branch, keep the existing entitlement: add the recap to Senior principal and set
`seniorHighWaterMark = max(existingHighWaterMark, principalAfter)`. Equivalently, route a recap with both principals at
zero through the same "restore Senior up to H, then unassigned" rule as any other recap. Junior must never receive
recovery before Senior is back at its pre-loss H.



## Link to Proof of Concept

https://gist.github.com/chj2934/7aaf6cd560347c611e9312e8bd6bdfda

## Proof of Concept

## Proof of concept
PoC (secret gist): https://gist.github.com/chj2934/7aaf6cd560347c611e9312e8bd6bdfda

Save all four files in `packages/perps/test/poc/hwm/` (any folder two levels below `test/`) and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/hwm forge test --dynamic-test-linking --match-path "test/poc/hwm/*.t.sol" -vv --gas-limit 100000000000
```

Result: 3 passed.

| Test | Shows |
|---|---|
| `test_RecapBootstrapOverwritesSeniorHwm` | Branches A/B above: H overwritten, Senior −1,531,896, JUN1 withdraws 753,689 |
| `test_FrozenPermissionlessSettleOverwritesSeniorHwm` | Any address's frozen settle overwrites H; Junior principal 1,097,581 created from Senior's entitlement |
| `test_RecapBootstrapAboveOldHwm` | An over-recap sets H above the old entitlement, so no recap amount is correct under this code |

The harness (`Recov20Base.sol`, `B59Base.sol`) inherits the vendor's `BasePerpTest` with release risk parameters. Trades go through `commitOrder`/`executeOrder`, marks through `updateMarkPrice`, and settlements through `settleLpEpoch`, each called by an unrelated keeper or any address. The only owner calls are the documented runbook (`recapitalizePool`, `clearDegradedMode`). There are no storage writes.

