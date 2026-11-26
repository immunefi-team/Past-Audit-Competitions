# \#99199 [SC-Insight] IPletherOracle.PriceSnapshot.publishTime is documented as the earliest component publish time, but the implementation returns the newest

**Submitted on Nov 12th 2026 at 11:18:12 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99199
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

`IPletherOracle.PriceSnapshot.publishTime` is documented as the earliest component publish time, but `PletherOracle` returns the newest. The getter builds the snapshot at `PletherOracle.sol:874`, where the publish time is taken from the newest component rather than the oldest, so a caller who trusts the NatSpec and treats the value as the oldest data in this basket will underestimate the freshness of the basket, not overestimate it.

The distinction matters wherever the snapshot is used as a freshness input. A consumer that compares `publishTime` against its own staleness bound expects the value to be the conservative one, the oldest component, because that is the component most likely to be stale. Receiving the newest makes the basket look fresher than its weakest component, which is the opposite of the conservative reading.

No value moves on this path: the snapshot is a read surface, and the execution paths apply their own staleness and divergence checks against the components directly. The finding is therefore documentation and API semantics, the same class as the ClosePreview fee field, and it is worth correcting because integrations that display or gate on this field will be systematically optimistic.

Recommendation: either change the getter to return the earliest component publish time as documented, or correct the NatSpec to state that the field is the newest component publish time and, if the conservative value is useful, expose the earliest as a separate field.

## Proof of Concept

This is a read-surface semantics insight, so the reproduction compares the documented meaning with the returned value.

Step 1. Construct a basket whose components have different publish times, for example one at t0 and one at t0+10 seconds, and accept it.

Step 2. Call the snapshot getter on the oracle and read publishTime.

Step 3. Compare with the component publish times. The returned value equals the newest component (t0+10), while the NatSpec on IPletherOracle.PriceSnapshot describes the field as the earliest (t0).

Step 4. The consequence is directional: a consumer that treats the field as the conservative oldest-data bound will consider the basket fresher than its weakest component by up to the divergence window, which is exactly the wrong direction for a staleness gate.

Suggested fix verification: after the fix, either the returned value should equal the earliest component (t0) and match the NatSpec, or the NatSpec should say newest and the tests should assert t0+10 so the documented meaning is pinned.
