# Attackathon \_ Fuel Network 33407 - \[Smart Contract - Insight] Missing Zero-Check for to Address in w

Submitted on Fri Jul 19 2024 18:17:46 GMT-0400 (Atlantic Standard Time) by @bugtester for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33407

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-bridge/tree/e3e673e31f9e72d757d68979bb6796a0b7f9c8bc/packages/solidity-contracts>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The withdraw function in the smart contract lacks a check to ensure that the recipient address (to parameter) is not a zero addressThis oversight can lead to potential loss of funds by sending tokens to an invalid address.

## Vulnerability Details

in the withdraw function, the recipient address parameter (to) is not validated to ensure it is not a zero address. Sending funds to a zero address is an invalid operation and could result in irreversible loss of tokens.

<https://github.com/FuelLabs/fuel-bridge/blob/623dc288c332b9d55f59b1d3f5e04909e2b4435d/packages/solidity-contracts/contracts/messaging/gateway/FuelERC20Gateway/FuelERC20GatewayV4.sol#L231C2-L234C78>

IERC20MetadataUpgradeable(tokenAddress).safeTransfer(to, amount);

```
//emit event for successful token withdraw
emit Withdrawal(bytes32(uint256(uint160(to))), tokenAddress, amount);
```

## Impact Details

loss of funds

## Proof of concept

## Proof of Concept

function finalizeWithdrawal( address to, address tokenAddress, uint256 l2BurntAmount, uint256 /*tokenId*/ ) external payable virtual override whenNotPaused onlyFromPortal { if (l2BurntAmount == 0) { revert CannotWithdrawZero(); }

```
if (messageSender() != assetIssuerId) {
    revert InvalidSender();
}


uint8 decimals = _getTokenDecimals(tokenAddress);
uint256 amount = _adjustWithdrawalDecimals(decimals, l2BurntAmount);


//reduce deposit balance and transfer tokens (math will underflow if amount is larger than allowed)
_deposits[tokenAddress] = _deposits[tokenAddress] - l2BurntAmount;
IERC20MetadataUpgradeable(tokenAddress).safeTransfer(to, amount);


//emit event for successful token withdraw
emit Withdrawal(bytes32(uint256(uint160(to))), tokenAddress, amount);
```

}

### fix

Add check address check to param
