# #44161 \[SC-Low] Return value of low level call not ckecked in \`Dispatcher.sol\` contract

**Submitted on Apr 17th 2025 at 12:21:27 UTC by @ZestfulHedgehog609 for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44161
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In the `Dispatcher.sol` contract , in the `_dispatch(bytes1 _commandType, bytes calldata _inputs)` function , if the `command` is `Commands.TRANSFER_NATIVE`, the function does a low level call to the recipient address.The problem is however, after the call is done , its returned `boolean` value is not checked to see if the transfer went successfully or not. If the transfer was not successful, this will cause a silent revert and the transaction is marked a successfull even though the recipient did not receive any ether.

This is not a major problem when transferring to an EOA but the problem arises when the transfer is sent to a smart contract that doesn't accept native eth transfer (doesn't have a payable receive function).

If native eth is transferred to a smart-contract that doesn't accept eth (mentioned above),the returned `success` will be false, but the transaction continues without reverting.

## Vulnerability Details

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

in the above mentioned code we can see that the returned `boolean` value is not checked to confirm whether the transfer was successful or not

## Impact Details

1. The transfer fails, but the rest of the function executes.
2. Inconsistent state: The contract assumes the transfer succeeded when it didn’t.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

paste this test in the `test/Routertest.t.sol` file

```solidity
contract Receiver {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }
}

    function test_SendnativeFails() external {
        address sender = makeAddr("sender");
        address recepient = makeAddr("recepient");
        Receiver receiver = new Receiver(recepient);
        deal(sender, 10 ether);
        deal(payable(router), 10 ether);
        console.log("sender balance:", sender.balance);
        bytes memory commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
        bytes memory data = abi.encode(address(receiver), 10 ether);
        bytes[] memory inputs = new bytes[](1);
        inputs[0] = data;
        vm.prank(sender);
        uint256 gasBefore = gasleft();
        router.execute(commands, inputs);
        uint256 gasAfter = gasleft();
        console.log("receiver balance:", address(receiver).balance);
        console.log("sender balance after:", sender.balance);
        console.log("gas spent:", gasBefore - gasAfter);
    }
```

```javascript
Logs
  sender balance: 10000000000000000000
  receiver balance: 0
  sender balance after: 10000000000000000000
  gas spent: 42651

  Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.13s
```

In the test we can see that the test passes even if the receiver did not get any eth.
