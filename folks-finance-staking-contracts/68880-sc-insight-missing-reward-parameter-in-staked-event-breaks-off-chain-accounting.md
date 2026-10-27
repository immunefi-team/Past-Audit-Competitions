# 68880 sc insight missing reward parameter in staked event breaks off chain accounting

**Submitted on Mar 11th 2026 at 19:35:35 UTC by @Oxb4b for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68880
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

## Brief/Intro

The `Staked` event emits the deposited `amount` but omits the calculated `reward` value. This creates an observability gap where off-chain systems (indexers, dashboards, analytics) cannot accurately reconstruct user rewards from event logs alone. When staking period parameters are updated between stake execution and off-chain processing, the reconstructed reward will differ from the actual on-chain value, leading to incorrect accounting and user-facing data.

## Vulnerability Details

**Affected Code**: `Staking.sol`, `_stake()` function

**Current Implementation**:

```solidity
// Reward calculated using CURRENT period parameters
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;

// Reward stored immutably in UserStake
userStakes[msg.sender].push(
    UserStake({
        amount: amount,
        reward: reward,  // ← Stored on-chain
        claimedAmount: 0,
        claimedReward: 0,
        aprBps: stakingPeriod.aprBps,
        stakeTime: uint64(block.timestamp),
        unlockTime: uint64(block.timestamp) + stakingPeriod.stakingDurationSeconds,
        unlockDuration: stakingPeriod.unlockDurationSeconds
    })
);

// Event emission WITHOUT reward
emit Staked(msg.sender, periodIndex, params.referrer, stakeIndex, amount);
//                                                                 ^^^^^^
//                                                                 Missing: reward
```

**Root Cause**:\
The `Staked` event signature (defined in `IStakingV1.sol`) is:

```solidity
event Staked(
    address indexed user,
    uint8 indexed periodIndex,
    address indexed referrer,
    uint8 stakeIndex,
    uint256 amount
    // Missing: uint256 reward
);
```

**Attack Path / Observability Failure Scenario**:

1. **T₀**: User stakes 10 ETH with Period 0 (APR = 50%, duration = 30 days)
   * On-chain: `reward = 10 * 5000 * 30 days / (1e4 * 365 days) ≈ 0.4109 ETH` stored in UserStake
   * Event emitted: `Staked(user, 0, referrer, 0, 10 ether)` — no reward field
2. **T₁**: Manager updates Period 0 parameters (legitimate governance action)
   * Period 0 APR changed from 50% → 10%
   * `StakingPeriodUpdated` event emitted
3. **T₂**: Off-chain indexer processes `Staked` event from T₀
   * Indexer reads current Period 0 parameters via RPC: `aprBps = 1000` (10%)
   * Indexer reconstructs: `reward = 10 * 1000 * 30 days / (1e4 * 365 days) ≈ 0.0821 ETH`
   * **Discrepancy**: 0.0821 ETH (reconstructed) vs 0.4109 ETH (actual) — **80% error**
4. **Impact**: User dashboard shows incorrect reward, audit trails fail, accounting mismatch

## Impact and Likelihood Details

**Impact**: **Low** — No on-chain funds at risk, no user loss, no exploitable attack vector

**Affected Systems**:

* Off-chain indexers (TheGraph subgraphs, custom backends)
* User-facing dashboards showing staking positions
* Analytics platforms tracking protocol metrics
* Accounting/audit systems reconciling rewards
* Mobile/light clients unable to make RPC calls

**Likelihood**: **High** — Occurs whenever:

* Staking periods are updated (routine governance operation)
* Network congestion delays event processing
* Indexer catches up on historical events after period updates

## Link to Proof of Concept

<https://gist.github.com/tharunbethina/497b3263c659393a585118e2dbf2c74c>

## Proof of Concept

1. User stakes 10 ETH in Period 0 (50% APR, 30-day duration)
2. On-chain reward calculated and stored: **0.4109 ETH**
3. Manager updates Period 0 APR to 10% (legitimate governance)
4. Off-chain indexer reconstructs reward from event data: **0.0821 ETH** (uses updated APR)
5. Assertion: `reconstructedReward ≠ actualReward` → **80% discrepancy**

Foundry Testcase gist:

<https://gist.github.com/tharunbethina/497b3263c659393a585118e2dbf2c74c>
