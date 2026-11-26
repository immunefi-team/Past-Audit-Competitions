# \#100103 [SC-Insight] RiskParams occupies ten storage words, though seven of its ten fields are tightly bounded enough to pack into one

**Submitted on Nov 16th 2026 at 08:23:06 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100103
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

## Summary

`CfdTypes.RiskParams` (`CfdTypes.sol:104-115`) declares ten `uint256` fields, and the engine stores its active copy
in `_activeRiskParams` (`CfdEngine.sol:126`), so every field occupies a full 32-byte storage word: ten words in
total. The public getter `riskParams()` (`CfdEngine.sol:971-997`) is hand-written assembly that loops over those
ten words and issues one `sload` per word, returning the full 320-byte tuple. This getter is not an occasional
read: it is called from the settlement sidecar (`CfdEngineSettlementSidecar.sol:231`), the order-policy evaluator
(`CfdOrderPolicyEvaluatorBase.sol:227`), the position-protection book (`PositionProtectionBook.sol:948`), and the
account and protocol lenses (`PerpsPublicLens.sol:438`, among others), so its ten-word cost recurs across commit,
execution, withdrawal, protection, and liquidation paths, not just in one place.

**Seven of the ten fields are tightly bounded by validation already in the pinned source; three are not.**
`CfdEngine`'s constructor (`CfdEngine.sol:307-327`) and `CfdEngineAdmin._validateRiskParams`
(`CfdEngineAdmin.sol:254-277`) enforce, before any value reaches storage:

- `maxSkewRatio` <= `CfdMath.WAD` (1e18) - needs at most 60 bits.
- `baseCarryBps` <= 100,000 - needs at most 17 bits.
- `maintMarginBps`, `initMarginBps`, `fadMarginBps`, `keeperShareBps`, `protocolShareBps` each <= 10,000 - each
  needs at most 14 bits.

