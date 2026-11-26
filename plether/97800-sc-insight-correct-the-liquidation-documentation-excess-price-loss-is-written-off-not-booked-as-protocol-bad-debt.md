# \#97800 [SC-Insight] Correct the liquidation documentation: excess price loss is written off, not booked as protocol bad debt

**Submitted on Nov 3rd 2026 at 09:09:47 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97800
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

Plether's public liquidation documentation classifies uncovered liquidation loss as protocol bad debt. The frozen implementation and accounting specification instead treat terminal price loss beyond the account's same-account claim and PnL pledge as a diagnostic write-off; they do not book it as protocol debt or an LP deficit. Risk monitors, LP dashboards, integrators, and incident procedures can therefore look for or respond to a bad-debt balance that the pinned protocol does not record.

## Vulnerability Details

The public Plether Docs liquidation page describes uncovered liquidation loss as protocol bad debt in two places:

- step 9 of “What happens during liquidation” says any uncovered shortfall becomes protocol bad debt; and
- “Liquidation does not necessarily consume everything” says the remaining uncovered loss becomes bad debt borne by the LP waterfall.

The same classification recurs, in different wording, on other pages of the docs, for example “Risks you should understand first” and “The Plether Perps market model”; the correction below applies to each.

The frozen implementation and its accounting specification instead treat terminal price loss beyond the account's same-account claim and PnL pledge as a diagnostic write-off. They do not book it as protocol debt or an LP deficit.

## Impact Details

This distinction is security-relevant documentation, not wording style. The repository's security notes say that price loss above the cap “is not socialized as LP debt or terminal deficit” (`packages/perps/SECURITY.md:1239-1240`) and that “the current accounting model has no accumulated-debt ledger to clear” (`:1502-1503`). The docs instead describe an “accumulated bad debt” counter and call it a “recapitalization target” (`how-plether-works/the-liquidity-pool-and-tranche-waterfall` on the same site), and the pinned engine has no such getter or clearing function.

Risk monitors, LP dashboards, integrators, and incident procedures that rely on the docs can look for, or respond to, a bad-debt balance that the pinned protocol does not record, and can deduct an uncollectible price loss that pool value does not count as an asset.

## References

- https://docs.plether.com/how-plether-works/margin-leverage-and-liquidation
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1475-L1522
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1142-L1152
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L857-L859
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineSettlementSidecar.sol#L23-L25


## Proof of Concept

## Proof of Concept

## Reproduction from documentation and source

Use competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. Open `https://docs.plether.com/how-plether-works/margin-leverage-and-liquidation` (text as
   captured on 2026-11-02). Under “What happens during liquidation”, step 9 reads: “Any uncovered
   shortfall becomes protocol bad debt.” Under “Liquidation does not necessarily consume
   everything”, the last bullet reads: “only the remaining uncovered loss becomes bad debt borne
   by the LP waterfall.” Both classify the uncovered remainder as bad debt.
2. In `packages/perps/src/libraries/CfdEnginePlanLib.sol:1512-1522`, follow a liquidation
   with negative price PnL. The planner first consumes the same-account trader claim,
   then consumes available PnL pledge, and assigns only the remainder to
   `priceLossWrittenOffAssets`.
3. In the same file at `1475-1481`, confirm the stated invariant: excess price loss is a
   diagnostic write-off, action-charge remainder is waived, and no shortfall becomes
   protocol debt.
4. In `packages/perps/ACCOUNTING_SPEC.md:1142-1152`, confirm that uncollateralized price
   loss above the cap is reported as a diagnostic write-off rather than protocol debt or
   terminal deficit.
5. In `packages/perps/src/CfdEngineSettlementSidecar.sol:857-859`, observe that the live
   settlement path emits `PriceLossWrittenOff` for this amount and writes no debt balance. The
   event is declared in `packages/perps/src/interfaces/ICfdEngineSettlementSidecar.sol:23-25`
   as raw price loss above the claim-plus-pledge cap: “This amount was never an LP asset and
   therefore is not protocol bad debt.”

Conditions under which this correction is valuable: a reader acts on the liquidation page (a
monitor, dashboard, integration or incident procedure) and the discrepancy appears when
liquidation price loss exceeds the account's nettable same-account claim plus its PnL pledge
after carry. The planner consumes the claim first, then the pledge, and writes off only the
excess, so a loss equal to claim plus pledge writes off nothing and each further unit of loss is
written off one for one. Example: a 50,000 USDG price loss against a 930 USDG pledge and no claim
writes off 49,070 USDG. The same two statements, read literally, also cover an uncollectible
carry or negative-VPI remainder, which the planner waives rather than books. The discrepancy
does not concern a newly created trader claim for an unfunded positive price gain, the
liquidation bounty, or an actual liabilities-over-assets state, which the protocol records
separately as trader claims and terminal deficit.

## Recommendation

Update both public statements to say that terminal price loss above the nettable claim
and PnL-pledge cap is reported as a diagnostic price-loss write-off. It does not create
protocol debt or an LP deficit in the current accounting model, and liquidating the account does
not itself reduce pool value or tranche principal: pool value already carries the full marked
liability to winning traders and counts a losing account's loss only up to what that account can
pay.

Document newly created trader claims for unfunded positive price gains and action-charge
waivers separately, and reserve “bad debt” for a state the implementation actually
records as a protocol liability or LP deficit. This keeps the public liquidation flow
aligned with the planner, the `PriceLossWrittenOff` settlement event, the accounting
specification, and the emergency procedures in `packages/perps/SECURITY.md`.
