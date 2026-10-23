# 56657 bc insight inactive validator scheduling bypass in vechain thor pos consensus mechanism

**Submitted on Oct 19th 2025 at 01:53:25 UTC by @Angry\_Mustache\_Man for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)

* **Report ID:** #56657
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

A vulnerability exists in the VeChain Thor Proof-of-Stake (PoS) consensus mechanism that allows `inactive/offline` validators to be included in the block production scheduling sequence, bypassing the staking system's online participation requirements. This vulnerability affects the `pos.NewScheduler()` function in `pos/sched.go` and can lead to consensus calculation errors and scheduling inconsistencies.

### Vulnerability Details

The condition `p.Active || p.Address == addr` creates a privilege escalation where:

<https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/pos/sched.go#L75-L82>

```go
// but only active/online validators will be picked for block production
if p.Active || p.Address == addr {
    shuffled = append(shuffled, entry{
        address: p.Address,
        weight:  p.Weight,
        active:  p.Active,
        score:   -math.Log(random) / float64(p.Weight),
    })
```

* Normal validators: Must be Active = true (online) to be included in scheduling.
* Node Master: Gets included regardless of their online status.
* There are no checks done before this function too.

Even the codebase comments indicate an error should be returned if the node master was not Active, but that check is not implemented:

<https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/pos/sched.go#L40-L47>

```go
// If `addr` is not listed in `proposers` or not active, an error returned.
func NewScheduler(
    addr thor.Address,
    proposers []Proposer,
    parentBlockNumber uint32,
    parentBlockTime uint64,
    seed []byte,
) (*Scheduler, error) {
```

## Bug Flow

{% stepper %}
{% step %}

#### Step: Node master goes offline

Node master goes offline but remains registered in staking system.
{% endstep %}

{% step %}

#### Step: Scheduler includes offline node master

Scheduler includes offline node master in scheduling sequence due to `p.Address == addr` bypass.
{% endstep %}

{% step %}

#### Step: Consensus calculations include offline validator's weight

Consensus calculations include offline validator's weight, affecting totals.
{% endstep %}

{% step %}

#### Step: Scheduling becomes inconsistent

Scheduling becomes inconsistent with actual network state and violates online participation requirements.
{% endstep %}
{% endstepper %}

### Impact Details

Including an offline Node creates delays and inconsistencies. It violates the "online participation" consensus requirement for block production and creates consensus calculation errors by including offline validator's weight in total weight calculations.

## Proof of Concept

Include this test in the file `pos/sched_test.go`:

{% code title="pos/sched\_test.go" %}

