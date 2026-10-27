# 59411 sc insight inconsistency in migratetokenmanager in terms of the permitted caller

* **Submitted on:** Nov 12th 2025 at 05:41:45 UTC by @Oxodus for [Audit Comp | Vechain | Stargate Hayabusa](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)
* **Report ID:** #59411
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>

## Description

### Brief / Intro

There is an inconsistency between the NatSpec in `IStargateNFT` and the `StargateNFT` implementation regarding who is allowed to call the `migrateTokenManager` function.

* `IStargateNFT` states that "this function is only callable by the DEFAULT\_ADMIN\_ROLE".
* The implementation in `StargateNFT` instead restricts the call to `TOKEN_MANAGER_MIGRATOR_ROLE`:

```solidity
function migrateTokenManager(
    uint256 _tokenId,
    address _manager
) external onlyRole(TOKEN_MANAGER_MIGRATOR_ROLE) {
    ...
}
```

This mismatch between interface documentation and contract implementation can lead to confusion or incorrect assumptions about access control.

## References

* Implementation: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol#L660>
* Interface NatSpec: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/interfaces/IStargateNFT.sol#L467>

## Proof of Concept

No additional PoC provided.
