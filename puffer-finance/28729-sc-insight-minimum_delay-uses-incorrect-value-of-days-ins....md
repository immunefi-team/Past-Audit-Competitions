# 28729 - \[SC - Insight] MINIMUM\_DELAY uses incorrect value of days ins...

Submitted on Feb 25th 2024 at 09:04:26 UTC by @MrPotatoMagic for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28729

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Timelock.sol contract currently uses a `MINIMUM_DELAY` **constant** value of 7 days. But according the specification/expected behaviour mentioned by the sponsors in the Technical walkthrough (referenced below), the minimum delay is supposed to be 2 days.

## References

<https://youtu.be/u1lYGykS-7g?si=-xVed8AAHZ8NkVHh\\&t=2436>

## Impact Details

Since the minimum delay is expected to be 2 days, when the operation/community multisig try to set a delay in the range \[2,7], the call would revert with `InvalidDelay` error.

Due to this, the issue has been marked as low-severity since the contract fails to deliver promised results and does not abide by its specification/expected behaviour.

## Vulnerability Details

Here is the whole process:

1. First , let's take a look at the MINIMUM\_DELAY constant value in the Timelock.sol contract:

* As we can see, the code is deployed with a constant (immutable) value of 7 days rather than the expected value of 2 days.

```solidity
File: Timelock.sol
101:     uint256 public constant MINIMUM_DELAY = 7 days; 
```

2. This mean that when the operation/community multisig calls the function `setDelay()` through the function `executeTransaction()` with a value in the range \[2,7], the call would revert with the `InvalidDelay` error. This denies the operators/community from setting a value in that range and **forces them to set delay to a value greater than 7 days**.

```solidity
File: Timelock.sol
295:     function _setDelay(uint256 newDelay) internal {
296:         
297:         if (newDelay <= MINIMUM_DELAY) {
298:             revert InvalidDelay(newDelay);
299:         }
300:         emit DelayChanged(delay, newDelay);
301:         delay = newDelay;
302:     }
```

## Proof of Concept

How to use this POC:

* Add the POC to the Timelock.t.sol file.
* Run the test using `forge test --fork-url <ETH_MAINNET_RPC_URL> --match-test testIncorrectMinDelayIssue -vvv`

```solidity
  function testIncorrectMinDelayIssue() public {

        uint256 operationId;
        for (uint256 i = 2;  i <= 7 ; i++) {
            bytes memory callData = abi.encodeCall(Timelock.setDelay, (86400*i));

            operationId++;

            vm.prank(timelock.COMMUNITY_MULTISIG());
            timelock.executeTransaction(address(timelock), callData, operationId);

            assertNotEq(timelock.delay(), 86400*i);
        }
    }
```
