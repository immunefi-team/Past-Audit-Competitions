# 56403 bc insight there is a problem in the dpos threshold switch undercounts votes at hayabusa activation&#x20;

**Submitted on Oct 15th 2025 at 16:13:52 UTC by @XDZIBECX for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain/hayabusa-upgrade-attackathon)

* **Report ID:** #56403
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:** A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief / Intro

When Hayabusa switches on PoS, the round immediately begins using a stake-weighted 2/3 rule to decide finality. In that same round, COM votes from blocks at or below the activation height are recorded with weight = 0. These early votes are never upgraded later in the round because the justifier keeps only the first-seen vote per signer and never replaces it. As a result, some real stake is not counted toward the weighted threshold. An epoch that truly has ≥ 2/3 by stake can still show not justified/committed until additional post-boundary (properly weighted) votes arrive. This causes a temporary finality delay right at the switch. Blocks continue producing and contracts remain callable; there is no chain split. See PoC and details below.

## Vulnerability Details

At the start of a round, the engine constructs a `justifier` and, if PoS is active, sets a weighted threshold. This happens here:

* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/justifier.go#L83C1-L91C10>

Excerpt:

```go
posActive, err := staker.IsPoSActive()
...
if posActive {
    totalWeight, err := engine.getTotalWeight(sum)
    thresholdWeight := totalWeight * 2 / 3
    return newJustifier(parentQuality, checkpoint, 0, thresholdWeight), nil
}
```

In the same round, the engine already uses a stake-weighted 2/3 threshold, so any votes cast at or before the activation height are recorded with weight = 0:

* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/engine.go#L323C3-L348C1>

Excerpt:

```go
 if h.Number() > engine.forkConfig.HAYABUSA+thor.HayabusaTP() {
    parentBlockSummary, err = engine.repo.GetBlockSummary(h.ParentID())
    if err != nil {
        return nil, err
    }
    state := engine.stater.NewState(parentBlockSummary.Root())
    staker := builtin.Staker.Native(state)

    var weight uint64
    if posActive, _ := staker.IsPoSActive(); posActive {
        // PoS is active, get validator weight
        validator, err := staker.GetValidation(signer)
        if err != nil {
            return nil, err
        }
        if validator == nil {
            return nil, errors.New("validator not found")
        }
        weight = validator.Weight
    }
    // If PoS is not active or error occurred, weight remains nil
    js.AddBlock(signer, h.COM(), weight)
 } else {
    js.AddBlock(signer, h.COM(), 0)  << here the zero-weighted vote at boundary 
 }
```

The justifier keeps only the first vote per signer and doesn’t upgrade a signer’s weight later in the round. Thus if the first-seen vote is zero-weight (because it was at/below the activation height), it remains zero-weight for the rest of the round. When the protocol uses the stake-weighted rule (≥ 2/3 of total stake), these zero-weight early votes undercount the real stake and can prevent justification/commit even when the actual stake participation is ≥ 2/3.

## Impact Details

At the Hayabusa PoS activation boundary, the protocol switches to a stake-weighted 2/3 threshold but still counts some early votes in the same round as zero weight. That undercounts real stake, so an epoch that actually has ≥ two-thirds weighted participation can fail to justify/commit until more counted votes arrive. The chain continues producing blocks and contracts are still callable, but finality is temporarily delayed. This can delay unlocks/settlements, extend safety windows, and lag oracle/bridge updates that depend on timely finalization. No direct funds are at risk.

## References

* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/justifier.go#L83C1-L91C10>
* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/engine.go#L323C3-L348C1>

## Proof of Concept

A unit test demonstrating the issue (the test shows counted weight under the threshold while actual stake equals the threshold, then later justification occurs only after extra counted weight arrives):

```go
package bft

import (
	"testing"

	"github.com/vechain/thor/v2/test/datagen"
)

func TestFinalityBoundary_UndeniableProof(t *testing.T) {
	total := uint64(900_000_000)
	threshold := total * 2 / 3 // 600,000,000

	// Weighted mode justifier (DPoS)
	js := newJustifier(0, 0, 0, threshold)

	var actualWeight uint64  // what SHOULD count per DPoS rule (>= 2/3 total weighted stake)
	var countedWeight uint64 // what DOES count in code at the boundary (some early votes zero-weighted)

	// EARLY COM votes at/below boundary: these would have non-zero stake, but are zero-counted here
	for i := 0; i < 3; i++ {
		actualWeight += 100_000_000
		js.AddBlock(datagen.RandAddress(), true, 0) // zero weight counted
	}

	// LATE COM votes above boundary: counted with weight
	js.AddBlock(datagen.RandAddress(), true, 300_000_000)
	actualWeight += 300_000_000
	countedWeight += 300_000_000

	// At this point:
	// - actualWeight == threshold (600,000,000)
	// - countedWeight == 300,000,000 (UNDER threshold due to early zero-weight votes)
	st := js.Summarize()
	t.Logf("[Boundary BEFORE] total=%d threshold=%d actual=%d counted=%d => Justified=%v Committed=%v",
		total, threshold, actualWeight, countedWeight, st.Justified, st.Committed)
	if st.Justified || st.Committed {
		t.Fatalf("expected NOT justified/committed when actual==threshold but counted<threshold")
	}

	// Add minimal extra counted weight so counted > threshold
	more := threshold - countedWeight + 1 // => 300,000,001
	js.AddBlock(datagen.RandAddress(), true, more)
	countedWeight += more
	actualWeight += more

	st = js.Summarize()
	t.Logf("[Boundary AFTER ] total=%d threshold=%d actual=%d counted=%d => Justified=%v Committed=%v",
		total, threshold, actualWeight, countedWeight, st.Justified, st.Committed)
	if !st.Justified || !st.Committed {
		t.Fatalf("expected justified/committed once counted > threshold")
	}
}
```

<details>

<summary>PoC test run logs</summary>

```go
Running tool: C:\Program Files\Go\bin\go.exe test -timeout 30s -run ^TestFinalityBoundary_UndeniableProof$ github.com/vechain/thor/v2/bft

=== RUN   TestFinalityBoundary_UndeniableProof
    c:\Users\baouc\thor\hayabusa\bft\finality_boundary_bug_test.go:78: [Boundary BEFORE] total=900000000 threshold=600000000 actual=600000000 counted=300000000 => Justified=false Committed=false
    c:\Users\baouc\thor\hayabusa\bft\finality_boundary_bug_test.go:91: [Boundary AFTER ] total=900000000 threshold=600000000 actual=900000001 counted=600000001 => Justified=true Committed=true
--- PASS: TestFinalityBoundary_UndeniableProof (0.00s)
PASS
ok      github.com/vechain/thor/v2/bft  0.335s
```

</details>

***

If you want, I can suggest concrete code fixes (e.g., upgrade existing zero-weight first-seen votes when PoS becomes active, or delay using stake-weighted threshold until all signer weights for the round are known) and produce a small patch sketch.
