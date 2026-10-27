# 56692 sc medium zeroxswapverifier verification will always revert due to wrong hardcoded execution function selectors

**Submitted on Oct 19th 2025 at 14:45:50 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56692
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

`ZeroXSwapVerifier` hardcodes the function selectors called on the 0x Settler contract for calldata verification. Those selectors however are incorrect considering the current Settler contract implementation. This makes all calls to the verifier library revert, making all strategies that rely on it in the future unable to execute and verify 0x swaps, potentially causing DOS on MYT strategies.

## Vulnerability Details

`ZeroXSwapVerifier` hardcodes the following 0x Settler function selectors for verification of incoming swap calldata.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L20-L21>

Those selectors are subsequently checked on every calldata verificaiton call and if the calldata selector does not match, the whole call reverts.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L99-L115>

However, based on the most recent Settler contract on Mainnet, those selectors listed as `0xcf71ff4f` for `execute(..)` and `0x0476baab` for `executeMetaTxn` are actually incorrect. Looking at the 0x Settler contract repository we can find the official implementations on Ethereum mainnet by calling the `ownerOf(id)` function using the ids of 2 (for regular execution) and 3 (for meta tx execution) (ref <https://github.com/0xProject/0x-settler?tab=readme-ov-file#how-do-i-find-the-most-recent-deployment>) on the deployer proxy. This leads us to 2 Settler contracts:

Regular Settler: <https://etherscan.io/address/0x207e1074858A7e78f17002075739eD2745dbaEce#writeContract#F1>

MetaTx Settler <https://etherscan.io/address/0x1e1Ed00F1048C99240bB56defC20de44A0A005Cb#writeContract#F1>

We can observe from the urls that both execution selectors are different than the ones hardcoded in `ZeroXSwapVerifier`. The issue lies in the fact that `execute` and `executeMetaTxn` take in an `AllowedSlippage` struct instead of a `SlippageAndActions` and an additional `bytes32 /* zid & affiliate */,` parameter which even if unused, changes the function signature.

```solidity
    // Constants for 0x Settler function selectors
    // @audit Wrong selector for execute, missing additional bytes32 param
    // interface ISettlerTakerSubmitted is ISettlerBase {
    //     function execute(AllowedSlippage calldata slippage, bytes[] calldata actions, bytes32 /* zid & affiliate */ )
    //         external
    //         payable
    //         returns (bool);
    // }
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    // @audit wrong selector here as well. Missing additional bytes32 param
    // interface ISettlerMetaTxn is ISettlerBase {
    //     function executeMetaTxn(
    //         AllowedSlippage calldata slippage,
    //         bytes[] calldata actions,
    //         bytes32 /* zid & affiliate */,
    //         address msgSender,
    //         bytes calldata sig
    //     ) external returns (bool);
    // }
    bytes4 private constant EXECUTE_META_TXN_SELECTOR = 0x0476baab; // executeMetaTxn(SlippageAndActions,bytes[],address,bytes)
```

The functions can also be found in the respective files in the official 0x repository on github

* <https://github.com/0xProject/0x-settler/blob/6add8e30f495ab0499204052adb05906d657bd46/src/Settler.sol#L114>
* <https://github.com/0xProject/0x-settler/blob/6add8e30f495ab0499204052adb05906d657bd46/src/SettlerMetaTxn.sol#L148-L154>

## Impact Details

The `ZeroXSwapVerifier`, as stated by the protocol team, is expected to be used in strategies that require custom, user-supplied swap actions to execute allocations and deallocations on the MYT vault. One such strategy is Tokemak, where a user can supply their own custom swap calldata for the underlying tolen upon withdrawal. In this case, all calls to the verification contract will revert, making the whole allocation and deallocation functions revert which could expose the MYT vault to losses (in the case where custom calldata would reduce slippage) or DOS the allocation flow entirely (in the case calldata verification is required on every interaction with the vault adapter). Especially in the case of Tokemak strategies, user supplied calldata for withdrawals could be crutial to optimize swaps since idle ETH or USDC funds in the vault might not cover the whole withdrawal (<https://docs.auto.finance/developer-docs/integrating/large-withdrawals#custom-swap-handling>). The problem is also worsened by the fact that the strategy contracts are non-upgradeable so the only way to update the selectors would be to deploy a whole different strategy contract and set it as an adapter to the MYT vault.

## References

ZeroXSwapVerifier function selectors: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L20-L21>

ZeroXSwapVerifier function selector checks and revert: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L99-L115>

Regular Settler: <https://etherscan.io/address/0x207e1074858A7e78f17002075739eD2745dbaEce#writeContract#F1>

MetaTx Settler <https://etherscan.io/address/0x1e1Ed00F1048C99240bB56defC20de44A0A005Cb#writeContract#F1>

## Proof of Concept

## Proof of Concept

1. Add this piece of code at the top of `ZeroXSwapVerifier.t.sol` before the test contract declaration. The interfaces declared are the exact same as in the official 0x settler contracts from etherscan ( <https://vscode.blockscan.com/ethereum/0x1e1Ed00F1048C99240bB56defC20de44A0A005Cb),(https://vscode.blockscan.com/ethereum/0x207e1074858A7e78f17002075739eD2745dbaEce>) ):

```solidity
import "forge-std/console.sol";

interface ISettlerTakerSubmitted {
    struct AllowedSlippage {
        address recipient;
        address buyToken;
        uint256 minAmountOut;
    }
    function execute(
        AllowedSlippage calldata slippage,
        bytes[] calldata actions,
        bytes32 /* zid & affiliate */
    ) external returns (bool);
}

interface ISettlerMetaTxn {
    struct AllowedSlippage {
        address recipient;
        address buyToken;
        uint256 minAmountOut;
    }
    function executeMetaTxn(
        AllowedSlippage calldata slippage,
        bytes[] calldata actions,
        bytes32 /* zid & affiliate */,
        address msgSender,
        bytes calldata sig
    ) external returns (bool);
}
```

2. Add the following test case:

```solidity
    function test_pocExecuteWrongSelector() public {
        ///////////// Test with regular execute selector /////////////
        console.log("execute selector");
        console.logBytes4(ISettlerTakerSubmitted.execute.selector);
        assertNotEq(ISettlerTakerSubmitted.execute.selector, EXECUTE_SELECTOR);
        console.log("Performing the call to execute");
        bytes memory _calldata = abi.encodeWithSelector(
            ISettlerTakerSubmitted.execute.selector,
            ISettlerTakerSubmitted.AllowedSlippage({recipient: owner, buyToken: address(token), minAmountOut: 0}),
            new bytes[](0),
            bytes32(0)
        );
        vm.expectRevert(bytes("IS"));
        ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            owner,
            address(token),
            1000 // 1000 bps = 10% max slippage
        );

        ///////////// Test with executeMetaTxn selector /////////////
        console.log("executeMetaTxn selector");
        console.logBytes4(ISettlerMetaTxn.executeMetaTxn.selector);
        assertNotEq(ISettlerMetaTxn.executeMetaTxn.selector, EXECUTE_META_TXN_SELECTOR);
        console.log("Performing the call to executeMetaTxn");
        bytes memory _calldataMetaTxn = abi.encodeWithSelector(
            ISettlerMetaTxn.executeMetaTxn.selector,
            ISettlerMetaTxn.AllowedSlippage({recipient: owner, buyToken: address(token), minAmountOut: 0}),
            new bytes[](0),
            owner,
            bytes32(0)
        );
        vm.expectRevert(bytes("IS"));
        ZeroXSwapVerifier.verifySwapCalldata(
            _calldataMetaTxn,
            owner,
            address(token),
            1000 // 1000 bps = 10% max slippage
        );
    }
```

3. Run the test: \`\`forge test --match-test test\_pocExecuteWrongSelector -vv\`\`\`
4. The test should pass. We verified both that the selectors are wrong and that the call to `verifySwapCalldata` reverts on both occations
