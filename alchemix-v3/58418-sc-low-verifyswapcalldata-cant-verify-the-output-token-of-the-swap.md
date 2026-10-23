# 58418 sc low verifyswapcalldata cant verify the output token of the swap

**Submitted on Nov 2nd 2025 at 07:03:51 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58418
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

the purpose of `ZeroXSwapVerifier` is to verify the calldata from 0x protocol. but the verifier only verify what is already verified in the 0x internal verification. there are no check if the output of swap calldata is the one that a Strategy wants.

## Vulnerability Details

for instance the `_verifyBasicSellToPool` :

```solidity
    function _verifyBasicSellToPool(bytes memory action, address owner, address targetToken, uint256 maxSlippageBps) internal view {
        (address sellToken, uint256 bps, , , ) = abi.decode(
            _slice(action, 4),
            (address, uint256, address, uint256, bytes)
        );

        require(sellToken == targetToken, "IT");
        require(bps <= maxSlippageBps, "Slippage too high");
    }
```

it only check that the `sellToken` match the `targetToken` , as we know that the `sellToken` is the token a strategy wants to sell, so it is necessary for the strategy to have this token in the first place because if its not then the swap should revert.

the issue is that the prior function does not forward the `buyToken` that is inside `SlippageAndActions` struct. this is a crucial data to make sure the output of a swap match the intent of strategy. if this not checked then a valid calldata from 0x protocol can still be executed even though that the output token completely different.

## Impact Details

it is possible for a swap to result to unintended output token/worthless token because there are no check if the output token is inded what the strategy intent to have after the swap.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L195C1-L203C6>

## Proof of Concept

## Proof of Concept

we can see in the code that the `buyToken` is not checked.

so the PoC can be straight forward. add to `src/test/ZeroXSwapVerifier.t.sol` :

```solidity
    function testBuyTokenNotVerified() public {
        uint256 bps = 1000;
        address recipient = makeAddr("recipient");
        address buyToken = makeAddr("buyToken");
        address randomBuyToken = makeAddr("randomBuyToken");

        bytes memory action = abi.encodeWithSelector(
            BASIC_SELL_TO_POOL,
            address(token),
            bps, // bps
            recipient,
            0,
            ""
        );
        
        // create output buyToken
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: recipient,
            buyToken: buyToken,
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = action;
        
        bytes memory _calldata = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        // verify the _calldata 
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            owner, 
            address(token), 
            bps // 1000 bps = 10% max slippage
        );
        assertTrue(verified);

        // create output randomBuyToken
        saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: recipient,
            buyToken: randomBuyToken,
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = action;
        
        _calldata = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));

        // verify the _calldata 
        verified = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            owner, 
            address(token), 
            bps // 1000 bps = 10% max slippage
        );
        assertTrue(verified);
    }

```

the calldata is verified only if the `sellToken == targetToken` and `bps <= maxSlippageBps` . there are no verification for the buyToken or output token.
