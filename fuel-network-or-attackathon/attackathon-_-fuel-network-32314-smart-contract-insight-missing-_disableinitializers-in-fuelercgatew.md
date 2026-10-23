# Attackathon \_ Fuel Network 32314 - \[Smart Contract - Insight] Missing \_disableInitializers in FuelER

Submitted on Tue Jun 18 2024 06:05:13 GMT-0400 (Atlantic Standard Time) by @shanb1605 for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32314

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-bridge/tree/623dc288c332b9d55f59b1d3f5e04909e2b4435d/packages/solidity-contracts>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `FuelERC20GatewayV4` contract is a UUPS upgradeable contract. It's missing `_disableInitializers` inside constructor to stop initializing the implementation contract.

## Vulnerability Details

Since there is missing `_disableInitializers` inside constructor. Anyone could initialize the implementation of `FuelERC20GatewayV4` contract.

<https://github.com/FuelLabs/fuel-bridge/blob/623dc288c332b9d55f59b1d3f5e04909e2b4435d/packages/solidity-contracts/contracts/messaging/gateway/FuelERC20Gateway/FuelERC20GatewayV4.sol>

## Impact Details

Openzeppelin advises initializing the UUPS contract implementation, but `FuelERC20GatewayV4` doesn't initialize it under constructor. This leaves anyone to initialize the implementation contract.

## References

<https://forum.openzeppelin.com/t/security-advisory-initialize-uups-implementation-contracts/15301>

## Mitigation

Consider adding this snippet to `FuelERC20GatewayV4.sol`

```solidity
constructor() {
    _disableInitializers()
}
```

## Proof of concept

## Proof of Concept

The POC is tested through tenderly local simulation that proves anyone can initialize the impl contract.

1. Go to Tenderly simulator: <https://dashboard.tenderly.co/project/simulator/>
2. Click on `New Simulation`
3. Insert the impl address on sepolia `0xf6024ccbfbb2201c3d43c0c2bbd162d65d4a07c4`
4. Choose Network as `Sepolia`
5. Now choose `Enter raw input data` and fill ***0xc4d66de80000000000000000000000005a36ec816a51d76542cf45f7f7c24ced5b1671e900000000000000000000000000000000000000000000000000000000***
6. Click on `Simulate Transaction`

***The above input data is call data encoding of initialize(address)***
