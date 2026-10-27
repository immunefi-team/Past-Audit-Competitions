# 26423 - \[SC - Insight] Timelock executeTransaction function will succe...

Submitted on Dec 2nd 2023 at 19:11:25 UTC by @hoshiyari for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26423

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Unrequired additional delays
* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

Where : [Timelock for DepositContractProxy](https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5?utm_source=immunefi#code), [Timelock for ExchangeContractProxy](https://etherscan.io/address/0x0d2ec0a5858730e7d49f5b4ae6f2c665e46c1d9d?utm_source=immunefi#code)

#### Summary

* Function `executeTransaction()` verify the queued transaction and then call the target address with 'data' passed in the argument.

```
        bytes memory callData;

        if (bytes(signature).length == 0) {
            callData = data;
        } else {
            callData = abi.encodePacked(bytes4(keccak256(bytes(signature))), data);
        }

        // if target is a non-existent contract then also it will return true. 
        (bool success, bytes memory returnData) = target.call{value: value}(callData);
        require(success, "Timelock::executeTransaction: Transaction execution reverted.");

        emit ExecuteTransaction(txHash, target, value, signature, data, eta);

```

* Ideal expected behaviour is that the call to the target contract with correct data should be successful and if the call to the target contract fails somehow, the transaction should revert.
* However, that is not the case every time. Assuming that target is mistakenly input with an address that doesnot have any code or any contract that is yet to deploy. The call to that particular address will succeed which is not the ideal contract behaviour.
* Openzeppelin has similar Timelock execution issue open mentioning the same issue of call to non-existent contract.
* Attaching POC for further understanding.

## Impact

* Timelock contract will execute the invalid transaction successfully while failing the expectations of the admin causing to repeat the process and thus adding delay to the protocol.
* Although both queue transaction and execute transaction are under admin control. There could be a loss of admin funds in certain transaction scenario too.
* Transactions with contracts that are yet to deploy like new Tokens, LP of new tokens whose address can be predetermined at risk while interacting with this contract.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

## References

Openzeppelin have this issue open and currently under discussion regarding it's mitigation. <https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3874> - "Consider adding a contract existence check in TimelockController's \_execute function"

## Proof of concept

1. Fork chain in local - anvil -f RPC\_URL
2. Copy following test in foundry environment and run "forge test --match-contract DeGate --evm-version Shanghai -vvvv"

```
pragma solidity 0.8.22;

import {Test,console, console2} from "forge-std/Test.sol";

interface ITimelock{
    function admin() external view returns(address);
    function delay() external view returns(uint256);
    function queueTransaction(address, uint, string memory, bytes memory data, uint256) external returns(bytes32);
    function executeTransaction(address, uint, string memory signature, bytes memory data, uint eta) external payable returns(bytes memory);
}
contract DeGate is Test {
    string RPC_URL = "http://127.0.0.1:8545";
    address public timeLockAdmin; 
    address public timeLockAddress;
    uint public delay;
    function setUp() public {
        uint fork = vm.createFork(RPC_URL);
        vm.selectFork(fork);

        timeLockAddress = 0x0D2eC0a5858730E7D49f5B4aE6f2C665e46c1d9d; // TimelockContract for exchange Proxy
        timeLockAdmin = 0x2028834B2c0A36A918c10937EeA71BE4f932da52;
        delay = ITimelock(timeLockAddress).delay();
    }

    function testCallToUnknownContractFail() public {
        vm.startPrank(timeLockAdmin);
        address target = makeAddr("UnknownContract");
        string memory signature = "upgradeTo(address)";
        uint value = 0;
        uint eta = block.timestamp+delay+10;
        // random data for the sake of test. 
        bytes memory data = abi.encode(address(123));
        bytes32 txHash = ITimelock(timeLockAddress).queueTransaction(target,value,signature, data, eta);
        console2.log("Transaction added to the queue:");
        console2.logBytes32(txHash);
        vm.warp(eta+5);
        console2.log("Fast forward to execution time");
        try ITimelock(timeLockAddress).executeTransaction(target, value, signature, data, eta){
            console.log("Transaction successful despite non existent contract.");
        }
        catch{
            console.log("Transaction unsuccessful.");
        }

    }

}
```
