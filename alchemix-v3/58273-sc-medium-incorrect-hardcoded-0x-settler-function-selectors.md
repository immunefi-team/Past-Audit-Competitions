# 58273 sc medium incorrect hardcoded 0x settler function selectors

**Submitted on Oct 31st 2025 at 21:42:23 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58273
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The ZeroXSwapVerifier library attempts to validate calldata targeted at 0x Settler execute(...) and executeMetaTxn(...) functions by matching hardcoded bytes4 selectors:

```sol
bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
bytes4 private constant EXECUTE_META_TXN_SELECTOR = 0x0476baab;
```

These hardcoded values do not match the actual function selectors used by the 0x Settler ABI. As a result, verifySwapCalldata(...) will either reject legitimate 0x Settler calldata or incorrectly route calldata to the wrong decoder path — causing immediate reverts or incorrect decoding.

The correct selectors are

```sol
EXECUTE_SELECTOR = 0x1ff991f -> "execute((address,address,uint256),bytes[],bytes32)"

EXECUTE_META_TXN_SELECTOR = 0xfd3ad6d4 -> "executeMetaTxn((address,address,uint256),bytes[],bytes32,address,bytes)"
```

The incorrect selectors used are

```sol
EXECUTE_SELECTOR = "executeMetaTxn((address,address,uint256),bytes[],address,bytes)" - 0xcf71ff4f 

EXECUTE_META_TXN_SELECTOR = "setDescription(uint128,string)" - 0x0476baab
```

## Vulnerability Details

The verifySwapCalldata currently does:

```sol
 function verifySwapCalldata(bytes calldata calldata_, address owner, address targetToken, uint256 maxSlippageBps) external view returns (bool verified) {
        if (calldata_.length < 4) {
            return false;
        }

        bytes4 selector = bytes4(calldata_[0:4]);

        // Check if it's a valid 0x Settler function
        require(selector == EXECUTE_SELECTOR || selector == EXECUTE_META_TXN_SELECTOR, "IS");

        decodeAndVerifyActions(calldata_, owner, targetToken, maxSlippageBps);
        return true;
    }
```

Because the hardcoded values are wrong:

Valid 0x Settler transactions fail the require check and revert with "IS".

Calls in \_verifyExecuteCalldata and \_verifyExecuteMetaTxnCalldata will revert or perform incorrect decoding when legitimate flows rely on this verifier.

## Impact Details

Calls that should pass verification will be rejected, blocking users or integrations that rely on this verifier.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol#L20>

## Proof of Concept

Add the code below to src/test/ZeroXSwapVerifier.t.sol

Run forge test --mt testVerifyExecute1 -vvvv

```sol
function _buildExecute(TestERC20 _token, address recipient, uint256 bps) internal pure returns (bytes memory) {
        bytes memory action = abi.encodeWithSelector(
            BASIC_SELL_TO_POOL,
            address(_token),
            bps, // bps
            recipient,
            0,
            ""
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa =
            ZeroXSwapVerifier.SlippageAndActions({recipient: recipient, buyToken: address(0), minAmountOut: 0, actions: new bytes[](1)});
        saa.actions[0] = action;

        return abi.encodeWithSignature("execute(SlippageAndActions,bytes[])", saa, new bytes[](0));

        //return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
    }

    function testVerifyExecute1() public {
        bytes memory _calldata = _buildExecute(token, spender, 500); // 500 bps = 5% slippage
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            owner,
            address(token),
            1000 // 1000 bps = 10% max slippage
        );
        assertTrue(verified);
    }
```
