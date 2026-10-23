# 74371 sc insight zkverifier sol declares sp1verificationfailed error that is never raised leaving the sp1 trust boundary without a defensive wrapper

**Submitted on Apr 22nd 2026 at 04:14:57 UTC by @notok for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74371
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>

## Impacts

## Description

### Brief/Intro

ZkVerifier declares an `SP1VerificationFailed` error that is never raised anywhere in the contract or elsewhere in the codebase. The `verify()` function calls `SP1_VERIFIER.verifyProof()` directly and returns `true` unconditionally, with no try/catch wrapper to translate SP1 failures into the declared error. The dead declaration implies the presence of a defensive layer that does not exist, parallel to the unused `L2_CHAIN_ID` immutable variable flagged in audit finding 3.1.3 of the March 2026 Cantina multiproof audit.

Target: `src/multiproof/zk/ZKVerifier.sol:19` (commit `01dad23`, tag `v8.1.0`).

### Vulnerability Details

`src/multiproof/zk/ZKVerifier.sol`:

```solidity
contract ZkVerifier is Verifier {
    ISP1Verifier public immutable SP1_VERIFIER;

    /// @notice Thrown when SP1 proof verification reverts.
    error SP1VerificationFailed();                                   // declared, never raised

    function verify(bytes calldata proofBytes, bytes32 imageId, bytes32 journal)
        external view override notNullified returns (bool)
    {
        SP1_VERIFIER.verifyProof(imageId, abi.encodePacked(journal), proofBytes);
        return true;                                                  // no catch, no wrapper
    }
}
```

Repository-wide grep confirms `SP1VerificationFailed` is declared once and raised zero times:

```bash
$ grep -rn "SP1VerificationFailed" src/
src/multiproof/zk/ZKVerifier.sol:19:    error SP1VerificationFailed();
```

No `revert SP1VerificationFailed()` or `catch { revert SP1VerificationFailed(); }` exists in the repository.

The NatSpec above the declaration describes `SP1VerificationFailed` as "Thrown when SP1 proof verification reverts." The implementation does not honor this contract. The `verify()` function delegates to `SP1_VERIFIER.verifyProof()` and returns `true` unconditionally. Failures raised by the SP1 gateway propagate with whatever selector and signature the SP1 implementation raises, not `SP1VerificationFailed.selector`.

Two interpretations explain the divergence between declared error surface and runtime behavior:

1. An earlier revision of the contract wrapped the SP1 call in a try/catch block and used the error to normalize failure reasons across different SP1 versions. The wrapper was removed before the backport landed, but the error declaration remained.
2. The error was added proactively with intent to wrap, but the wrapper was never implemented.

In either case, the current state presents a contract to the reader that the implementation does not fulfill. Downstream consumers (off-chain indexers, error-selector-based monitoring, custom revert-reason decoders) looking for `SP1VerificationFailed.selector` will never match a revert from `ZkVerifier`.

The parallel to audit finding 3.1.3 (`L2_CHAIN_ID` immutable) is exact. The 3.1.3 description reads: "the presence of an unused immutable variable is misleading, as it suggests proofs are explicitly bound to a specific L2 chain within this contract when they are not." The same reasoning applies one layer up: the presence of an unused error declaration is misleading, because it suggests SP1 verification failures are handled as a distinct, named failure mode when they are not. Coinbase accepted 3.1.3 as Informational. The `SP1VerificationFailed` case was not reviewed because `ZKVerifier.sol` was added to the codebase via commit `01dad23` on 2026-04-21 16:40 UTC, 3 hours 20 minutes before contest start, and has no coverage in the four Cantina audits on record.

### Impact Details

Code-quality impact: moderate. Off-chain monitors built against the declared error surface receive no matches. For a contract that sits on the SP1 trust boundary and whose failure mode is the single most important event to alert on, an unreachable named error is a concrete observability gap.

