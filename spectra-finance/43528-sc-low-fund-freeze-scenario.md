# #43528 \[SC-Low] fund freeze scenario

**Submitted on Apr 7th 2025 at 19:20:26 UTC by @Shahen for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43528
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

In the `Dispatcher._dispatch()` function, If the passed command is TRANSFER\_NATIVE and the recipient cant accept native tokens, instead of reverting the whole transaction the native tokens will be stuck in the contract where Dispatcher.sol is inherited to. Mitigation is simple by checking the return value of the low-level call in L485 (Dispatcher.sol).

I have attached a coded foundry poc, Make a new test file under /test and paste the test code. Run `forge test --match-path <testfile-name> -vvv`

## Vulnerability Details

Same as above brief intro

## Impact Details

Freezing of native tokens if recipient address cant accept natives.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: BUSL-1.1

pragma solidity 0.8.20;

import "forge-std/Test.sol";
import {Dispatcher} from "../../src/router/Dispatcher.sol";
import {Commands} from "../../src/router/Commands.sol";


contract A is Dispatcher {

    constructor() Dispatcher(address(0x3)) {}
    function with_val(bytes calldata inputs) external payable {
        bytes1 commandType = bytes1(uint8(Commands.TRANSFER_NATIVE));
        _dispatch(commandType,inputs);
    }


}

contract B {
    //cant accept natives
}

contract freezeTest is Test {

   
    A public _a;
    B public _b;
    

    function setUp() public {
        
        _a = new A();
        _b = new B(); //recipient

    }


    function test_freeze() public {
        
        deal(address(this),1e18);
        assertEq(address(this).balance, 1e18);
        bytes memory _inputs = abi.encode(address(_b),uint256(1e18));


        //1. calls _dispatch() with command TRANSFER_NATIVE and a certain msg.value
        //2. If the recipient cant accept native tokens, instead of reverting the whole transaction the native tokens will be stuck in the contract where Dispatcher.sol is inherited to.
        _a.with_val{value: 1e18}(_inputs);
        assertEq(address(_a).balance, 1e18);
        assertEq(address(_b).balance, 0);

        //3. Mitigation is simple, check the return value of the lowlevel call in L485 (Dispatcher.sol)

    }

}
```
