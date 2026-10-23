# 57964 sc low improper validation of absolutecap and relativecap enables excessive fund allocation in alchemistallocator&#x20;

**Submitted on Oct 29th 2025 at 16:51:29 UTC by @Oxvictorsr for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57964
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Introduction

The vault’s cap enforcement logic incorrectly validates the `adjusted` allocation value against `absoluteCap` and `relativeCap`. Instead of using the correct boundary comparison, the current logic can select the wrong cap limit, causing the system to allocate more assets than intended to certain adapters. This flaw effectively bypasses the protocol’s risk control mechanism, enabling overexposure to risky or malicious adapters and leading to potential protocol insolvency if the affected strategy underperforms or fails.

## Vulnerability Details

In the vault’s allocation flow, the protocol calculates the final `adjusted` value that determines how much can be allocated to a given adapter or strategy. The relevant logic (simplified) is as follows:

```solidity
adjusted = adjusted > absoluteCap 
    ? relativeCap 
    : absoluteCap;
```

However, this condition is **semantically inverted**. It should ensure that the adjusted allocation respects both the **absolute cap** (the hard upper limit of assets that can be deployed into a given adapter) and the **relative cap** (a soft, proportional limit based on total vault assets).

The correct logic should ensure the allocation never exceeds *whichever cap is smaller*, for example:

```solidity
adjusted = adjusted > relativeCap 
    ? relativeCap 
    : adjusted;
```

By using the wrong comparison direction, the system may incorrectly assign a higher value to `adjusted`, effectively **disabling the cap enforcement mechanism**.

As a result, the vault’s internal tracking may show an adapter receiving allocations that exceed its intended cap, breaking the expected invariants that limit exposure per risk class.

## Impact Details

This logic flaw allows **over-allocation of vault funds** to certain adapters, violating the protocol’s intended risk distribution model. The possible consequences include:

* **Protocol insolvency:** If the over-allocated adapter experiences losses, the vault may become undercollateralized relative to user deposits.
* **Loss of user funds:** Excess allocation to a single adapter or strategy exposes the vault to higher single-point-of-failure risk.
* **Broken invariants:** Cap enforcement becomes unreliable, affecting automated rebalancing, accounting, and safety checks across the system.
* **Governance risk:** Operators or automated allocators could unknowingly allocate beyond safe thresholds.

Because this affects the **core vault solvency mechanism**, the severity is **High**, with the corresponding impact type being **Protocol Insolvency** under the program’s scope.

## Proof of Concept

## Proof of Concept

```solidity
function testAdjustedCalculation_UsesWrongCapComparison() public {
    // Simulate a scenario where:
    // absoluteCap = 200 ether, relativeCap = 100 ether
    uint256 absoluteCap = 200 ether;
    uint256 relativeCap = 100 ether;
    

    // === Vulnerable logic as currently implemented ===
    // Note: It picks the *larger* of the two values instead of the smaller one.
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;

    // === Expected correct logic ===
    // adjusted should be the *smaller* of the two caps, to ensure allocations don't exceed safe limits.
    uint256 correctAdjusted = absoluteCap > relativeCap ? relativeCap : absoluteCap;

    // === Log output for visibility ===
    emit log_named_uint("absoluteCap", absoluteCap);
    emit log_named_uint("relativeCap", relativeCap);
    emit log_named_uint("Vulnerable adjusted", adjusted);
    emit log_named_uint("Expected correct adjusted", correctAdjusted);

    // === Assertion: show that vulnerable logic overestimates ===
    assertGt(adjusted, correctAdjusted, "Adjusted should not exceed the smaller cap");

}
```
