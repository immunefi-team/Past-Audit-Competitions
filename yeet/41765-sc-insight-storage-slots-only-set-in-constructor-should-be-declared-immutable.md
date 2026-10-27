# #41765 \[SC-Insight] Storage slots only set in constructor should be declared \`immutable\`

**Submitted on Mar 18th 2025 at 07:54:52 UTC by @Victor\_TheOracle for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41765
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**

## Description

## Brief/Intro

The issue involves two state variables in the `yeetback.sol` contract that are set only during construction but are not declared as `immutable`. In non-upgradeable contracts, failing to mark such variables as immutable results in unnecessary gas costs since these variables occupy storage slots, potentially increasing the cost of contract interactions on mainnet.

## Vulnerability Details

In Solidity, variables that are assigned a value only once in the constructor and never modified should be declared as immutable. This allows the Solidity compiler to optimize these variables by embedding their values directly into the bytecode rather than storing them in a storage slot.

In `yeetback.sol`, the variables `entropy` and `entropyProvider` are initialized in the constructor but are not declared as `immutable`:

```solidity
/// @dev The entropy contract address
//@audit (info) -----> Should be immutable
IEntropy private entropy;
/// @dev The address of the entropy provider
//@audit (info) -----> Should be immutable
address private entropyProvider;
```

```solidity
constructor(address _entropy, address _entropyProvider) Ownable(msg.sender) {
    require(_entropy != address(0), "Yeetback: Invalid entropy address");
    require(_entropyProvider != address(0), "Yeetback: Invalid entropy provider address");
    entropy = IEntropy(_entropy);
    entropyProvider = _entropyProvider;
}
```

## Impact Details

The main impact of this vulnerability is increased gas consumption during contract execution. By storing these values in storage rather than embedding them in the contract's code, each access to these variables requires an SLOAD operation, which is more gas-intensive.

## References

Relevant Code snippet: <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L33-L35>

## Proof of Concept

## Proof of Concept