Security-adjacent impact: low but non-trivial. Correct behavior of `verify()` depends entirely on SP1 reverting on failure. The deployed SP1 gateway is immutable, so an adversary cannot swap it. However, SP1 verifier gateways in production use selector-based routing (bytes4 prefix of the proof bytes selects the verifier implementation), and the route table is typically governed by the gateway operator. If the operator routes to an implementation that returns without reverting on an invalid proof (a misconfigured route, a future verifier with different semantics, an emergency admin action during incident response), `ZkVerifier.verify()` returns `true` for arbitrary inputs. A try/catch + `revert SP1VerificationFailed()` wrapper would have bounded this risk by enforcing revert semantics at the `ZKVerifier` layer regardless of SP1 gateway state. As implemented, no such bound exists.

Under current SP1 standard semantics, the silent-acceptance scenario is hypothetical. The gap is narrow but real: the trust boundary at `ZKVerifier` is thinner than the error surface implies.

## Recommendation

Resolve the divergence between declared error surface and runtime behavior. Either option closes the gap.

**Option A: Remove the dead declaration** (aligns declaration with actual behavior):

```solidity
// Delete lines 18-19:
// /// @notice Thrown when SP1 proof verification reverts.
// error SP1VerificationFailed();
```

**Option B: Restore the defensive wrapper** (aligns runtime with declared contract):

```solidity
function verify(bytes calldata proofBytes, bytes32 imageId, bytes32 journal)
    external view override notNullified returns (bool)
{
    try SP1_VERIFIER.verifyProof(imageId, abi.encodePacked(journal), proofBytes) {
        return true;
    } catch {
        revert SP1VerificationFailed();
    }
}
```

Option B adds defense in depth: the revert reason seen by `AggregateVerifier._verifyZkProof` and any off-chain observer becomes stable regardless of which SP1 verifier implementation is active on the gateway. This matches the pattern used by the parent `Verifier` abstract where error surfaces are explicitly normalized (`NotProperGame`, `Nullified`).

Option A is the minimal fix if the declared error is confirmed legacy residue.

## References

* `src/multiproof/zk/ZKVerifier.sol` (51 LoC, commit `01dad23`, 2026-04-21 16:40 UTC)
* Cantina Coinbase Multiproof Audit (March 2026), finding 3.1.3 "Inaccurate NatSpec and Dead Code" (parallel reasoning)
* `ISP1Verifier` interface contract at `src/dispute/zk/ISP1Verifier.sol`
* Parent `Verifier` abstract at `src/multiproof/Verifier.sol` (normalized error surface reference)

## Link to Proof of Concept

<https://gist.github.com/xnotok-ops/98acbc2827185ec41b99a95e0bbe1b5b>

<details>

<summary>Proof of Concept</summary>

### Static evidence (primary, conclusive)

Repository-wide grep from the `base/contracts` v8.1.0 checkout:

```bash
$ cd repos/contracts
$ grep -rn "SP1VerificationFailed" src/
src/multiproof/zk/ZKVerifier.sol:19:    error SP1VerificationFailed();
```

Single match on the declaration line. Zero usages of `revert SP1VerificationFailed()` or `catch { revert SP1VerificationFailed(); }` anywhere in the repository. The declared error has no reachable code path. This grep is conclusive on its own.

### Dynamic evidence (optional, source attached)

The file `PoC_ZKVerifier_DeadError.t.sol` (attached) contains Foundry tests that demonstrate two corollaries of the static proof:

1. `test_SP1VerificationFailed_isUnreachable`: when the SP1 gateway reverts, the revert propagates with the gateway's own error signature, not `SP1VerificationFailed.selector`.
2. `test_silentAcceptance_whenSP1DoesNotRevert`: when the SP1 gateway does not revert (hypothetical misrouted verifier), `verify()` returns `true` for a zero-byte proof with zero imageId and zero journal, confirming the absence of a defensive layer.
3. `test_staticGrep_noUsageOfDeclaredError`: documentation marker for the static grep step.

### Setup and run

Requires `base/contracts` Foundry dependencies to be installed:

```bash
cd repos/contracts
make deps
mkdir -p test/poc
cp PoC_ZKVerifier_DeadError.t.sol test/poc/
forge test --match-contract ZKVerifierDeadErrorTest -vvvv
```

Reproduction is optional. The static grep above is complete proof of the finding.

</details>
