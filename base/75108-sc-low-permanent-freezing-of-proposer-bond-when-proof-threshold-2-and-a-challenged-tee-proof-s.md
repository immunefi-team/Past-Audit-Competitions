# 75108 sc low permanent freezing of proposer bond when proof threshold 2 and a challenged tee proof s zk challenge gets nullified in aggregateverifier

Submitted on Apr 27th 2026 at 09:25:35 UTC by [@aboudjem](https://immunefi.com/audit-competition/audit-comp-base-azul) for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75108
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

## Brief/Intro

On `PermissionedZKDisputeGame` deployments configured with `PROOF_THRESHOLD = 2` (the hardened production setting), a TEE-proven proposal that gets challenged with a ZK proof and then has that ZK proof nullified via `AggregateVerifier.nullify()` enters a state where `resolve()` reverts forever and `claimCredit()` reverts forever. The proposer's bond, locked in `DELAYED_WETH` at game creation, is irrecoverable for the lifetime of the contract. The NatSpec at `AggregateVerifier.sol:526-528` documents the opposite intent ("we allow the remaining TEE proof to resolve"), so the implementation contradicts the protocol's own documentation. In production this freezes operator and challenger bonds with no proposer-controllable recovery path.

## Vulnerability Details

### Root cause

`AggregateVerifier.resolve()` requires `proofCount >= PROOF_THRESHOLD` to take the resolution branch. With `PROOF_THRESHOLD = 2`, the following sequence locks the bond permanently:

1. Init: gameCreator submits a TEE proof. `proofCount = 1`, `expectedResolution = type(uint64).max`. Bond goes into `DELAYED_WETH`. (`AggregateVerifier.sol:383, 386-412`)
2. Challenge: a challenger files a ZK proof at intermediate root index `i`. `proofCount = 2`, `counteredByIntermediateRootIndexPlusOne = i + 1`, `expectedResolution = block.timestamp + SLOW_FINALIZATION_DELAY` (7 days). (`AggregateVerifier.sol:480-541`)
3. Nullify: a contradicting ZK proof at the same intermediate root gets submitted via `nullify(...)`. The codepath calls `_proofRefutedUpdate(ZK)`, which deletes `proofTypeToProver[ZK]` and decrements `proofCount` to 1, then calls `IVerifier(ZK_VERIFIER).nullify()`. (`AggregateVerifier.sol:548-602`)
4. After `SLOW_FINALIZATION_DELAY`, `gameOver()` returns true.

At this point:

* `resolve()` reverts `NotEnoughProofs()` because `proofCount(1) < PROOF_THRESHOLD(2)`. (`AggregateVerifier.sol:458`)
* No new ZK proof can rescue the count. `ZK_VERIFIER` is globally nullified, so every subsequent `verify()` reverts via the `notNullified` modifier. (`Verifier.sol:27-30`)
* `claimCredit()` reverts `GameNotResolved()` at `AggregateVerifier.sol:614`. `expectedResolution` is a concrete timestamp (not `type(uint64).max`), so the L613 branch is taken and `resolvedAt == 0`.

### Code citations (verified against base-contracts v8.1.0)

```
AggregateVerifier.sol:458   if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
AggregateVerifier.sol:526-528  // If the ZK is nullified, we allow the remaining TEE proof to resolve.
                               // The expected resolution time can no longer be increased as both proof types have been submitted.
                               // The exception is if the ZK proof is nullified, in which case the expected resolution will be
                               // increased by SLOW_FINALIZATION_DELAY from the time of nullification.
AggregateVerifier.sol:589   _proofRefutedUpdate(proofType);  // proofTypeToProver[ZK]=0, proofCount-=1
AggregateVerifier.sol:594-598 if (proofType == ProofType.ZK) {
                                delete counteredByIntermediateRootIndexPlusOne;
                                IVerifier(ZK_VERIFIER).nullify();
                              }
AggregateVerifier.sol:613-617 if (expectedResolution.raw() != type(uint64).max) {
                                if (resolvedAt.raw() == 0) revert GameNotResolved();
                              } else {
                                if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
                              }
Verifier.sol:14, 27-30      bool public nullified;
                            modifier notNullified() { if (nullified) revert Nullified(); _; }
Verifier.sol:39-47          function nullify() external override { ... } nullified = true; ...
```

The NatSpec at L526-528 says nullifying the ZK proof should leave the remaining TEE proof free to resolve the game. The implementation does the opposite: with `PROOF_THRESHOLD = 2`, `proofCount = 1` after nullification cannot satisfy `resolve`'s minimum, and the global `ZK_VERIFIER` nullification prevents restoring it.

### Why governance rescue does not apply

`AggregateVerifier.resolve()` has a side branch at L453-454 that resolves to `CHALLENGER_WINS` when the parent game's status is `CHALLENGER_WINS`, which would unlock the bond to a challenger. Reaching that branch requires invalidating the parent game. The stuck-game proposer cannot trigger that themselves. From their perspective the freeze is permanent.

## Impact Details

* **Direct loss**: per-game proposer bond, locked permanently in `DELAYED_WETH` (`AggregateVerifier.sol:411-412`). Base operator bond size for `PermissionedZKDisputeGame` sits in the 0.08-1 ETH range across networks (\~$256-$3,200 at $3.2k ETH).
* **Repeatable**: every game finalised under PROOF\_THRESHOLD=2 is exposed. An attacker who controls a contradicting ZK proof can lock every challenged proposal indefinitely.
* **Aggregate**: at one proposal per 30 minutes on a busy L2, a 1-week attack window strands ~~336 bonds (~~$100k-$1M depending on bond size).
* **Maps to Immunefi v2.2 Smart Contracts**: "Permanent freezing of funds (>24h)", High tier.
* **Program impact match**: the Base Azul program impact list lists Permanent Freezing as in-scope (item 6).

## References

* `src/multiproof/AggregateVerifier.sol` lines 383, 458, 480-541, 548-602, 604-634, 763-823 (Base Azul scope, base-contracts v8.1.0)
* `src/multiproof/Verifier.sol` lines 14, 27-30, 39-47
* Immunefi v2.2 severity classification: knowledge/immunefi-severity-classification-v2-2-FULL.md

## Link to Proof of Concept

<https://gist.github.com/Aboudjem/3a1b20583f22593bf0e1b1cfc76babc4>

## Proof of Concept

Foundry test in `poc/H-A2-01/H-A2-01.t.sol` plus README at `poc/H-A2-01/README.md`.

{% stepper %}
{% step %}

## Deploy

Deploy `AggregateVerifier` with `PROOF_THRESHOLD = 2` and mock TEE/ZK verifiers extending the abstract `Verifier`.
{% endstep %}

{% step %}

## Initialise

Initialise a clone with `gameCreator = proposer` and a TEE proof. `proofCount = 1`.
{% endstep %}

{% step %}

## Challenge

Challenger calls `challenge(zkProof, 0, intermediateRoot)`. `proofCount = 2`, `expectedResolution = now + 7 days`.
{% endstep %}

{% step %}

## Nullify

Attacker calls `nullify(contradictingZkProof, 0, differentRoot)`. `_proofRefutedUpdate` decrements `proofCount` to 1; `IVerifier(ZK_VERIFIER).nullify()` flips the global flag.
{% endstep %}

{% step %}

## Advance time

`vm.warp(now + 7 days + 1)`. `gameOver() == true`.
{% endstep %}

{% step %}

## Resolve

`resolve()` reverts `NotEnoughProofs`.
{% endstep %}

{% step %}

## Verify again

Any further `verifyProposalProof(zk)` reverts `Nullified()` (`Verifier.sol:27-30`).
{% endstep %}

{% step %}

## Claim credit

`claimCredit()` reverts `GameNotResolved` at `AggregateVerifier.sol:614`.
{% endstep %}
{% endstepper %}

Expected outcome (per NatSpec L526-528): bond claimable after the resolution delay.

Actual outcome: bond permanently frozen in `DELAYED_WETH`. Proposer has no on-chain recovery path.

Code references walked by the PoC:

* `AggregateVerifier.sol:383`, expectedResolution sentinel set at init
* `AggregateVerifier.sol:458`, NotEnoughProofs revert
* `AggregateVerifier.sol:526-528`, NatSpec contradicting the implementation
* `AggregateVerifier.sol:594-598`, global ZK\_VERIFIER.nullify() invocation
* `AggregateVerifier.sol:613-617`, claimCredit branching that reverts GameNotResolved forever
* `Verifier.sol:27-30, 39-47`, global nullified flag
