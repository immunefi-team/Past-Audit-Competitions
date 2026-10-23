# 26422 - \[SC - Insight] there is no explicit gas limit in external call...

Submitted on Dec 2nd 2023 at 19:06:22 UTC by @KaloMen for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26422

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x0d2ec0a5858730e7d49f5b4ae6f2c665e46c1d9d#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

there is no explicit gas limit in external call during executeTransaction on the timelock; execution of a proposed action has the possibility to behave differently with different gasLimit together with the unbounded return bytes

On the timeLock contract, an action is a set combination of:

1. target
2. value
3. calldata (+& funcSelector)
4. eta

On a malicious level,

Consider there is a target with function C, that is due to be called. An external call forwards 63/64 gas to the external contract With EIP150. However since the unbounded bytes is also returned, that means the remaining gas in the main execution can be carefully calculated in the external contract, such that it returns a big enough bytes to consume all of the remaining 1/64 gas.

Therefore It is possible for this function/target to return True, but always make the TimeLock hit OOG on a dynamic gas forwarding scheme since the timeLock needs to return the unbounded bytes as a result too.

On the general level, the execution result of an action without gas limit is also non-deterministic, since the caller may pass in insufficient gas limit and do not able to execute it successfully. Depending on the runtime execution path, the gas limit estimated by the admin may not be sufficient to cover the runtime cost too and cost unnecessary failure.

Alternatively the target can also be engineered in a way that makes dynamic gas forwarding fail as illustrated above.

## Impact

non-deterministic execution result of the proposal. Potential grieving attack from a carefully calculated gas scheme in the external call.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

1. Add gas limit, or max gas limit as part of the action variable included in the hash.
2. Consider not to bubble up the unbounded bytes from the external call.

## Reference

## Proof of concept

just psuedo code to illustrate the logic

```solidity
function executeTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public payable returns (bytes memory) {
...
        // solium-disable-next-line security/no-call-value
        (bool success, bytes memory returnData) = target.call{value: value}(callData);
        require(success, "Timelock::executeTransaction: Transaction execution reverted.");

        emit ExecuteTransaction(txHash, target, value, signature, data, eta);

        return returnData;
    }
```

```solidity
contract Target {

    // this function get forwarded 63/64 gas ; it knows the main execution in`executeTransaction` would emit `ExecuteTransaction` with set amount of gas, and return the bytes returned by this function.

    function action( bytes calldata b) external returns(bytes memory) {
        uint256 gasLeft = gasleft() ;
         uint256 gasToDepleteInMain = gasLeft / 63;
          uint256 gasForEmit = 27863;
         uint256 gasToDepleteInMainByBytes = gasToDepleteInMain - 27863;
         // return the number of bytes that would deplete the above gas
        bytes memory result;
        // 80000 is the gas required to execute 1-loop of appendBytes
        while (gasleft()  > 80000) {
           result = appendBytes(result);
        }
        return result;
    }


    function appendBytes(bytes memory result) internal returns (bytes memory) { 
        return abi.encodePacked(result, bytes("1"));
    }
}
```
