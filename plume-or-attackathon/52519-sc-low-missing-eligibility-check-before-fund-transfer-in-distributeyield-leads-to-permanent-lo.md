# 52519 sc low missing eligibility check before fund transfer in distributeyield leads to permanent loss of yield tokens

**Submitted on Aug 11th 2025 at 11:21:59 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52519
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

The `distributeYield` function transfers yield tokens into the `ArcToken` contract before verifying if any holders are eligible to receive them.

When all token holders are restricted (blacklisted or sanctioned), the function exits early after the transfer, leaving the yield tokens permanently trapped in the contract with no recovery mechanism, resulting in irreversible loss of funds.

### Vulnerability Details

The `distributeYield` function follows a flawed order of operations that creates a permanent fund trap when all holders are restricted from receiving yield.

Problematic execution flow (excerpt):

```solidity
function distributeYield(uint256 amount) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
    // ... validation checks ...
    
    // STEP 1: Transfer funds INTO contract (always executes)
    ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
    yToken.safeTransferFrom(msg.sender, address(this), amount);  // Line 435
    
    // STEP 2: Calculate eligible holders
    uint256 effectiveTotalSupply = 0;
    for (uint256 i = 0; i < holderCount; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) {
            effectiveTotalSupply += balanceOf(holder);
        }
    }
    
    // STEP 3: If no eligible holders, EXIT without distributing
    if (effectiveTotalSupply == 0) {
        emit YieldDistributed(0, yieldTokenAddr);  // Line 449
        return;  // Funds already in contract, now trapped forever!
    }
}
```

The critical flaw is that funds are pulled into the contract (Step 1) before checking if anyone can receive them (Step 2). When `effectiveTotalSupply == 0` (all holders restricted), the function returns early, abandoning the transferred yield tokens in the contract.

No Recovery Mechanism Exists:

* No `withdrawStuckTokens` function
* No admin rescue capability
* No mechanism to redistribute trapped funds later

Thus the contract has no way to return or recover these tokens.

This can occur through normal operations when:

* Global sanctions list is updated to include all holder jurisdictions
* All holders become blacklisted due to regulatory compliance
* A token is created but all initial holders are restricted before first distribution

## Impact Details

Yield tokens become permanently inaccessible once trapped. Each failed distribution adds to the accumulated locked value.

## Proof of Concept

A step-by-step demonstration of permanent fund loss:

{% stepper %}
{% step %}

### Initial Setup

* Deploy `ArcToken` with 3 holders:
  * Alice: 1,000 tokens
  * Bob: 1,000 tokens
  * Charlie: 1,000 tokens
* Set yield token to USDC
* Grant `YIELD_DISTRIBUTOR_ROLE` to distributor address
  {% endstep %}

{% step %}

### Create Restriction Scenario

* Add all three holders to yield blacklist via `YieldBlacklistRestrictions.addToBlacklist()`\
  Or update global sanctions module to restrict all holders
  {% endstep %}

{% step %}

### Attempt Distribution

* Distributor approves 1,000 USDC to `ArcToken` contract
* Call `distributeYield(1000e6)` (1,000 USDC)
  {% endstep %}

{% step %}

### Execution Flow

* Line 435: Contract successfully pulls 1,000 USDC from distributor
* Lines 440–446: Loop calculates `effectiveTotalSupply = 0` (all restricted)
* Line 448: Condition `effectiveTotalSupply == 0` is true
* Line 449: Emits `YieldDistributed(0, USDC)`
* Line 450: Function returns
  {% endstep %}

{% step %}

### Verify Permanent Loss

* Check `USDC.balanceOf(arcToken) == 1,000 USDC`
* No function exists to recover these funds
* Funds are permanently trapped
  {% endstep %}
  {% endstepper %}

The vulnerability is confirmed when yield tokens remain trapped in the `ArcToken` contract after all holders are restricted, with no mechanism to recover them.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L384-L460>
