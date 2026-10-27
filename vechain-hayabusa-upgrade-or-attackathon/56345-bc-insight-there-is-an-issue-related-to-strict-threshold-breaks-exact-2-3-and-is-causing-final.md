# 56345 bc insight there is an issue related to strict threshold breaks exact 2 3 and is causing finality freeze

Submitted on Oct 14th 2025 at 18:46:03 UTC by @XDZIBECX for [Attackathon | VeChain Hayabusa Upgrade](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* Report ID: #56345
* Report Type: Blockchain/DLT
* Report severity: Insight
* Target: <https://github.com/vechain/thor/compare/master...release/hayabusa>
* Impacts:
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief / Intro

There is an issue in Hayabusa’s BFT finality logic in the `Summarize()` function: it checks whether enough validators participated to “finalize” an epoch. It should accept at least two-thirds participation, but the code mistakenly requires strictly more than two-thirds.

Concretely:

* In PoA (count-based), the code uses `len(votes) > thresholdVotes`.
* In DPoS (weight-based), the code uses `justifiedWeight > thresholdWeight`.

Both checks use `>` where the protocol requires `>=`. When participation is exactly two-thirds (66.67%), due to integer division producing exact two-thirds thresholds frequently, the epoch is treated as not justified/committed. As a result the `Quality` metric does not increase, the finalized checkpoint isn’t updated, and finality stalls. This can cause the chain to stop confirming new transactions whenever participation is exactly 2/3.

## Vulnerability Details

The bug occurs in `Summarize()` where both PoA (vote-count) and DPoS (weight-based) branches use `>`:

{% code title="justifier.go (relevant excerpt)" %}

```
```

{% endcode %}

```go
func (js *justifier) Summarize() *bftState {
    var justified, committed bool

    // Pre-HAYABUSA (PoA mode)
    if js.thresholdWeight == 0 {
        justified = uint64(len(js.votes)) > js.thresholdVotes      // should be >=
        committed = js.comVotes > js.thresholdVotes                 // should be >=
    } else {
        // Post-HAYABUSA (DPoS mode)
        justified = js.justifiedWeight > js.thresholdWeight         // should be >=
        committed = js.comWeight > js.thresholdWeight               // should be >=
    }

    var quality uint32
    if justified {
        quality = js.parentQuality + 1
    } else {
        quality = js.parentQuality
    }

    return &bftState{
        Quality:   quality,
        Justified: justified,
        Committed: committed,
    }
}
```

This violates the VIP specification which states finality requires at least two-thirds of the total weighted staked VET (see VIP-253): "At least" means using `>=`. The code instead requires strictly more than two-thirds (`>`), so exact equality fails.

The threshold is calculated using integer arithmetic as exactly two-thirds:

{% code title="justifier.go (threshold calculation excerpt)" %}

```
```

{% endcode %}

```go
// PoS active path

if posActive {
    totalWeight, err := engine.getTotalWeight(sum) // Get total validator weight
    if err != nil {
        return nil, err
    }
    thresholdWeight := totalWeight * 2 / 3 // Calculate 2/3 threshold 
    return newJustifier(parentQuality, checkpoint, 0, thresholdWeight), nil
} else {
    mbp, err := engine.getMaxBlockProposers(sum)
    if err != nil {
        return nil, err
    }
    thresholdVotes := mbp * 2 / 3
    return newJustifier(parentQuality, checkpoint, thresholdVotes, 0), nil
}
```

When participation equals the computed threshold (exactly 2/3), `Summarize()` sets `Justified=false` and `Committed=false`, so `Quality` does not increase. Downstream, `CommitBlock()` only writes a new finalized checkpoint when `state.Committed == true` and `state.Quality > 1`. Therefore, no checkpoint is finalized and finality can stall. An attacker can withhold \~34% by stake to keep participation pinned at 2/3, indefinitely stalling finality until participation exceeds 2/3 by any epsilon.

## Impact Details

Because `Summarize()` uses `>` instead of `>=` for the 2/3 supermajority:

* Epochs with exactly two-thirds participation never reach `Committed=true`.
* `CommitBlock()` never writes a new finalized checkpoint.
* An attacker (or natural outages) withholding \~34% of stake can keep participation at exactly 2/3, stalling finality indefinitely.
* During such a stall, new transactions cannot be finalized; exchanges/wallets treat transfers as unsafe; bridges and downstream protocols likely pause; and fork-choice progress based on `Quality` halts, causing a network-wide liveness failure.

## Proof of Concept

Single minimal test proving the bug in both modes (PoA count and DPoS weight):

{% code title="finality\_bug\_test.go" %}

```
```

{% endcode %}

```go
package bft

import (
    "testing"

    "github.com/vechain/thor/v2/test/datagen"
)

// Single minimal test proving the bug in both modes (PoA count and DPoS weight).
func TestFinalityBug_MinimalProof(t *testing.T) {
    // PoA mode (count-based): exactly 2/3 should pass per spec, but fails in code
    t.Run("PoA_exactly_two_thirds_fails", func(t *testing.T) {
        mbp := uint64(9)
        threshold := mbp * 2 / 3 // 6
        js := newJustifier(0, 0, threshold, 0)
        for i := uint64(0); i < threshold; i++ {
            js.AddBlock(datagen.RandAddress(), true, 0)
        }
        st := js.Summarize()
        if st.Justified || st.Committed {
            t.Fatalf("expected FAIL at exactly 2/3; got Justified=%v Committed=%v", st.Justified, st.Committed)
        }
    })

    // PoA mode: 2/3 + 1 passes (proves strict > instead of >=)
    t.Run("PoA_two_thirds_plus_one_passes", func(t *testing.T) {
        mbp := uint64(9)
        threshold := mbp * 2 / 3 // 6
        js := newJustifier(0, 0, threshold, 0)
        for i := uint64(0); i < threshold+1; i++ {
            js.AddBlock(datagen.RandAddress(), true, 0)
        }
        st := js.Summarize()
        if !st.Justified || !st.Committed {
            t.Fatalf("expected PASS at 2/3 + 1; got Justified=%v Committed=%v", st.Justified, st.Committed)
        }
    })

    // DPoS mode (weight-based): pick totals divisible by 3 to hit exact 2/3
    t.Run("DPoS_exactly_two_thirds_weight_fails", func(t *testing.T) {
        total := uint64(900_000_000)           // total network weight
        threshold := total * 2 / 3             // 600,000,000 (exact 2/3)
        js := newJustifier(0, 0, 0, threshold) // weight mode
        js.AddBlock(datagen.RandAddress(), true, threshold)
        st := js.Summarize()
        if st.Justified || st.Committed {
            t.Fatalf("expected FAIL at exactly 2/3 weight; got Justified=%v Committed=%v", st.Justified, st.Committed)
        }
    })

    // DPoS mode: threshold + 1 passes (proves strict > instead of >=)
    t.Run("DPoS_two_thirds_plus_one_weight_passes", func(t *testing.T) {
        total := uint64(900_000_000)
        threshold := total * 2 / 3 // 600,000,000
        js := newJustifier(0, 0, 0, threshold)
        js.AddBlock(datagen.RandAddress(), true, threshold+1)
        st := js.Summarize()
        if !st.Justified || !st.Committed {
            t.Fatalf("expected PASS at threshold+1; got Justified=%v Committed=%v", st.Justified, st.Committed)
        }
    })
}
```

Logs of the test run:

{% code title="test\_run.log" %}

```
```

{% endcode %}

\`\`\`text Running tool: C:\Program Files\Go\bin\go.exe test -timeout 30s -run ^TestFinalityBug\_MinimalProof$ github.com/vechain/thor/v2/bft

\=== RUN TestFinalityBug\_MinimalProof === RUN TestFinalityBug\_MinimalProof/PoA\_exactly\_two\_thirds\_fails --- PASS: TestFinalityBug\_MinimalProof/PoA\_exactly\_two\_thirds\_fails (0.00s) === RUN TestFinalityBug\_MinimalProof/PoA\_two\_thirds\_plus\_one\_passes --- PASS: TestFinalityBug\_MinimalProof/PoA\_two\_thirds\_plus\_one\_passes (0.00s) === RUN TestFinalityBug\_MinimalProof/DPoS\_exactly\_two\_thirds\_weight\_fails --- PASS: TestFinalityBug\_MinimalProof/DPoS\_exactly\_two\_thirds\_weight\_fails (0.00s) === RUN TestFinalityBug\_MinimalProof/DPoS\_two\_thirds\_plus\_one\_weight\_passes --- PASS: TestFinalityBug\_MinimalProof/DPoS\_two\_thirds\_plus\_one\_weight\_passes (0.00s) --- PASS: TestFinalityBug\_MinimalProof (0.00s) PASS ok github.com/vechain/thor/v2/bft 0.238s

```

## References

- VIP: https://github.com/vechain/VIPs/blob/master/vips/VIP-253.md#finality
- Affected repository comparison: https://github.com/vechain/thor/compare/master...release/hayabusa
- Code references in the analysis above are from:
  - https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/justifier.go#L84C2-L98C3
  - https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/bft/justifier.go#L130C2-L137C3

(References used in the vulnerability details are included above.)

-- End of report.
```
