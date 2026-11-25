# \#99192 [SC-Insight] ClosePreview.executionFeeAssets NatSpec says the fee is assessed, but the field returns the capped and credited value, which is zero when a VPI rebate absorbs the fee

**Submitted on Nov 12th 2026 at 10:43:32 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99192
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

`ClosePreview.executionFeeAssets` is documented in `interfaces/ICfdEngineTypes.sol:268` as "Protocol execution fee assessed on the close notional". The word assessed means the fee computed on the notional, 4 bps of close size, regardless of how it is settled. The implementation does not return that value: `CfdEngineLens.previewClose` (`:399`) copies `delta.executionFeeAssets`, which the planner caps at the assets actually eligible to pay it (`libraries/CfdEnginePlanLib.sol:1066-1068`): `executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets`, and `delta.executionFeeAssets = min(cs.executionFeeAssets, executionFeeEligibleAssets)`.

The divergence appears on skew-reducing closes, where the VPI rebate is larger than the fee. In that case `executionFeeEligibleAssets` is zero, so `delta.executionFeeAssets` is zero, and the preview reports zero fee for a close that actually incurs the 4 bps fee, absorbed into the VPI netting rather than charged.

The inconsistency is visible against the other read surfaces. `previewOpen` reports the full assessed fee in the same field even when the open is VPI-rebated, because the open path keeps the fee separate (`CfdEnginePlanLib.sol:399`). The order receipt (`CfdOrderPolicyEvaluatorBase.sol:431`) also reports the full assessed fee for the same close. Only the close preview reports the capped value, because it mirrors the planner's netted figure rather than the assessed charge.

The practical consequence is disclosure: an integrator that displays `executionFeeAssets` to a trader before closing shows zero fee for any skew-reducing close, while the trader's net payout already reflects the fee inside the VPI netting. The trader pays the same net amount either way, so this is a documentation and API-semantics defect rather than a value leak, but it is exactly the kind of read-surface mismatch that misleads integrations and support.

Recommendation: either rename the NatSpec to "Protocol execution fee credited on the close" and add a separate `assessedExecutionFeeAssets` field carrying the full 4 bps, or change `CfdEngineLens.sol:399` to copy `cs.executionFeeAssets` from `CloseAccountingLib` so the preview matches its documentation.

## Proof of Concept

This is a read-surface semantics insight, so the reproduction compares the preview against the receipt for the same close.

Step 1. Construct a skew-reducing close whose VPI rebate is larger than the 4 bps execution fee. On the release configuration the fee on a 100,000 notional close is 40.000000 USDG, and a rebate of that size is reachable because the VPI rebate is proportional to the skew reduction.

Step 2. Call CfdEngineLens.previewClose for that close and read executionFeeAssets. It returns 0, because CfdEnginePlanLib caps delta.executionFeeAssets at executionFeeEligibleAssets, and the eligible assets are zero when the rebate absorbs the charge.

Step 3. Execute the same close through the production path and read the order receipt. CfdOrderPolicyEvaluatorBase reports the full assessed fee (40.000000) for the same close, because the receipt field carries the assessed charge rather than the capped one.

Step 4. Compare with previewOpen on an open of the same size that is also VPI-rebated: the open preview reports the full assessed fee, not zero.

The observable inconsistency is therefore: two read surfaces for the same close disagree, and the documented meaning of the field (assessed) matches the receipt, not the close preview.

Suggested fix verification: after either renaming the field and adding assessedExecutionFeeAssets, or switching the copy source to cs.executionFeeAssets, the close preview and the receipt should report the same fee for the step 1 close.
