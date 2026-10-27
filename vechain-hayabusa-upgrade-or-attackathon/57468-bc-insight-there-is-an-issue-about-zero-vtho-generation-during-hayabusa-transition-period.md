# #57468 \[BC-Insight] there is an issue about zero vtho generation during hayabusa transition period

**Submitted on Oct 26th 2025 at 13:52:29 UTC by @XDZIBECX for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)

* **Report ID:** #57468
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/v2.3.2...release/hayabusa>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief / Intro

There is an issue that creates a 14-day blackout period where nobody on the network earns any VTHO. When the HAYABUSA fork activates, the runtime calls `StopEnergyGrowth()` in `runtime.go`, which disables the passive VTHO generation system. The new PoS block rewards, however, don't kick in until the transition is complete — requiring 2/3 of validators to stake — which can take the full 14-day transition period. During that window, passive growth is already turned off, but `posActive` is still false so block rewards aren't distributed either. The result is roughly 520 million VTHO that should be generated across the network but never is. Every VET holder stops earning VTHO for up to 14 days. This behavior contradicts VIP-253 and VIP-254, which state VTHO generation changes "with the adoption of DPoS", not immediately at fork activation.

## Vulnerability Details

The VIPs referenced:

* VIP-253: Defines the Hayabusa transition period and states "If the check is successful at hardfork\_block+60,480 this block will become the first DPoS block" (i.e., adoption occurs at hardfork\_block+60,480 minimum).
* VIP-254: States "With the adoption of DPoS consensus, the fixed-rate VTHO generation will be discontinued. Instead, VTHO will be generated dynamically through block rewards" — implying the change happens when DPoS is adopted.

The code, however, calls `StopEnergyGrowth()` at `hardfork_block+0`, freezing passive growth immediately at the fork rather than waiting until DPoS is actually adopted (hardfork\_block+60,480+). This creates a mismatch (60,480 blocks / 7–14 days) between VIP wording and actual code behavior.

Root cause: passive VTHO generation is disabled too early.

At the HAYABUSA fork block, runtime.go executes:

```go
// hayabusa/runtime/runtime.go:138-147
if forkConfig.HAYABUSA == ctx.Number {
    logger.Info("HAYABUSA fork, setting up staker contract")
    if err := state.SetCode(builtin.Staker.Address, builtin.Staker.RuntimeBytecodes()); err != nil {
        panic(err)
    }
    if err := builtin.Energy.Native(state, ctx.Time).StopEnergyGrowth(); err != nil {  << here is the  BUG Called too early
        panic(err)
    }
}
```

`StopEnergyGrowth()` permanently records a stop time. After that timestamp, accounts no longer accumulate VTHO over time from just holding VET:

```go
func (e *Energy) StopEnergyGrowth() error {
    if ts, err := e.GetEnergyGrowthStopTime(); err != nil {
        return err
    } else if ts != math.MaxUint64 {
        return nil
    }

    if err := e.state.EncodeStorage(e.addr, growthStopTimeKey, func() ([]byte, error) {
        return rlp.EncodeToBytes(e.blockTime) << freezes growth at fork time this is the Caps growth at HAYABUSA timestamp
    }); err != nil {
        return err
    }

    e.stopTime = e.blockTime
    return nil
}
```

Once the stop time is set, later balance reads only ever accrue up to that point in time and never beyond it:

```go
// hayabusa/state/account.go:62-78
if a.BlockTime < stopTime {
    timeDiff := uint64(0)
    if blockTime <= stopTime {
        timeDiff = blockTime - a.BlockTime
    } else {
        // if current block time is greater than growth stop time, we are taking the time diff only up to growth stop time.
        timeDiff = stopTime - a.BlockTime  // << Growth frozen at stopTime
    }
    growth.SetUint64(timeDiff)
    growth.Mul(growth, a.Balance)
    growth.Mul(growth, thor.EnergyGrowthRate)
    growth.Div(growth, bigE18)
}
```

At the fork block, passive VTHO is effectively frozen for everyone. At that same time, the network is still considered PoA / not yet active DPoS. In staker protocol:

```go
// hayabusa/builtin/staker/protocol.go:24-29
func (s *Staker) SyncPOS(forkConfig *thor.ForkConfig, current uint32) (Status, error) {
    status := Status{}
    // still on PoA
    if forkConfig.HAYABUSA+thor.HayabusaTP() > current {
        return status, nil  // << here is  Returns posActive=false during transition
    } 
```

`HayabusaTP()` is defined as `8640 * 14` blocks (\~14 days). Until that period ends, `posActive` remains false.

Block rewards are only distributed when `posActive == true`:

```go
if posActive {
    staker := builtin.Staker.Native(state)
    if err := staker.ContractBalanceCheck(0); err != nil {
        return nil, nil, consensusError(fmt.Sprintf("staker sanity check failed while verifying block: %v", err))
    }
    energy := builtin.Energy.Native(state, header.Timestamp())
    if err := energy.DistributeRewards(blk.Header().Beneficiary(), signer, staker, header.Number()); err != nil {
        return nil, nil, err
    }
}
```

