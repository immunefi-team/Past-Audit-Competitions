# \#97801 [SC-Medium] Make terminal close and liquidation accounting coalition-neutral when the trader controls the keeper

**Submitted on Nov 3rd 2026 at 09:14:47 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97801
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Insight

Plether applies individually documented accounting policies to voluntary close and
liquidation:

- voluntary full close releases the position's liquidation reserve into action-charge
  and carry collection;
- liquidation caps its charge by that reserve, credits the configured keeper share to
  the liquidation caller, and waives remaining terminally uncollectible charges; and
- liquidation is permissionless, so the position owner can call it directly or control
  the keeper address.

Their composition makes the trader's terminal settlement depend on the keeper's economic
identity.
From the same position state, a trader-controlled liquidation can retain value that a
valid voluntary close would route to protocol beneficiaries. An address-inequality check
would not restore the invariant because a second address can belong to the same economic
actor.

The ordinary carry/reserve case is the primary example. With 100,000 USDG of notional
(all amounts below are in USDG) and the release liquidation parameters, voluntary close
can collect the full 100-unit reserve for beneficiaries. Liquidation instead allocates 50 units to
beneficiaries and credits 50 units to the caller as keeper bounty. If the caller is the
trader, the coalition retains 50 units, equal to 5 basis points of notional.

The oracle-frozen case adds a second effect that does not depend on the keeper. Voluntary
close charges a 4-basis-point execution fee plus a 50-basis-point frozen-close spread, or
540 units on 100,000 units of notional. Liquidation charges 10 basis points, credits half
of that charge to the caller, applies a 2.5-basis-point adverse-confidence shift under the
release limits, and does not assess the frozen-close spread (`SECURITY.md:843`,
`ACCOUNTING_SPEC.md:1188`). The trader-plus-keeper coalition therefore bears 75 units
rather than 540, a 465-unit difference equal to 46.5 basis points of notional. Only 50
of those 465 units depend on who calls the liquidation: if an unrelated address calls it,
the trader bears 125 units. The other 415 units are the 500-unit spread that liquidation
does not assess, less the 85 units by which its charge and shift (125) exceed the 40-unit
execution fee, so an address rule alone would not remove them.

This is a Security Best Practices Insight, not a Critical, High, Medium, or Low
vulnerability claim. The accounting specification states that pending carry is trader
debt rather than an LP-owned receivable, and it documents the liquidation waiver and
keeper allocation. `SECURITY.md:845` describes the frozen-market pricing split as a
deliberate trade-off, and `ACCOUNTING_SPEC.md:1007` states that close and liquidation
should share the same economic assumptions wherever the question is identical and differ
only where the product deliberately differs. The issue is the end-to-end coalition
incentive created by combining those policies, not theft of already-owned LP yield.

## Proof of Concept

## Proof of Concept

## Reproduction from source and release parameters

Use frozen competition commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. In `OrderRouter.sol:486-500`, observe that liquidation is permissionless.
2. In `OrderLiquidationBatchLogic.sol:701-729`, observe that `msg.sender` is forwarded as
   the keeper.
3. In `CfdEnginePlanLib.sol:1093-1133`, observe that a full close releases the liquidation
   reserve into action-charge collection.
4. In `CfdEnginePlanLib.sol:1475-1481` and `1553-1609`, compare liquidation: the reserve
   is kept out of carry collection and residual action charges are waived.
5. In `LiquidationAccountingLib.sol:62-78` and
   `MarginClearinghouse.sol:1234-1269`, trace the liquidation charge split and keeper
   credit.
6. Apply the checked-in release values from `packages/perps/DEPLOYMENT.md:128-156`
   (`bountyBps` 10, `keeperShareBps` 5,000, `protocolShareBps` 0, `executionFeeBps` 4,
   `frozenCloseSpreadBps` 50, `basketMaxConfidenceRatioBps` 10,
   `adverseConfidenceMultiplierBps` 2,500). Amounts are USDG. The example position has
   100,000 USDG of notional and a 100 USDG liquidation reserve, the larger of notional
   times `bountyBps` and `minBountyAssets` (`OpenAccountingLib.sol:148-152`):

```text
notional                                      = 100,000
liquidation charge                            = 100,000 * 10 / 10,000 = 100
keeper bounty                                 = 100 * 5,000 / 10,000 = 50
ordinary beneficiary amount on liquidation   = 100 - 50 = 50
ordinary beneficiary amount on full close    = 100
controlled-keeper coalition difference        = 100 - 50 = 50 = 5 bps

frozen voluntary-close charge                 = 100,000 * (4 + 50) / 10,000 = 540
maximum adverse-confidence shift              = 10 bps * 25% = 2.5 bps = 25
controlled-liquidation coalition cost         = (100 - 50) + 25 = 75
frozen coalition difference                   = 540 - 75 = 465 = 46.5 bps
```

7. Confirm the ownership boundary in `ACCOUNTING_SPEC.md:731-740` and `1292`: pending
   carry is a trader obligation, while LP ownership follows collection. Confirm the
   intended liquidation policy in `ACCOUNTING_SPEC.md:1162-1189`.

The arithmetic uses only the source formulas and checked-in release configuration. It
isolates the execution fee, frozen-close spread, liquidation charge, keeper share and
confidence shift, and it assumes signed VPI is zero in both branches, carry is the same,
the account is liquidatable, the liquidation reserve covers the full charge, the account
can pay the full voluntary-close charge, and, in the ordinary example, unpaid carry
absorbs the whole reserve. With the release VPI factor the voluntary close also settles a
signed VPI term, which moves the frozen difference by that amount.

## Conditions in which the recommendation is valuable

The ordinary route matters when:

- a position has accrued carry or action charges;
- both voluntary full close and liquidation are available from the same economic state,
  which requires the account to be liquidatable (equity at or below the maintenance
  requirement, 3% of notional in the Friday Afternoon Deleverage (FAD) window, which
  contains the oracle-frozen period, and 0.1% otherwise, or carry left uncovered);
  liquidating a healthy account reverts;
- the liquidation reserve is the remaining collectible source, and the charges still
  unpaid after the pledge is exhausted (carry, the execution fee, any VPI charge) are at
  least the reserve, 100 units in the example; and
- the trader can invoke the permissionless liquidation path directly or through a
  controlled keeper.

The frozen route adds two conditions: the oracle is frozen and a valid voluntary close
is available or already committed. The recommendation is especially useful at the FAD
margin boundary (3% of notional), where ordinary carry or a policy-valid oracle price
move—an update accepted through the configured Router/oracle validation path—can make the
account liquidatable after the voluntary close path has been selected. A close committed
before the freeze but finalized after it is assessed the spread, because the state at
execution governs.

## Recommendation

Make the terminal accounting invariant independent of the keeper address:

1. Preserve the voluntary-close protocol and LP collection floor before paying a keeper
   bounty when liquidation substitutes for an available committed full close, without
   letting a queued close remove the keeper's incentive to liquidate (a close carries no
   execution bounty); or
2. Apply a liquidation charge floor that prevents the position owner plus any controlled
   keeper from retaining more than the voluntary-close branch from the same state.

Keep liquidation liveness and the protected-claim boundary intact. Add exact-snapshot
regressions for direct self-liquidation and second-address liquidation in both the
ordinary carry/reserve case and the FAD/frozen boundary case. Each regression should
compare coalition balances and LP-plus-treasury collections against the precommitted
voluntary close, including confidence, VPI, carry, execution-fee, spread, liquidation,
and keeper-share effects.

