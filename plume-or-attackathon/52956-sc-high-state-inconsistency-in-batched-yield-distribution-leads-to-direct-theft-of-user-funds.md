# 52956 sc high state inconsistency in batched yield distribution leads to direct theft of user funds and protocol insolvency

**Submitted on Aug 14th 2025 at 13:37:47 UTC by @Nuesayo for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52956
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

### Brief/Intro

The `distributeYieldWithLimit` function in `ArcToken.sol` has a fundamental architectural flaw: it recalculates the total supply on every batch call using current token balances instead of maintaining a consistent snapshot from distribution initiation. This allows attackers to purchase tokens between batch executions and immediately receive unearned yield, while simultaneously causing the protocol to attempt distributing more funds than available, leading to both direct theft of user funds and potential protocol insolvency.

### Vulnerability Details

The vulnerability exists in the core logic of how batched yield distribution calculates recipient shares. The function is designed to process large holder lists across multiple transactions to avoid gas limits, but it makes a critical error in state management.

Problematic code flow (illustrative):

```solidity
function distributeYieldWithLimit(uint256 totalAmount, uint256 startIndex, uint256 limit) external {
    // Step 1: Transfer entire yield amount only on first call
    if (startIndex == 0) {
        yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
    }
    
    // Step 2: CRITICAL BUG - Recalculate total supply on EVERY call
    uint256 effectiveTotalSupply = 0;
    for (uint256 i = 0; i < totalHolders; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) {
            effectiveTotalSupply += balanceOf(holder); // Uses current balances!
        }
    }
    
    // Step 3: Process batch using current supply calculation
    for (uint256 i = startIndex; i < endIndex; i++) {
        address holder = $.holders.at(i);
        uint256 holderBalance = balanceOf(holder);
        uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
        yToken.safeTransfer(holder, share);
    }
}
```

Fundamental flaw: `effectiveTotalSupply` is recalculated on every batch call using current balances rather than the balances that existed when distribution began. This creates several critical issues:

* State Inconsistency: The denominator used for calculating shares changes between batch calls based on current blockchain state, not the original distribution state.
* New Holder Injection: When tokens are purchased between batches, new addresses are added to the holders array and immediately become eligible for yield they didn't earn.
* Mathematical Impossibility: The sum of all calculated shares can exceed the total available yield amount because each batch calculates shares based on different denominators.

The vulnerability is triggered by any token balance changes between batch calls, which happens constantly through:

* DEX trading activity (Uniswap, Sushiswap, etc.)
* Token transfers between wallets
* Cross-chain bridge completions
* Exchange deposits and withdrawals
* Automated rebalancing protocols

Concrete mathematical demonstration:

Initial State:

* Alice: 1000 tokens, Bob: 2000 tokens, Charlie: 1000 tokens
* Total: 4000 tokens, Yield: 4000 USDC
* Batch size: 2 holders per call

Batch 1 (indices 0-1):

```
effectiveTotalSupply = 1000 + 2000 + 1000 = 4000
Alice share = (4000 * 1000) / 4000 = 1000 USDC
Bob share   = (4000 * 2000) / 4000 = 2000 USDC
Contract balance: 4000 - 3000 = 1000 USDC remaining
```

Between batches: Attacker buys 1000 tokens, is added to holders array

Batch 2 (index 2):

```
effectiveTotalSupply = 1000 + 2000 + 1000 + 1000 = 5000 (WRONG!)
Charlie share = (4000 * 1000) / 5000 = 800 USDC (underpaid by 200)
Contract balance: 1000 - 800 = 200 USDC remaining
```

Batch 3 (index 3 - attacker):

```
effectiveTotalSupply = 5000
Attacker share = (4000 * 1000) / 5000 = 800 USDC (stolen)
Required: 800 USDC, Available: 200 USDC → TRANSACTION REVERTS
```

Correct behavior would use original total supply (4000) for all batches. Recalculating leads to mathematical inconsistency and exploitable states.

## Impact Details

Detailed breakdown of possible losses and impact vectors:

Direct Fund Theft (CRITICAL)

* Attackers can steal yield tokens by acquiring token positions between batch calls.
* Theft amount proportional to tokens acquired and total distribution size.
* Example: In a 10,000 USDC distribution, buying 10% of tokens mid-distribution steals \~1,000 USDC.
* Attack requires minimal capital risk when using flash loans. Profit scales with distribution frequency and size.

Protocol Insolvency (CRITICAL)

* Contract may attempt to distribute more tokens than it holds.
* Later batch transactions revert due to insufficient balance.
* Partial distributions leave the protocol in an inconsistent state.
* Manual intervention or funds injection required.

