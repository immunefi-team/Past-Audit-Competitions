# 57749 sc low zeroxswapverifier misses critical sender recipient minout validations allowing malicious 0x calldata to drain funds critical direct theft&#x20;

**Submitted on Oct 28th 2025 at 16:49:01 UTC by @humaira45 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57749
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The ZeroXSwapVerifier library is intended to validate 0x/Matcha calldata before forwarding it to an executor (i.e., 0x Settler/Router). However, the current implementation does not enforce critical invariants:

* It does not validate SlippageAndActions.recipient, buyToken, or minAmountOut at the top level.
* For several actions, notably TRANSFER\_FROM, it only checks token == targetToken and ignores:
  * The source address (from) is the whitelisted owner,
  * The destination address (to) is a safe recipient,
  * Any amount or minAmountOut limiting bound.

As a result, an attacker can craft 0x calldata that passes verification and, once forwarded to the executor, transfers tokens directly from the approved source to the attacker. This enables direct theft of funds when used as intended (verify → forward).

## Vulnerability Details

What the contract does (intended)

* decodeAndVerifyActions and verifySwapCalldata parse 0x Settler calldata (execute / executeMetaTxn) and are expected to validate:
  * Actions types are allowed and within bounds,
  * Sender/recipient semantics (from is whitelisted source; destination is safe),
  * buyToken/minAmountOut are sane and match the intended swap outcome.

Where verification fails

* Top-level fields are never validated:
  * SlippageAndActions.recipient is ignored.
  * SlippageAndActions.buyToken is ignored.
  * SlippageAndActions.minAmountOut is ignored.
* TRANSFER\_FROM action (\_verifyTransferFrom) only checks token == targetToken, but does not verify:
  * from == owner (the whitelisted source address provided to the verifier),
  * to == recipient or any safe recipient,
  * amount bounds/minOut.
* Other actions are also too permissive:
  * \_verifySellToLiquidityProvider: only checks sellToken == targetToken.
  * \_verifyRFQVIP: ignores amount entirely.
  * \_verifyUniswapV3VIP: ignores recipient/buyToken/minAmountOut.

Root cause (code level)

```
src/utils/ZeroXSwapVerifier.sol:
    _verifyExecuteCalldata / _verifyExecuteMetaTxn: no checks on saa.recipient, saa.buyToken, saa.minAmountOut.
    _verifyTransferFrom: decodes (token, from, to, amount) but only requires token == targetToken; owner/from/recipient/amount are not enforced.
```

## Impact Details

Severity: Critical — Direct theft of funds

* An attacker can craft a 0x payload containing a TRANSFER\_FROM action that pulls tokens from a source address (with allowance) directly to the attacker’s address. ZeroXSwapVerifier “approves” this payload, and once the calldata is forwarded to the 0x executor, the transfer occurs.
* This maps to “Direct theft of funds” in the program taxonomy.

## References

* In-scope file: src/utils/ZeroXSwapVerifier.sol
  * Functions: verifySwapCalldata, \_verifyExecuteCalldata, \_verifyExecuteMetaTxn, \_verifyAction, \_verifyTransferFrom
* Test reference (their tests): src/test/ZeroXSwapVerifier.t.sol shows TRANSFER\_FROM verification returns true without binding from/to.

## Link to Proof of Concept

<https://gist.github.com/humairar301-droid/0f349aa30540edb06c7a620a2ab0de83>

## Proof of Concept

## Proof of Concept

What this PoC proves (end-to-end)

* A production-like harness ZeroXSwapExecutorHarness verifies the payload via ZeroXSwapVerifier and then forwards the same calldata to a simulated 0x executor (FakeSettlerSimulator).
* The payload includes a TRANSFER\_FROM action that moves tokens from a funded owner to an attacker.
* The verifier returns true (payload “valid”), and the executor successfully transfers tokens to the attacker.
* We also prove:
  * The verifier still returns true when action.from != owner (mismatch ignored),
  * The meta-transaction variant (executeMetaTxn) is equally vulnerable.

PoC files (add to repo)

* This Gist contains the PoC: <https://gist.github.com/humairar301-droid/0f349aa30540edb06c7a620a2ab0de83>
* src/utils/ZeroXSwapExecutorHarness.sol
* Minimal executor that verifies and forwards the calldata to an external “settler.”
* Bubbles up revert reasons if the settler reverts.
* src/test/ZeroXSwapVerifier\_ExecutorDrain.t.sol
* End-to-end tests:
  * test\_ZeroXVerifier\_EndToEnd\_DrainViaExecutor (execute selector, recovers funds directly to attacker) test\_ZeroXVerifier\_VerifyAllowsMismatchedFromAndRecipient\_StillDrains (from != owner still verifies and drains) test\_ZeroXVerifier\_EndToEnd\_DrainViaMetaTxn (executeMetaTxn selector, also drains)

How to run

* Commands:
  * forge test --match-test test\_ZeroXVerifier\_EndToEnd\_DrainViaExecutor -vvvv --evm-version cancun
  * forge test --match-test test\_ZeroXVerifier\_VerifyAllowsMismatchedFromAndRecipient\_StillDrains -vvvv --evm-version cancun
  * forge test --match-test test\_ZeroXVerifier\_EndToEnd\_DrainViaMetaTxn -vvvv --evm-version cancun

Representative results (from your traces)

```
Verifier returns true for both execute and executeMetaTxn payloads.
After executor forwards the payload to FakeSettlerSimulator:
    ERC20.transferFrom(owner → attacker, amount) succeeds.
    Asserts confirm attacker’s balance increases by AMOUNT.
```

Why this is in-scope and feasible

* ZeroXSwapVerifier is explicitly in-scope, and Alchemix requested dedicated attention to ensure in-place verification matches 0x logic (sender/recipient/amount/slippage).
* The exploit does not rely on edge conditions or oracle manipulation; it is purely a verification-logic gap.
* The repository already follows a Permit2-based approval pattern in strategies (MYTStrategy manages Permit2 and sets approvals). It is realistic that, after verification, an executor/settler has the ability to pull tokens per the payload.

Suggested remediation

Fail-closed design and minimal checks

* Top-level (execute/executeMetaTxn):
  * require(saa.recipient == expectedRecipient) — bind to a safe recipient (e.g., the calling strategy or a known sink).
  * require(saa.buyToken == targetToken) — ensure intended output asset is enforced.
  * require(saa.minAmountOut >= configuredBound) — enforce slippage and minimum output.
  * Thread through allowedRecipient to all action verifiers.
* TRANSFER\_FROM:
  * require(from == owner) — enforce the only whitelisted source is used.
  * require(to == allowedRecipient) — ensure funds cannot be directed to arbitrary addresses.
  * Optionally enforce amount bounds (or validate against minAmountOut).
* Other actions (RFQ VIP, SellToLiquidityProvider, UniswapV3 VIP, Velodrome V2 VIP):
  * Validate sender/recipient/token path and minAmountOut/bps as appropriate for each action’s ABI.
  * Reject (fail-closed) action types you cannot fully parse/validate now.

Optional hardening

```
Add explicit unit tests that expect revert on:
    action.from != owner,
    saa.recipient not in allowed set,
    buyToken mismatch or minAmountOut == 0,
    Unsupported action types or incomplete parsing.
Keep 0x function signatures/types in sync with 0x upstream to avoid decoding drift.
```
