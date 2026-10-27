# 58105 sc medium zeroxswapverifier decodes execute payload with wrong abi bytes vs bytes temporary freezing of funds

**Submitted on Oct 30th 2025 at 17:36:01 UTC by @humaira45 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58105
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

ZeroXSwapVerifier is meant to pre‑validate 0x/Matcha calldata before forwarding it to the 0x Settler/Router. The verifier’s execute branch decodes the payload with the wrong ABI: it expects (SlippageAndActions, bytes) but the canonical 0x Settler uses execute(AllowedSlippage, bytes\[] actions, bytes32). As soon as the second parameter (actions) is non‑empty (the normal case), abi.decode reverts inside the verifier, so any path that relies on verifier→forward fails.

We provide an end‑to‑end Foundry PoC where withdraw/deallocate is gated by ZeroXSwapVerifier. Using a valid execute payload with a non‑empty bytes\[] actions, verifySwapCalldata reverts deterministically, withdraw fails, and funds remain stuck. The freeze persists after vm.warp +1 hour and +24 hours (time‑independent bug).

## Vulnerability Details

What the contract does

* ZeroXSwapVerifier.verifySwapCalldata decodes a 0x Settler call (execute or executeMetaTxn) and is expected to:
  * Decode the top‑level payload,
  * Parse and verify actions,
  * Return true for valid payloads.

Where the bug happens

* In-scope file: src/utils/ZeroXSwapVerifier.sol
  * Constants declare:
    * EXECUTE\_SELECTOR = 0xcf71ff4f // execute(SlippageAndActions,bytes\[])
    * EXECUTE\_META\_TXN\_SELECTOR = 0x0476baab // executeMetaTxn(SlippageAndActions,bytes\[],address,bytes)
  * But the decoder for execute is wrong:
    * \_verifyExecuteCalldata(bytes data, …) currently:
    * (SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes)); // WRONG TYPE
    * It should decode (SlippageAndActions, bytes\[] actions, bytes32 tag) per 0x Settler.
  * Meta variant decodes (SlippageAndActions, bytes\[], address, bytes) correctly.

Why this is a bug (with 0x Settler ABI)

* Canonical 0x Settler (0xProject/0x-settler):
  * Settler.execute(AllowedSlippage slippage, bytes\[] actions, bytes32 tag)
  * SettlerMetaTxn.executeMetaTxn(AllowedSlippage, bytes\[] actions, bytes32, address, bytes)
* Therefore, decoding execute as (SlippageAndActions, bytes) is ABI-incompatible and causes abi.decode to revert on any non‑empty actions array (the normal case). The third parameter (bytes32 tag) is also ignored.

## Impact Details

* Any withdraw/deallocate path that relies on ZeroXSwapVerifier + execute will fail for valid 0x payloads (actions non‑empty) due to the wrong decoder. This temporarily freezes user funds (withdraw reverts) and persists over time until code/params are changed.

## References

* In-scope file: src/utils/ZeroXSwapVerifier.sol
  * Functions: verifySwapCalldata, \_verifyExecuteCalldata, \_verifyExecuteMetaTxnCalldata
* 0x Settler (canonical ABI used by 0xProject):
  * Settler.execute(AllowedSlippage, bytes\[] actions, bytes32)
  * SettlerMetaTxn.executeMetaTxn(AllowedSlippage, bytes\[] actions, bytes32, address, bytes)

## Link to Proof of Concept

<https://gist.github.com/humairar301-droid/d923020002f41b291e1489c34a3a8b85>

## Proof of Concept

## Proof of Concept

What this PoC proves (end‑to‑end)

* A minimal “withdraw router” holds user funds and requires ZeroXSwapVerifier.verifySwapCalldata to pass before releasing tokens (no user‑facing fallback).
* For a valid execute payload with non‑empty bytes\[] actions (as 0x Settler expects), the verifier decodes with the wrong ABI and reverts inside abi.decode.
* Withdraw therefore reverts and funds stay stuck. After vm.warp +1 hour and +24 hours, withdraw still reverts (time‑independent code bug).

Full PoC (single file) Gist: <https://gist.github.com/humairar301-droid/d923020002f41b291e1489c34a3a8b85>

Save as: src/test/ZeroX\_DecodeBug\_Freeze\_PoC.t.sol

How to run

* Medium (≥ 1 hour)
  * forge test -vvv --match-test test\_ExecuteDecodeBug\_WithdrawFreeze\_Persists\_1h --evm-version cancun
* Persistence (≥ 24 hours; not claimed as severity here)
  * forge test -vvv --match-test test\_ExecuteDecodeBug\_WithdrawFreeze\_Persists\_24h --evm-version cancun

Representative results

* test\_ExecuteDecodeBug\_WithdrawFreeze\_Persists\_1h: PASS
  * First withdraw → revert from abi.decode inside \_verifyExecuteCalldata
  * vm.warp +1h → withdraw still reverts
  * Router still holds user funds; user balance unchanged
* test\_ExecuteDecodeBug\_WithdrawFreeze\_Persists\_24h: PASS
  * Same behavior after vm.warp +24h

Why this is in‑scope and feasible

* ZeroXSwapVerifier is explicitly in-scope, and the bounty requested special attention that “in-place verification matches 0x protocol logic.”
* The bug is purely in the verifier’s ABI decoding (not a third-party outage). Any production path that uses execute with actions\[].length > 0 is impacted.
* The PoC demonstrates an end-to-end freeze with no user‑facing fallback. The failure persists over time until code is fixed or configuration is changed.

Suggested remediation

```
Fix execute decoder to match 0x Settler ABI:
    _verifyExecuteCalldata should decode (SlippageAndActions, bytes[] actions, bytes32 tag), and verify top-level fields:
    (SlippageAndActions memory saa, bytes[] memory actions, bytes32 tag) =
    abi.decode(data, (SlippageAndActions, bytes[], bytes32));
        require(saa.recipient == allowedRecipient)
        require(saa.buyToken == expectedTargetToken)
        require(saa.minAmountOut >= configuredMinOut)
        Then validate each action accordingly
Keep meta decoder unchanged (it already uses bytes[]).
Add a unit test that builds execute calldata with a non‑empty bytes[] actions and asserts the verifier does NOT revert.
```
