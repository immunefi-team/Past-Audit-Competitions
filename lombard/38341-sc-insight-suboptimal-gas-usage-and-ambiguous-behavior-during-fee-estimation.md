# #38341 \[SC-Insight] Suboptimal gas usage and ambiguous behavior during fee estimation

**Submitted on Dec 31st 2024 at 13:10:15 UTC by @security for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38341
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/Bridge.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## Description

## Brief/Intro

The `getAdapterFee` function forwards 228 bytes as payload for fee estimation. However, these bytes are not included in the actual payload or fee calculation during the bridging process. Additionally, if no adapter is set for a destination, the function reverts instead of returning zero.

## Vulnerability Details

When calculating the adapter fee via `getAdapterFee`, the function forwards `new bytes(228)` to `destConfig.adapter.getFee` for fee estimation:

```solidity
destConfig.adapter.getFee(
    toChain,
    destConfig.bridgeContract,
    toAddress,
    amount,
    new bytes(228)
);
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L133)

However, in the implementation of `CLAdapter::_buildCCIPMessage`, the `payload` is ignored:

```solidity
function _buildCCIPMessage(
    bytes memory _receiver,
    uint256 _amount,
    bytes memory _payload
) private view returns (Client.EVM2AnyMessage memory) {
    // ...
    data: "",
    // ...
}
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L169)

Similarly, during deposits, the same `_buildCCIPMessage` is called, which skips the `payload`. As a result, the message used for fee estimation does not include the `payload`:

```solidity
Client.EVM2AnyMessage memory message = _buildCCIPMessage(
    abi.encodePacked(_toAddress),
    _amount,
    _payload
);
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L177)

The following comment in the code is misleading because the 228-byte payload is not actually bridged during deposit:

```solidity
// payload data doesn't matter for fee calculation, only length  
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L126)

Additionally, if no adapter is set in the destination configuration, `getAdapterFee` reverts. Instead, it should return zero for better handling.

### Impact

These issues lead to suboptimal gas usage and ambiguous behavior during fee estimation and payload processing. Addressing them ensures clearer logic and better gas efficiency.

## Proof of Concept

### PoC

The payload length is not included in fee estimation or deposit processing, which is the correct behavior. However, for clarity and gas optimization, the following changes are recommended:

```diff
function getAdapterFee(
    bytes32 toChain,
    bytes32 toAddress,
    uint64 amount
) external view returns (uint256) {
    DestinationConfig memory destConfig = getDestination(toChain);
    if (destConfig.bridgeContract == bytes32(0)) {
        return 0;
    }
+   if (destConfig.adapter == address(0)) {
+       return 0;
+   }

    // payload data doesn't matter for fee calculation, only length
    return
        destConfig.adapter.getFee(
            toChain,
            destConfig.bridgeContract,
            toAddress,
            amount,
-           new bytes(228)
+           new bytes(0)
        );
}
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L116)

```diff
function _buildCCIPMessage(
    bytes memory _receiver,
    uint256 _amount,
-   bytes memory _payload
+   bytes memory
) private view returns (Client.EVM2AnyMessage memory) {
    //...
}
```

[Reference](https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L222)
