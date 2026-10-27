# 56256 bc insight redundant sload for global endorsement parameter

**Submitted on:** Oct 13th 2025 at 16:07:30 UTC by @rionnaldi\
**Competition:** [Attackathon | VeChain Hayabusa Upgrade](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)\
**Report ID:** #56256\
**Report Type:** Blockchain/DLT\
**Severity:** Insight\
**Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>

***

## Description

### Brief / Intro

The `native_isEndorsed` function is called to verify if a validator meets the endorsement requirements. Every time it is called, it reads the global `ProposerEndorsement` value from the `Params` contract storage.

### Vulnerability Details

* `ProposerEndorsement` is a global parameter shared by all validators and does not change within a single transaction.
* Reading this value uses an SLOAD operation; a cold SLOAD costs \~2100 gas.
* If `isEndorsed` is invoked multiple times within the same transaction (for example, by a contract iterating through validators), the SLOAD is repeated. Subsequent reads are warm and cost \~100 gas each, but the initial cold read cost (\~2100 gas) can still be significant if repeated across calls.

### Impact

* Gas savings of \~2100 gas per call to `isEndorsed` (for each avoided cold SLOAD).

***

## Recommendation

{% hint style="info" %}
Read the global `ProposerEndorsement` parameter once per block/transaction at a higher layer and pass it into `isEndorsed` to avoid repeated SLOADs. For system-level processes that check endorsements for multiple validators (e.g., consensus engine), prefer reading the parameter once and reusing it.
{% endhint %}

* If multiple on-chain calls are expected in a single transaction, consider adding a batch API such as `areEndorsed(address[] calldata nodeMasters)` which reads the parameter once and checks a batch of validators in a loop.

***

## Benefit

* Gas savings: avoids at least one SLOAD (\~2100 gas) for every additional call to `isEndorsed` within the same transaction.
* Efficiency: reduces redundant storage reads and makes on-chain validation checks more efficient.

***

## Measurable Impact

Caching the global parameter or batching endorsement checks will reduce gas consumption and lower blockchain bloat when multiple endorsement checks are performed in a single transaction.

***

## References

* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/authority\\_native.go>

***

## Proof of Concept

<details>

<summary>Relevant code snippet (builtin/authority_native.go)</summary>

{% code title="builtin/authority\_native.go" %}

```go
{"native_isEndorsed", func(env *xenv.Environment) []any {
// ...
    env.UseGas(thor.SloadGas) // SLOAD for endorsement param
    endorsement, err := Params.Native(env.State()).Get(thor.KeyProposerEndorsement)
    }
}
// ...
```

{% endcode %}

</details>
