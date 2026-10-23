# 50470 sc insight inefficient design in distributeyieldwithlimit arctoken creates unnecessary gas consumption

**Submitted on Jul 25th 2025 at 07:09:41 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50470
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Description

### Brief / Intro

The `distributeYieldWithLimit` function contains a gas inefficiency: it calls the expensive `_isYieldAllowed()` function twice for each token holder — once during the `effectiveTotalSupply` calculation loop and again during the actual distribution loop. This redundant pattern doubles the gas cost for yield restriction checks, creating unnecessary expense.

### Vulnerability Details

#### Root Cause Analysis

The inefficiency stems from duplicate yield allowance checks in two separate loops:

{% code title="vulnerable pattern (pseudo-solidity)" %}

```
```

{% endcode %}

```solidity
function distributeYieldWithLimit(...) external {
    // ... setup code ...
    
    // ❌ FIRST LOOP: Calculate effectiveTotalSupply
    uint256 effectiveTotalSupply = 0;
    for (uint256 i = 0; i < totalHolders; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) { // ❌ First call to _isYieldAllowed
            effectiveTotalSupply += balanceOf(holder);
        }
    }
    
    // ... transfer and setup logic ...
    
    // ❌ SECOND LOOP: Distribute yield  
    for (uint256 i = 0; i < batchSize; i++) {
        uint256 holderIndex = startIndex + i;
        address holder = $.holders.at(holderIndex);

        if (!_isYieldAllowed(holder)) { // ❌ Second call to _isYieldAllowed for same holder
            continue;
        }
        
        uint256 holderBalance = balanceOf(holder);
        if (holderBalance > 0) {
            uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
            if (share > 0) {
                yToken.safeTransfer(holder, share);
                amountDistributed += share;
            }
        }
    }
}
```

* First loop calls `_isYieldAllowed()` for all holders to compute `effectiveTotalSupply`.
* Second loop calls `_isYieldAllowed()` again for holders in the current batch.
* The yield allowance status does not change between these calls.
* `_isYieldAllowed()` is an expensive function (multiple contract calls, storage reads, try/catch blocks).

#### The Gas Waste Problem

* Redundant calls double the gas cost for yield restriction checks.
* When distributing across multiple batches, the gas waste multiplies.
* This results in materially higher gas costs for routine yield distributions.

### Impact Details

#### Gas Consumption Analysis

Each `_isYieldAllowed()` call involves:

* Storage reads for restriction modules
* External contract calls to restriction interfaces
* Try/catch blocks with potential interface mismatches
* Router address validation and global module queries

Real-World Impact:

* Multiple batches -> multiplied gas waste
* Operational cost increase for yield distributions

## References

1. First redundant loop (calculating effectiveTotalSupply): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L516-L522>
2. Second redundant loop (distribution check): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L537-L539>

## Recommended Mitigation

Cache yield allowance results so `_isYieldAllowed()` is called at most once per holder. Example pattern:

{% code title="mitigation (pseudo-solidity)" %}

```
```

{% endcode %}

```solidity
function distributeYieldWithLimit(
    uint256 totalAmount,
    uint256 startIndex,
    uint256 maxHolders
) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant 
  returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed) {
    
    // ... validation logic ...
    
    // ✅ SINGLE LOOP: Calculate effectiveTotalSupply AND cache results
    uint256 effectiveTotalSupply = 0;
    mapping(address => bool) yieldAllowedCache;
    
    for (uint256 i = 0; i < totalHolders; i++) {
        address holder = $.holders.at(i);
        bool isAllowed = _isYieldAllowed(holder); // ✅ Call once per holder
        yieldAllowedCache[holder] = isAllowed;     // ✅ Cache result
        
        if (isAllowed) {
            effectiveTotalSupply += balanceOf(holder);
        }
    }
    
    // ... transfer logic ...
    
    // ✅ DISTRIBUTION LOOP: Use cached results
    for (uint256 i = 0; i < batchSize; i++) {
        uint256 holderIndex = startIndex + i;
        address holder = $.holders.at(holderIndex);

        if (!yieldAllowedCache[holder]) { // ✅ Use cache instead of re-calling
            continue;
        }
        
        uint256 holderBalance = balanceOf(holder);
        if (holderBalance > 0) {
            uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
            if (share > 0) {
                yToken.safeTransfer(holder, share);
                amountDistributed += share;
            }
        }
    }
    
    // ... return logic ...
}
```

Notes:

* Use an in-memory mapping (or array of structs keyed by index) to cache results within the function execution.
* Ensure the caching structure fits within Solidity constraints (memory vs storage).
* This maintains identical behavior while avoiding redundant expensive checks.

## Proof of Concept

To demonstrate the gas inefficiency and improvement:

{% stepper %}
{% step %}

### Measure current implementation

1. Deploy the existing contract.
2. Populate holders and set restrictions so some holders are yield-allowed and others not.
3. Call `distributeYieldWithLimit` for a batch and log gas used.
   {% endstep %}

{% step %}

### Measure mitigated implementation

1. Implement the caching approach in a separate branch or a patched contract.
2. Deploy and repeat the same distribution call with the same state.
3. Log gas used.
   {% endstep %}

{% step %}

### Compare results

1. Compare gas logs from both runs.
2. The patched implementation should show substantially lower gas usage per distribution, especially when there are many holders or many batches.
   {% endstep %}
   {% endstepper %}

***

If you want, I can:

* Produce a concrete Solidity patch applying the caching approach (with attention to memory usage and gas optimization), or
* Create a simple test script to measure gas difference between the current and patched implementations. Which would you prefer?
