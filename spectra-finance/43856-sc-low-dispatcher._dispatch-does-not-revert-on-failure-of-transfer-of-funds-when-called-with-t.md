# #43856 \[SC-Low] Dispatcher.\_dispatch() does not revert on failure of transfer of funds when called with the TRANSFER\_NATIVE command

**Submitted on Apr 13th 2025 at 07:37:26 UTC by @Vanshika for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43856
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**

## Description

## Brief/Intro

`Dispatcher._dispatch()` silently fails if native ether transfer returns a false boolean.

## Vulnerability Details

`Dispatcher._dispatch()` should revert if the following case returns a false value:

```
        function _dispatch(bytes1 _commandType, bytes calldata _inputs) internal {
            uint256 command = uint8(_commandType & Commands.COMMAND_TYPE_MASK);
            // function truncted to only show relevant parts.
            . . .
            else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            // call does not revert on failure
            (bool success, ) = payable(recipient).call{value: amount}("");
            }
            . . .
        }
```

In other if-clauses in the same function, a false boolean is reverted with a CallFailed() error as shown below. This is missing when command == TRANSFER\_NATIVE.

```
    if (!success) {
        revert CallFailed();
    }
```

## Impact Details

The function does not revert on failure, which may cause errors if there are other commands relying on its success.

## Proof of Concept

## Proof of Concept

Copy the following into `RouterTest.t.sol` and run with `forge test --mt test_transferNativeRevert -vvvv`.

```
    function test_transferNativeRevert() public {
        address alice = makeAddr("alice");
        bytes memory commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
        bytes[] memory inputs = new bytes[](1);
        inputs[0] = abi.encode(alice, 1e18);
        
        vm.prank(testUser);
        router.execute(commands, inputs);

        // unsuccessful transfer but no revert
        assertEq(address(router).balance, 0);
        assertEq(alice.balance, 0);
    }
```

Expected result:

```
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.23s (147.87ms CPU time)

Ran 1 test suite in 7.25s (7.23s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