```go
func TestScheduler_InactiveProposerVulnerability_POC(t *testing.T) {
    t.Log("=== POC: Inactive Proposer Vulnerability Test ===")
    t.Log("This test demonstrates that an inactive proposer can still be scheduled")
    t.Log("for block production, while other inactive proposers are excluded.")

    // Create test proposers with one inactive target proposer
    proposers := []Proposer{
        {Address: thor.BytesToAddress([]byte("active1")), Weight: 100_000_000, Active: true},
        {Address: thor.BytesToAddress([]byte("active2")), Weight: 200_000_000, Active: true},
        {Address: thor.BytesToAddress([]byte("inactive_target")), Weight: 300_000_000, Active: false}, // Target: inactive but should be scheduled
        {Address: thor.BytesToAddress([]byte("inactive_other")), Weight: 400_000_000, Active: false},  // Other: inactive and should be excluded
        {Address: thor.BytesToAddress([]byte("active3")), Weight: 500_000_000, Active: true},
    }

    targetProposer := thor.BytesToAddress([]byte("inactive_target"))
    otherInactiveProposer := thor.BytesToAddress([]byte("inactive_other"))

    // Step 1: Create scheduler with inactive target proposer
    t.Log("\n Step 1: Creating scheduler with inactive target proposer...")
    sched, err := NewScheduler(targetProposer, proposers, 1, 10, []byte("vulnerability_test"))
    assert.NoError(t, err, "Scheduler should be created successfully even with inactive target")

    // Step 2: Verify the target proposer is included in the sequence despite being inactive
    t.Log("\n Step 2: Verifying target proposer inclusion in sequence...")
    targetInSequence := false
    otherInactiveInSequence := false
    activeCount := 0

    for _, entry := range sched.sequence {
        if entry.address == targetProposer {
            targetInSequence = true
            t.Logf(" VULNERABILITY CONFIRMED: Inactive target proposer %s is in sequence with weight %d",
                entry.address.String(), entry.weight)
        }
        if entry.address == otherInactiveProposer {
            otherInactiveInSequence = true
            t.Logf(" UNEXPECTED: Other inactive proposer %s is in sequence (should be excluded)",
                entry.address.String())
        }
        if entry.active {
            activeCount++
        }
    }

    // Assertions for the vulnerability
    assert.True(t, targetInSequence, "VULNERABILITY: Inactive target proposer should be in sequence")
    assert.False(t, otherInactiveInSequence, "Other inactive proposer should be excluded from sequence")

    t.Logf(" Sequence contains %d active proposers out of %d total entries", activeCount, len(sched.sequence))

    // Step 3: Test that the inactive proposer can be scheduled for block production
    t.Log("\n Step 3: Testing block production scheduling for inactive proposer...")
    parentTime := uint64(10)
    nowTime := uint64(20)

    // Schedule block production
    scheduledTime := sched.Schedule(nowTime)
    t.Logf(" Scheduled block time: %d", scheduledTime)

    // Verify the scheduled time is valid for the inactive proposer
    isValidTime := sched.IsTheTime(scheduledTime)
    assert.True(t, isValidTime, "VULNERABILITY: Inactive proposer should be able to produce blocks at scheduled time")

    // Step 4: Test that the inactive proposer appears in the scheduling sequence
    t.Log("\n Step 4: Verifying inactive proposer appears in scheduling sequence...")

    // Check multiple time slots to see if the inactive proposer gets scheduled
    foundInScheduling := false
    for i := uint64(0); i < 10; i++ {
        testTime := parentTime + thor.BlockInterval()*(i+1)
        if sched.IsScheduled(testTime, targetProposer) {
            foundInScheduling = true
            t.Logf(" VULNERABILITY CONFIRMED: Inactive proposer scheduled at time %d", testTime)
            break
        }
    }

    assert.True(t, foundInScheduling, "VULNERABILITY: Inactive proposer should appear in scheduling sequence")

    // Step 5: Test weight distribution impact
    t.Log("\n Step 5: Testing weight distribution impact...")

    // Calculate total weight in sequence (should include inactive target)
    totalWeightInSequence := uint64(0)
    for _, entry := range sched.sequence {
        totalWeightInSequence += entry.weight
    }

    // Calculate expected weight (all active + inactive target)
    expectedWeight := uint64(0)
    for _, p := range proposers {
        if p.Active || p.Address == targetProposer {
            expectedWeight += p.Weight
        }
    }

    t.Logf(" Total weight in sequence: %d", totalWeightInSequence)
    t.Logf(" Expected weight (active + inactive target): %d", expectedWeight)

    assert.Equal(t, expectedWeight, totalWeightInSequence,
        "Weight calculation should include inactive target proposer")

    // Step 6: Test Updates function behavior
    t.Log("\n Step 6: Testing Updates function with inactive proposer...")

    updates, score := sched.Updates(nowTime, totalWeightInSequence)
    t.Logf(" Updates count: %d, Score: %d", len(updates), score)

    // The inactive proposer should be marked as active in updates
    targetMarkedActive := false
    for _, update := range updates {
        if update.Address == targetProposer && update.Active {
            targetMarkedActive = true
            t.Logf(" VULNERABILITY: Inactive proposer marked as active in updates")
            break
        }
    }

    assert.True(t, targetMarkedActive, "VULNERABILITY: Inactive proposer should be marked as active in updates")

    // Final assertion to confirm the vulnerability
    assert.True(t, targetInSequence && foundInScheduling && targetMarkedActive,
        "VULNERABILITY CONFIRMED: Inactive proposer can participate in consensus")
}
```

{% endcode %}

Run the test:

go test -v -run TestScheduler\_InactiveProposerVulnerability\_POC ./pos -cover

<details>

<summary>Sample test output</summary>

```
=== RUN   TestScheduler_InactiveProposerVulnerability_POC
    sched_test.go:416: === POC: Inactive Proposer Vulnerability Test ===
    sched_test.go:417: This test demonstrates that an inactive proposer can still be scheduled
    sched_test.go:418: for block production, while other inactive proposers are excluded.
    sched_test.go:433:
         Step 1: Creating scheduler with inactive target proposer...
    sched_test.go:438:
         Step 2: Verifying target proposer inclusion in sequence...
    sched_test.go:446:  VULNERABILITY CONFIRMED: Inactive target proposer 0x0000000000696e6163746976655f746172676574 is in sequence with weight 300000000
    sched_test.go:463:  Sequence contains 3 active proposers out of 4 total entries
    sched_test.go:466:
         Step 3: Testing block production scheduling for inactive proposer...
    sched_test.go:472:  Scheduled block time: 40
    sched_test.go:479:
         Step 4: Verifying inactive proposer appears in scheduling sequence...
    sched_test.go:487:  VULNERABILITY CONFIRMED: Inactive proposer scheduled at time 40
    sched_test.go:495:
         Step 5: Testing weight distribution impact...
    sched_test.go:511:  Total weight in sequence: 1100000000
    sched_test.go:512:  Expected weight (active + inactive target): 1100000000
    sched_test.go:518:
         Step 6: Testing Updates function with inactive proposer...
    sched_test.go:521:  Updates count: 1, Score: 10000
    sched_test.go:528:  VULNERABILITY: Inactive proposer marked as active in updates
--- PASS: TestScheduler_InactiveProposerVulnerability_POC (0.00s)
PASS
coverage: 80.0% of statements
ok      github.com/vechain/thor/v2/pos  0.018s  coverage: 80.0% of statements
```

</details>
