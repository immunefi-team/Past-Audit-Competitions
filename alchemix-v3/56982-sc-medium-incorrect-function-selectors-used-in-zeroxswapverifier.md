# 56982 sc medium incorrect function selectors used in zeroxswapverifier

**Submitted on Oct 22nd 2025 at 11:48:28 UTC by @SOPROBRO for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56982
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Loss of functionality due to misconfiguration

## Description

## Brief/Intro

The ZeroXSwapVerifier has incorrect 0x function selectors, causing all swap verification calls to fail and rendering the contract completely non-functional.

## Vulnerability Details

The `ZeroXSwaoVerifier` is meant to permit 0x swap calldata, however, the constants for the 0x Settler function selectors are wrong, meaning, the entire contract is rendered useless.

For example:

```solidity
bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
```

The correct selector for:

```solidity
execute(SlippageAndActions,bytes[])
```

is:

```
0xe2297b1b
```

These constants are used when constructing calldata for the `decodeAndVerifyActions` and `verifySwapCalldata` functions, resulting in failed or misrouted calls when interacting with the `ZeroXSwapVerifier`, resulting in the following revert `revert("Unsupported function selector");`. Furthermore, the action selectors for different swap types also have wrong selectors

For example:

```solidity
bytes4 private constant TRANSFER_FROM = 0x8d68a156;
```

The correct selector for:

```solidity
transferFrom(IERC20 token, address from, address to, uint256 amount)
```

is:

```
0xd3f54f06
```

## Impact Details

While this contact is not currently implemented, I have been told in discord by @Ov3rkoalafied that we can treat this contact as if it is correctly implemented. Therefore, the impact of this is complete loss of functionality of the `ZeroXSwapVerifier`, which is important for atomic withdrawals from strategies.

## References

Please refer to the discord server for @Ov3rkoalafied response

## Proof of Concept

## Proof of Concept

Add the following test to `ZeroXSwapVerifier.t.sol`, and run in the console `forge test --mt testVerifyUniswapV3VIP_fails -vvvv` to run the test, and see the test will fail with "IS" due to the function selector not matching.

```solidity
function testVerifyUniswapV3VIP_fails() public {
	bytes memory fills = abi.encode(address(token), 100e18);
	bytes memory action = abi.encodeWithSelector(
		UNISWAPV3_VIP,
		spender,
		300, // bps
		3000, // feeOrTickSpacing
		false, // feeOnTransfer
		fills
	);

	ZeroXSwapVerifier.SlippageAndActions memory saa =
		ZeroXSwapVerifier.SlippageAndActions({recipient: spender, buyToken: address(0), minAmountOut: 0, actions: new bytes[](1)});
	saa.actions[0] = action;
	bytes4 correctSelector = bytes4(keccak256("execute(SlippageAndActions,bytes[])"));
	console.logBytes4(correctSelector);
	bytes memory _calldata = abi.encodeWithSelector(correctSelector, saa, new bytes[](0));
	vm.expectRevert(bytes("IS"));
	ZeroXSwapVerifier.verifySwapCalldata(
		_calldata,
		owner,
		address(token),
		1000 // 1000 bps = 10% max slippage
	);
}
```
