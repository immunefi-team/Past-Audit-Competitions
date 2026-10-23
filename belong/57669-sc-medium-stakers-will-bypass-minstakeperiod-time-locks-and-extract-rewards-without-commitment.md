# 57669 sc medium stakers will bypass minstakeperiod time locks and extract rewards without commitment through emergency withdrawal mechanism

**Submitted on Oct 28th 2025 at 00:52:01 UTC by @Lavender88 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57669
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The flat 10% emergency withdrawal penalty calculated on inflated share values will cause profitable early exits for stakers as rational actors will deposit before reward distributions, immediately emergency withdraw after rewards are added, and extract value without honoring the stated time lock commitment, contradicting the protocol's long term participation economic model.

### Vulnerability Details

In Staking.sol, the emergency withdrawal mechanism calculates the penalty as a fixed 10% of the current asset value (which includes accumulated rewards), rather than implementing a time weighted penalty or enforcing the minStakePeriod lock. This allows users to:

* Capture reward distributions with minimal lock time
* Pay a penalty that is less than the rewards gained when reward rate exceeds 11.11%
* Bypass the entire purpose of the staking time lock mechanism.

The specific issues:

```solidity
// Line 177 - Penalty calculated on inflated assets, not original deposit
uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);

// Line 182 - No time-lock enforcement (compare to line 251 which checks locks)
_removeAnySharesFor(_owner, shares); // Bypasses minStakePeriod

// Missing check: Should enforce minimum lock duration or time weighted penalty
```

The whitepaper states: time locks to encourage long term participation but the implementation allows immediate exits after reward capture.

### Internal Preconditions

1. Owner needs to call distributeRewards() to add rewards to the vault, increasing share value by at least 11.11%
2. penaltyPercentage needs to remain at default value of 1000 (10%) or below
3. minStakePeriod is set (default 1 day per line 116) but not enforced in emergency flows
4. Sufficient reward distributions must occur regularly (weekly/monthly) for repeated exploitation

### External Preconditions

1. LONG token price needs to remain stable or increase (volatility could affect profitability calculations)
2. Gas prices on BNB chain need to remain below 100 gwei for attack to be economically viable with small deposits
3. No external rate limiting or frontrunning protection on reward distribution transactions

## Attack Path

### Scenario 1: Strategic Timing Attack

Actors:

* Honest User (Victim): Long-term staker following intended protocol behavior
* Rational Actor (Attacker): User optimizing for profit by exploiting emergency withdrawal

{% stepper %}
{% step %}

### Day 0 — Honest user deposit

Honest user deposits 1,000 LONG, gets 1,000 shares, begins 30 day commitment.
{% endstep %}

{% step %}

### Day 28 — Attacker observes pattern

Rational actor monitors on chain and observes reward distribution pattern (occurs at Day 29, 00:00 every month).
{% endstep %}

{% step %}

### Day 28, 23:30 — Attacker deposits

Rational actor deposits 1,000,000 LONG, gets 1,000,000 shares.
{% endstep %}

{% step %}

### Day 29, 00:00 — Owner distributes rewards

Owner calls distributeRewards(120000e18) \[12% reward rate].

* Total assets: 1,001,000 → 1,121,000 LONG
* Total shares: 1,001,000
* Share value: 1.12 LONG per share
  {% endstep %}

{% step %}

### Day 29, 00:01 — Attacker emergency withdraws

Rational actor calls emergencyRedeem(1000000 shares):

assets = 1000000 \* 1.12 = 1,120,000 LONG\
penalty = 1,120,000 \* 0.10 = 112,000 LONG → treasury\
payout = 1,008,000 LONG → attacker\
profit = 8,000 LONG ($800 if LONG = $0.10)

Result: capital is free to be redeployed after \~1 minute (vs 30 days intended).
{% endstep %}
{% endstepper %}

Comparison:

