# #41741 \[SC-Insight] Improper Input Validation in zapInNative Leads to Theft of Residual Funds

**Submitted on Mar 17th 2025 at 23:24:44 UTC by @Nawsanders for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41741
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The zapInNative function in the Zapper contract has a critical flaw: it doesn’t ensure that the ETH provided by the caller (msg.value) is sufficient to cover the swap amounts (swapData0.inputAmount and swapData1.inputAmount). As a result, an attacker can send a small amount of ETH and use leftover WBERA in the contract to perform swaps, stealing those residual funds. If exploited on mainnet, this could drain any WBERA left in the contract from previous operations, leading to direct financial losses for the protocol or its users.

## Vulnerability Details

The vulnerability lies in the zapInNative function, which converts incoming ETH to WBERA and uses the total WBERA balance (including residuals) for swaps without proper validation. Here’s how it works:

The function starts by depositing msg.value into WBERA:

MockWETH(wbera).deposit{value: msg.value}();

It then sets wBeraDebt to the total WBERA balance:

uint256 wBeraDebt = MockWETH(wbera).balanceOf(address(this));

Swaps are executed using swapData0.inputAmount and swapData1.inputAmount, reducing wBeraDebt each time:

token0Debt = \_swap(wbera, token0, swapData0.inputAmount, swapData0.outputMin, address(this));\
wBeraDebt -= swapData0.inputAmount;

There’s no check to ensure msg.value >= swapData0.inputAmount + swapData1.inputAmount.

Because wBeraDebt includes both the caller’s msg.value and any pre-existing WBERA, an attacker can specify large inputAmount values that exceed msg.value, effectively spending the residual funds. The contract assumes all WBERA in its balance is available for use, which is a design flaw.

For example:

Contract has 0.2 ether of residual WBERA.\
Attacker sends 0.1 ether and requests swaps totaling 0.3 ether.\
The contract uses the full 0.3 ether (0.1 from attacker + 0.2 residual) without reverting.\
This lack of input validation allows unauthorized access to residual funds, which should be protected.

## Impact Details

The exploit results in the direct theft of user funds (residual WBERA) held in the Zapper contract. In the PoC:

The attacker sends 0.1 ether but drains 0.2 ether of residual WBERA, receiving 0.1 ether of token0 and 0.1 ether of token1 (total value: 0.2 ether).\
Loss = 0.2 ether of WBERA, stolen with only 0.1 ether provided.\
On mainnet, the severity depends on how much WBERA is left in the contract from prior operations. If significant funds accumulate (e.g., from failed transactions or unclaimed operations), an attacker could drain them all, causing financial loss to the protocol or its users. This matches the in-scope impact: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield".

## References

Affected code: <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>

PoC test file: <https://gist.github.com/Nawsanders/daf8c6d006d49845bff9efa9ebc9bec5>

## Link to Proof of Concept

<https://gist.github.com/Nawsanders/daf8c6d006d49845bff9efa9ebc9bec5>

## Proof of Concept

## Proof of Concept

The following PoC was used to identify and confirm the vulnerability in Zapper.sol. It simulates an attack where an attacker exploits the lack of input validation in zapInNative to steal residual WBERA funds. We ran this test and confirmed the bug with the results below.

Steps:

Deploy mock contracts (MockWETH, MockToken) and Zapper.

Seed Zapper with 0.2 ether of WBERA as residual funds.

Attacker calls zapInNative with 0.1 ether, requesting two swaps of 0.15 ether each (totaling 0.3 ether).

Check balances to confirm the theft of residual funds.

Results Obtained:

When we executed the test using forge test --match-path test/ZapperExploitPoC.t.sol -vvvv, we obtained the following logs, proving the bug exists:

## take a look at the screenshot

WBERA in Zapper before: 200000000000000000 (0.2 ether)\
WBERA in Zapper after: 0 (0 ether)\
Attacker Token0 balance: 100000000000000000 (0.1 ether)\
Attacker Token1 balance: 100000000000000000 (0.1 ether)

These results show that the attacker sent only 0.1 ether but drained the full 0.2 ether of residual WBERA, receiving 0.2 ether worth of tokens (token0 and token1).\
This confirms the vulnerability: the contract allowed the use of residual funds without requiring the attacker to cover the full swap amounts.

File: test/ZapperExploitPoC.t.sol

<https://gist.github.com/Nawsanders/daf8c6d006d49845bff9efa9ebc9bec5>

Run Instructions:

Save the file as test/ZapperExploitPoC.t.sol in a Foundry project.

Run: forge test --match-path test/ZapperExploitPoC.t.sol -vvvv

The logs above were obtained, confirming the bug’s existence.
