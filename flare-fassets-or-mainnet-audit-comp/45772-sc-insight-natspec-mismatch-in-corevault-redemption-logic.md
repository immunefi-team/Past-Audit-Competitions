# #45772 \[SC-Insight] NatSpec Mismatch in CoreVault Redemption Logic

**Submitted on May 20th 2025 at 10:05:51 UTC by @MyssTeeQue for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45772
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CoreVaultFacet.sol>
* **Impacts:**

## Description

## Brief/Intro

In `CoreVaultFacet.sol`, there is a mismatch between the NatSpec comment and the actual code implementation for the `redeemFromCoreVault` function. The NatSpec comment states that "the number of lots, must be larger than `coreVaultMinimumRedeemLots` setting", but the code uses a greater than or equal to (`>=`) comparison:

## Vulnerability Details

```solidity
//@audit-info natspec says "the number of lots, must be larger than `coreVaultMinimumRedeemLots` setting"
// but the code is using `_lots >= minimumRedeemLots`
require(_lots >= minimumRedeemLots, "requested amount too small");
```

The code's implementation is actually safer as it:

1. Allows redemption at exactly the minimum threshold
2. Avoids potential precision issues with strict inequalities

## Impact Details

The code is actually more permissive than documented, not less

* Could cause confusion for developers reading the documentation
* Might lead to unnecessary code changes if someone tries to "fix" the code to match docs

## Proof of Concept

## Proof of Concept

Update the NatSpec comment to accurately reflect the code's behavior and its rationale:

```solidity
// The number of lots must be greater than or equal to the minimum required lots.
// This allows redemption of exactly the minimum amount while avoiding strict inequality precision issues.
```

No code changes are needed as the current implementation using `>=` is the safer approach.