* Honest user: Locked 29 days, will get full 12% rewards when unlocked = 120 LONG profit
* Rational actor: Locked \~30 minutes, got 0.8% rewards net = 8 LONG profit BUT capital free 29.98 days earlier and can be redeployed for additional gains

Key difference: The rational actor can redeploy capital repeatedly between distributions, compounding advantage over long-term stakers.

### Scenario 2: Repeated Exploitation (Demonstrated in PoC)

* Attacker deposits 2,000 LONG
* Owner distributes 267 LONG rewards (13.35% increase)
* Attacker emergency withdraws: gets back 2,040.3 LONG → Profit: 40.3 LONG per cycle
* Repeats immediately (no cooldown enforced)

After 2 cycles: 80.6 LONG profit\
Extrapolated to 52 cycles/year: 2,095.6 LONG annual profit ($209 if LONG = $0.10)

Scale example: If 100 whales each do this with 1M LONG:

* Per cycle profit: 8,000 LONG × 100 = 800,000 LONG
* Annual extraction: 800,000 × 52 = 41.6M LONG ($4.16M at $0.10) This undermines staking commitment and long-term lock goals.

## Impact Details

The protocol suffers complete failure of its staking time lock mechanism:

* Stated Goal (from whitepaper): time locks to encourage long term participation
* Actual Outcome: Rational actors lock capital for minutes/hours instead of days/weeks
* Financial Effect: While treasury receives penalties, the protocol fails to achieve its core tokenomics goal of reducing circulating supply through long term locking.

## References

<https://belongnet.github.io/docs/belong-checkin/whitepaper>

## Proof of Concept

<details>

<summary>PoC: Emergency withdrawal bypass &#x26; profit (single cycle)</summary>

```solidity
function testEmergencyWithdrawalBypassesLockAndProfit() public {
    // Victim deposits and must wait 1 day to withdraw
    vm.prank(victim);
    uint256 victimShares = staking.deposit(1000e18, victim);
    
    // Attacker frontruns reward distribution with large deposit
    vm.prank(attacker);
    uint256 attackerShares = staking.deposit(5000e18, attacker);
    uint256 attackerBalanceBefore = long.balanceOf(attacker);
    
    // Owner distributes rewards (11.1% increase needed for profitability with 10% penalty)
    vm.prank(owner);
    staking.distributeRewards(667e18);
    
    // Attacker immediately emergency withdraws (no lock period)
    uint256 attackerAssets = staking.previewRedeem(attackerShares);
    uint256 penalty = (attackerAssets * staking.penaltyPercentage()) / staking.SCALING_FACTOR();
    uint256 expectedPayout = attackerAssets - penalty;
    
    vm.prank(attacker);
    staking.emergencyRedeem(attackerShares, attacker, attacker);
    
    // Calculate profits
    uint256 attackerBalanceAfter = long.balanceOf(attacker);
    uint256 attackerReceived = attackerBalanceAfter - attackerBalanceBefore;
    int256 attackerProfit = int256(attackerReceived) - 5000e18;
}
```

</details>

<details>

<summary>PoC: Repeated exploitation (multiple cycles)</summary>

```solidity
function testRepeatedExploitation() public {
    uint256 attackerInitialBalance = long.balanceOf(attacker);
    
    // Exploit cycle 1
    vm.prank(attacker);
    uint256 shares1 = staking.deposit(2000e18, attacker);
    
    vm.prank(owner);
    staking.distributeRewards(267e18);
    
    vm.prank(attacker);
    staking.emergencyRedeem(shares1, attacker, attacker);
    
    uint256 balanceAfterRound1 = long.balanceOf(attacker);
    
    // Exploit cycle 2
    vm.prank(attacker);
    uint256 shares2 = staking.deposit(2000e18, attacker);
    
    vm.prank(owner);
    staking.distributeRewards(267e18);
    
    vm.prank(attacker);
    staking.emergencyRedeem(shares2, attacker, attacker);
    
    uint256 finalBalance = long.balanceOf(attacker);
}
```

</details>