`posActive` only flips true after the transition period has elapsed and enough validators have staked (2/3 threshold) via `s.transition()`:

```go
if !status.Active && (thor.HayabusaTP() == 0 || (current-forkConfig.HAYABUSA)%thor.HayabusaTP() == 0) {
    activated, err = s.transition(current)
    if err != nil {
        return status, fmt.Errorf("failed to transition to dPoS: %w", err)
    }
    if activated {
        status.Active = true   // << posActive becomes true here
        status.Updates = true
    }
}
```

Result: at the Hayabusa fork block, the code calls `StopEnergyGrowth()` so passive VTHO issuance stops immediately while the chain is still “on PoA” (posActive == false). For the entire transition window (N+1 through N+120,960 ≈ 14 days), the network mints no new VTHO at all. Only after the transition succeeds does `posActive` flip to true and block rewards begin; issuance resumes only from that point forward. This is the zero-issuance gap during the transition.

## Impact Details

Every VET holder stops earning VTHO for up to 14 days during the transition. Example: a user with 1 billion VET would normally generate about 6.5 million VTHO over those two weeks, but instead they get nothing beyond what they earned the day before the fork. Scaled to \~86 billion VET total, roughly 520 million VTHO never gets created. This is not theft or loss of existing funds, but a tokenomics malfunction that violates VIP-253 and VIP-254 expectations.

## References

All references are included in the vulnerability details (code links and VIP quotes).

## Proof of Concept

The following Go tests demonstrate the behavior and quantify the gap. The tests reproduce phases showing passive growth working before the fork, `StopEnergyGrowth()` being called at the HAYABUSA block, a frozen period during the transition, and the resulting token issuance loss.

{% stepper %}
{% step %}

### PHASE 1: BEFORE HAYABUSA - Passive VTHO growth works normally

* Set up state and params.
* User holds 1 billion VET and accumulates passive growth.
* Check expected VTHO after 1 day (passive growth).
  {% endstep %}

{% step %}

### PHASE 2: HAYABUSA BLOCK - StopEnergyGrowth() is called (THE BUG)

* Create energy instance at HAYABUSA timestamp.
* Call `StopEnergyGrowth()` (simulating runtime.go behavior).
* Verify growth stop time equals HAYABUSA time and that passive VTHO growth is frozen.
  {% endstep %}

{% step %}

### PHASE 3: TRANSITION PERIOD - NO VTHO GENERATION (14 days)

* Fast forward through the entire 14-day transition window.
* Check VTHO at multiple checkpoints (day 1, day 7, day 14) and confirm VTHO remains frozen (no passive growth, no block rewards).
  {% endstep %}

{% step %}

### PHASE 4: BUG IMPACT ANALYSIS

* Compute expected VTHO if passive growth had continued for 14 days.
* Compare with actual (frozen) VTHO to compute per-user loss and network-wide loss (approx. 520,128,000 VTHO for \~86B VET).
  {% endstep %}

{% step %}

### PHASE 5: ROOT CAUSE VERIFICATION

* Verify stopTime was set at HAYABUSA.
* Show energy calculation remains capped (VTHO still frozen 30 days later).
* Conclude the bug — VTHO generation gap exists during the transition.
  {% endstep %}
  {% endstepper %}

### PoC test source (abridged)

```go
// Full test shown in the original report. This file demonstrates:
// - StopEnergyGrowth() being called at HAYABUSA
// - Zero VTHO accrual during the 14-day transition window
// - Calculation of per-user and network-wide VTHO not generated
// - Tests: TestVTHOGenerationGapDuringTransition, TestComparePoAVsTransitionVTHOGeneration,
//   TestBlockRewardsDontStartDuringTransition
```

(Full test code was included in the original report; keep the code blocks and outputs intact in the repository as provided.)

## Test Output / Observed Results

Example logs from running the tests (abridged):

* Pre-HAYABUSA: passive growth worked (e.g., \~432,000 VTHO in 1 day for 1B VET).
* HAYABUSA: `StopEnergyGrowth()` called; growth frozen.
* During transition: Day 1, Day 7, Day 14 — VTHO remains frozen.
* Bug impact: Expected \~6,480,000 VTHO after 14 days for 1B VET, actual frozen at \~432,000 VTHO — VTHO lost per user: \~6,048,000 VTHO.
* Network-wide (86B VET): \~520,128,000 VTHO not generated.
* Tests pass and confirm the described behavior.

Full test run outputs were included in the original report (all tests passed and confirmed the zero-generation gap).

## Conclusion

* The code calls `StopEnergyGrowth()` immediately at the HAYABUSA fork block, freezing passive VTHO growth.
* DPoS block rewards are only distributed when `posActive` becomes true, which only happens after the transition and validator staking threshold is met — potentially up to 14 days later.
* This creates a transition window where neither passive growth nor block rewards are active, resulting in zero VTHO issuance network-wide for that period.
* The behavior conflicts with the VIP wording that indicates VTHO generation changes "with the adoption of DPoS consensus" (i.e., when DPoS is active), not immediately at fork activation.

(Original code snippets, test source, and test outputs are preserved as provided in the report.)