Systematic Yield Theft from Legitimate Users (HIGH)

* Long-term holders receive reduced yields they rightfully earned.
* Yield reduction proportional to tokens purchased by attackers mid-distribution.
* Time-weighted distribution promise broken.

MEV Exploitation Opportunity (HIGH)

* Predictable profit opportunity for MEV bots monitoring distributions.
* Flash loans can enable capital-free attacks.

Business Logic Breakdown (MEDIUM)

* Time-weighted yield distribution is effectively broken.
* Loss of user trust and potential legal/fiduciary implications.

Quantified Loss Examples:

* Small Distribution (1,000 USDC): attacker buys 500 tokens mid-batch → theft 100–300 USDC depending on timing.
* Large Distribution (100,000 USDC): attacker buys 5,000 tokens mid-batch → potential theft 10,000–25,000 USDC and high insolvency risk.

The issue is triggered under normal operation due to constant token movement; exploitation is not purely theoretical.

## Link to Proof of Concept

<https://gist.github.com/Ayoseun/12aedb2f4d9fb04e18b5b702bb3175ce>

## Proof of Concept

### Prerequisites

* ArcToken contract deployed with yield distribution functionality.
* Multiple token holders exist in the system.
* Yield distribution requires multiple batches due to gas limits.
* Active token trading occurs on DEXs (normal blockchain activity).

{% stepper %}
{% step %}

### Step 1: Initial Setup

* Deploy ArcToken with holders: Alice (1000 tokens), Bob (2000 tokens), Charlie (1000 tokens)
* Total token supply: 4000 tokens
* Prepare yield distribution: 4000 USDC tokens (1:1 ratio)
* Set batch size to 2 holders per transaction
  {% endstep %}

{% step %}

### Step 2: Distribution Initiation

* Call `distributeYieldWithLimit(4000, 0, 2)`
* This processes the first batch (Alice and Bob)
* Contract receives 4000 USDC total
* Alice receives 1000 USDC: (4000 \* 1000) / 4000
* Bob receives 2000 USDC: (4000 \* 2000) / 4000
* Contract balance remaining: 1000 USDC
  {% endstep %}

{% step %}

### Step 3: Token State Change

* Between batch 1 and batch 2, attacker purchases 500 tokens from Uniswap
* Attacker is added to the holders array
* Current holders: Alice (1000), Bob (2000), Charlie (1000), Attacker (500)
* Total token supply remains 4000, but now distributed among 4 holders
  {% endstep %}

{% step %}

### Step 4: Second Batch Execution

* Call `distributeYieldWithLimit(4000, 2, 2)`
* This processes Charlie and the Attacker
* Function recalculates `effectiveTotalSupply = 4000` (current total based on holders)
* Expected per current logic:
  * Charlie: (4000 \* 1000) / 4000 = 1000 USDC
  * Attacker: (4000 \* 500) / 4000 = 500 USDC
* Total required for this batch: 1500 USDC
* Contract balance: 1000 USDC → Transaction fails or causes inconsistencies
  {% endstep %}

{% step %}

### Step 5: Impact Demonstration

Fund Theft Calculation:

* Legitimate distribution expected: Alice (1000), Bob (2000), Charlie (1000)
* Actual attempted distribution (with attacker): Alice (1000), Bob (2000), Charlie (1000), Attacker (500) = 4500 USDC
* Excess distribution: 500 USDC stolen from the protocol

Alternative (successful theft without causing immediate revert):

* Attacker buys 200 tokens instead of 500
* Attacker receives 200 USDC unearned, can immediately sell back — net theft with minimal capital
  {% endstep %}

{% step %}

### Step 6: Verification

* Check contract token balance before and after each batch
* Compare intended vs actual distribution amounts
* Observe transaction reverts when insufficient funds remain
* Monitor holder array changes between batch calls
  {% endstep %}
  {% endstepper %}

<details>

<summary>Expected vs Actual Behavior</summary>

* Expected: Each holder receives yield proportional to their token balance at the time distribution was initiated, regardless of when batches execute.
* Actual: Each holder receives yield based on their token balance at the time their specific batch executes, allowing new holders to claim unearned yield and potentially causing protocol insolvency.

</details>

### Notes

* The core fix is to capture and use a consistent snapshot of eligible holders' balances (or an aggregate effective supply) at distribution initiation and use that snapshot for all batch calculations, rather than recomputing from live balances on each batch.
* Do not change links or URLs when referencing resources above.
