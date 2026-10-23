# 25935 - \[SC - Insight] Permissive Fallback Function

## Permissive Fallback Function

Submitted on Nov 21st 2023 at 15:08:55 UTC by @kankodu for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25935

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## Bug Description

* The permissive fallback function executes successfully whenever a third-party contract attempts to call a function that doesn't exist in the wallet.
* There are multiple contracts and standards that rely on the pattern of calling a handler function on user contracts to ensure they intend to perform a specific action. The assumption is that the user contract would revert if that action was unintentional. This assumption is violated here. For this multi-sig contract, all calls to it that are not its functions would succeed.
* Here's an example of how it can materialize to funds being lost: <https://github.com/code-423n4/2023-06-lybra-findings/issues/215>

## Impact

* Contract fails to deliver promised returns but doesn't lose value.

## Risk Breakdown

## Recommendation

* Be intentional. Multi-sigs usually go with changeable fallback handlers managed by a fallback manager. See [Safe Contracts - Fallback Manager](https://github.com/safe-global/safe-contracts/blob/main/contracts/base/FallbackManager.sol)

## References

* [Ethereum Classic Multisig Issue](https://github.com/EthereumCommonwealth/ethereum-classic-multisig/issues/1)

### Proof of concept

* Initiate a new foundry repo by running forge init. see [here](https://book.getfoundry.sh/projects/creating-a-new-project).
* Add the below code in `test/PermissiveFallback.t.sol` and run it using `forge test --fork-url MAINNET_RPC_URL`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";

contract PermissiveFallback is Test {
    address multisig = 0x2028834B2c0A36A918c10937EeA71BE4f932da52;

    function setUp() public {}

    //This function checks that for any function signature and data, the call would succeed because of the permissive fallback function
    function test_FallbackIsPermissive(bytes memory functionSignatureAndData) public {
        (bool success,) = multisig.call(functionSignatureAndData);
        assertTrue(success);
    }
}

```
