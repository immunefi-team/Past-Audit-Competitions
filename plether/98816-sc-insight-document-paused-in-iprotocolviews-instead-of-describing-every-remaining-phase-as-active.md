# \#98816 [SC-Insight] Document Paused in IProtocolViews instead of describing every remaining phase as Active

**Submitted on Nov 10th 2026 at 11:47:20 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98816
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Summary

`IProtocolViews.getProtocolStatus()` documents `Configuring`, `Degraded`, and
then `Active` otherwise. The implementation has a fourth valid outcome:
`Paused`. For a fully wired, non-degraded protocol whose Router is paused, the
view returns numeric phase `3` (`ProtocolPhase.Paused`), while the public
interface comment places that state in its `Active otherwise` remainder.

This is a Documentation Improvements Insight. The runtime phase derivation is
correct; the interface's semantic contract is incomplete.

### Affected asset, category, and location

- **Selected asset:** the in-scope
  [`packages/perps/src` catch-all](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src)
- **Category:** Documentation Improvements
- **Bug location:**
  [`packages/perps/src/interfaces/IProtocolViews.sol:9-15`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IProtocolViews.sol#L9-L15)

`PerpsPublicLens.sol` is excluded from the selected asset and is cited only as
corroborating implementation evidence.

### Exact discrepancy

The interface says the phase is `Configuring` until the system is ready,
`Degraded` when insolvency is latched, and `Active` otherwise. It never names
`Paused`.

The canonical
[`ProtocolPhase` enum](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngine.sol#L251-L263)
contains four values and expressly places `Paused` after the degraded check.
The returned
[`ProtocolStatusView`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/PerpsViewTypes.sol#L228-L247)
exposes that phase as a numeric `uint8`. The
[`PerpsPublicLens` derivation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PerpsPublicLens.sol#L496-L524)
returns `Paused` when Router governance is paused; only the final ready case
returns `Active`.

### Concrete value of the improvement

A product consumer that derives an exhaustive phase mapping, generated API
documentation, or an operational label from the public interface NatSpec can
treat numeric phase `3` as unknown or apply the documented `Active otherwise`
fallback. Either result can misrepresent a governance-paused protocol in a
status page, alert, or integration.

The audience is repository-defined: the canonical entrypoint guide
[`directs protocol/status consumers to PerpsPublicLens and IProtocolViews`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L295-L340).
The correction changes one interface comment and requires no ABI, runtime,
storage, caller, or migration change.

### Relationship to Cecuro LOW-15

Cecuro
[`LOW-15`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
reported the earlier runtime problem: protocol status could return `Active`
while Router risk-off blocked new opens. Its remediation added the
`ProtocolPhase.Paused` enum value and the runtime branch that returns it.

The current runtime now reports `Paused` correctly. The remaining issue is the
different documentation defect left after that remediation: the public
`IProtocolViews` comment still describes only three outcomes and labels the
remainder `Active`. This report requests only completion of that interface
contract; it does not re-report LOW-15's fixed runtime behavior.

### Recommended correction

Update `IProtocolViews.getProtocolStatus()` to document the exact implemented
order:

1. missing mandatory dependency code, or a Router admin with no code, is
   `Configuring`;
2. otherwise degraded mode is `Degraded`;
3. otherwise Router pause is `Paused`;
4. otherwise a pool that cannot increase risk is `Configuring`; and
5. only the remaining fully ready state is `Active`.

### Limitations and non-claims

- No pause bypass, incorrect runtime phase, transaction failure, fund impact,
  or vulnerability severity is claimed.
- No external consumer failure or production occurrence was observed.
- The excluded `PerpsPublicLens.sol` is evidence, not the selected bug location.
- The requested change is limited to this one interface NatSpec.


## Proof of Concept

## Proof of Concept

Runnable code is not required for this Documentation Improvements Insight. The
deterministic source comparison is:

1. Read
   [`IProtocolViews.getProtocolStatus()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IProtocolViews.sol#L9-L15).
   Its NatSpec names `Configuring` and `Degraded`, then says `Active otherwise`;
   it does not name `Paused`.
2. Read the
   [`ProtocolPhase` enum](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngine.sol#L251-L263).
   Solidity's zero-based enum encoding makes `Configuring = 0`, `Active = 1`,
   `Degraded = 2`, and `Paused = 3`.
3. Read the
   [`ProtocolStatusView` return type](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/PerpsViewTypes.sol#L228-L247).
   It exposes the phase as `uint8`, not as a self-describing string.
4. Read the
   [`_getProtocolPhase()` implementation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PerpsPublicLens.sol#L496-L524).
   After the wiring and degraded checks, a paused Router admin returns
   `ProtocolPhase.Paused`; only the remaining ready case returns `Active`.

The decisive state comparison is:

| Condition | Interface NatSpec | Runtime result |
|---|---|---|
| Fully wired; non-degraded; Router paused; pool otherwise ready | `Active otherwise` | `uint8(ProtocolPhase.Paused)`, value `3` |

### Conditions under which the Insight is valuable

The correction is valuable whenever a maintainer, product consumer, monitoring
integration, generated-documentation consumer, test author, or incident
responder uses the public interface NatSpec to understand, implement, test, or
expose the protocol's complete phase model. The protocol need not currently be
paused and no consumer failure must already have occurred: the source-defined
paused branch proves that `Paused` is a valid public result which the present
interface contract omits.

In the concrete paused state shown above, a consumer that maps the numeric
`uint8` result can treat value `3` as unknown or apply the documented `Active
otherwise` remainder. The proposed comment-only correction prevents that
misunderstanding by naming all four valid values and their precedence without
changing the ABI or runtime behavior.

No Gist, attachment, transaction, RPC call, or runtime exploit is needed.