Summed, those seven fields need at most 147 bits (60 + 17 + 14*5) - well under one 256-bit word. The remaining
three fields - `vpiFactor`, `bountyBps`, and `minBountyAssets` - have no codified upper bound at either validation
site (`CfdEngineAdmin.sol:100-107`'s own natspec says so directly: "There is no admin-side upper bound on
`vpiFactor` or `bountyBps`"; `minBountyAssets` is checked only for non-zero), so each must keep a full word to
represent every value the validator still accepts.

**Concrete packing plan.** Replace the ten-word storage representation with a four-word one: `vpiFactor`,
`bountyBps`, and `minBountyAssets` each keep their own full `uint256` word (three words, unchanged range). The
other seven fields move into a fourth word as right-sized unsigned integers that exactly cover their validated
range and no less: `maxSkewRatio` as `uint64` (64 bits, covers the 60 needed), `baseCarryBps` as `uint32` (32
bits, covers the 17 needed), and `maintMarginBps`, `initMarginBps`, `fadMarginBps`, `keeperShareBps`,
`protocolShareBps` each as `uint16` (16 bits apiece, covers the 14 needed). That fourth word uses 176 of its 256
bits (64 + 32 + 16*5); the compiler packs all seven into one slot automatically because the running total never
exceeds 256 bits. No field's representable range shrinks relative to what validation already accepts: every
value the constructor or the admin's proposal validator would accept today still fits after packing, exactly.

**The external contract is unchanged.** `CfdTypes.RiskParams` (the type every other file that touches it keeps
using - the settlement sidecar, the policy evaluator, the protection book, the lenses, and the planning
libraries, eleven files besides `CfdEngine.sol` itself) keeps its current ten `uint256` fields, in its current
order; only the one physical storage
location inside `CfdEngine.sol` changes shape. `riskParams()` keeps its exact signature and returns the same ten
named `uint256` values in the same order, decoded back out of the packed representation before being returned.
Every caller, both on-chain and off-chain, observes identical behavior.

## References

All paths are under `packages/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`,
`immunefi-team/audit-comp-plether`.

- `src/CfdTypes.sol:104-115` - the `RiskParams` struct definition.
- `src/CfdEngine.sol:126` - the `_activeRiskParams` storage declaration.
- `src/CfdEngine.sol:307-327` - constructor-time validation bounds on every field.
- `src/CfdEngine.sol:971-997` - the `riskParams()` getter.
- `src/CfdEngineAdmin.sol:105` - natspec stating no admin-side upper bound exists for `vpiFactor` or `bountyBps`.
- `src/CfdEngineAdmin.sol:254-277` - proposal-time validation bounds (`_validateRiskParams`), mirroring the
  constructor's bounds.
- `src/CfdEngineSettlementSidecar.sol:231` - a `riskParams()` call site.
- `src/CfdOrderPolicyEvaluatorBase.sol:227` - a `riskParams()` call site.
- `src/PositionProtectionBook.sol:948` - a `riskParams()` call site.
- `src/PerpsPublicLens.sol:438` - a `riskParams()` call site.
- Plether Audit Competition Information tab, Insight Reporting rules and the Code Optimizations and Enhancements
  category definition; Scope tab PoC-requirement treatment for Insights.


## Proof of Concept

# Proof of Concept

Conditions under which this Insight is valuable: a fresh deployment, where the engine's storage layout can still be
chosen before any instance is live and holding real funds. The engine has no proxy or migration path, so a storage
layout is fixed for the lifetime of a given deployed instance; the saving applies to every transaction that reads
the risk parameters afterward, because each first read of a given word in a transaction pays the full cold-access
cost for that word.

Read-only source demonstration at the pinned commit: `CfdTypes.sol:104-115` shows the ten `uint256` fields of
`RiskParams`. `CfdEngine.sol:971-997` shows the getter's assembly loop, which runs for ten 32-byte strides and
issues one `sload` per stride. `CfdEngine.sol:307-327` and `CfdEngineAdmin.sol:254-277` show the validation bounds
each field is held to before it reaches storage, and `CfdEngineAdmin.sol:105` states directly that `vpiFactor` and
`bountyBps` have no admin-side upper bound. The getter's callers - the settlement sidecar, the order-policy
evaluator, the position-protection book, and the public lenses - are each a direct, grep-confirmed call to
`riskParams()` on the engine.

Measured confirmation: the packing plan described in this report was implemented against an isolated local copy
of the pinned source (same `packages/perps` tree, same compiler version, optimizer, and `via_ir` setting as the
project's own build profile) and compared to the unmodified baseline with a matched Foundry build of both. Every
value accepted by the current validator - including every field pushed to its exact validated maximum, and the
three uncapped fields pushed to the maximum `uint256` value - round-trips through the packed representation and
back out of the getter unchanged; no value the validator accepts is truncated by the narrower in-slot field
widths, because each width was sized to the validated bound, not guessed. With that packing in place, the
getter's average measured gas cost (Foundry's standard per-function gas accounting, a deterministic, non-fuzzed
call with no variance across repeated calls) fell from 23,530 gas to 10,721 gas - a reduction of 12,809 gas, about
54%, on every call. That reduction is consistent with removing six of the getter's ten word-reads: roughly 2,135
gas per removed read, in the range of a single avoided cold storage read under the project's own configuration,
plus the small loop overhead the original assembly also paid for those six iterations and no longer needs to.

The same matched build also measured the engine's compiled runtime bytecode size, since that is the other
resource a non-upgradeable contract spends once at deployment and carries for its entire lifetime. Packing the
struct is not bytecode-free: decoding seven bit-packed fields back out of one word costs more compiled code than a
flat word copy would, so the engine's runtime bytecode grew from 23,137 bytes to 23,584 bytes, 447 bytes larger.
Both figures stay comfortably inside the 24,576-byte contract-size ceiling - the baseline has 1,439 bytes of
headroom under that ceiling and the packed version still has 992. The trade this Insight proposes is therefore a
substantial, recurring, per-call gas reduction paid for with a small, one-time, headroom-safe increase in deployed
code size.

