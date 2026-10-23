# 28632 - \[SC - Insight] Setting delay at MINIMUM\_DELAY in timelock fails

Submitted on Feb 22nd 2024 at 19:10:48 UTC by @Haxatron for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28632

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Introduction

Due to the use of `>=` instead of `>`, setting the delay exactly at the `MINIMUM_DELAY` will fail.

## Vulnerability Details

When setting a delay for the timelock, there is a `MINIMUM_DELAY` set for the timelock.

```solidity
     * @notice Minimum delay enforced by the contract
     */
    uint256 public constant MINIMUM_DELAY = 7 days;
```

However when checking for this delay, `>=` is used instead of `>`.

```solidity
    function _setDelay(uint256 newDelay) internal {
        if (newDelay <= MINIMUM_DELAY) {
            revert InvalidDelay(newDelay);
        }
        emit DelayChanged(delay, newDelay);
        delay = newDelay;
    }
```

Therefore setting the delay at exactly `MINIMUM_DELAY` will cause the `newDelay <= MINIMUM_DELAY` check to be true and therefore the `setDelay` transaction to revert.

## Impact Details

This can be surprising to users such as the operations multisig, that waited to execute their `setDelay` transaction but their transaction will unexpectedly fail because of the above behaviour.

## References

<https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA?utm\\_source=immunefi#code>

## Proof of Concept

Add this to the `test/unit/Timelock.t.sol` test found in <https://github.com/PufferFinance/pufETH>

```solidity
function test_updating_to_minimum_delay_reverts() public {
        vm.startPrank(timelock.COMMUNITY_MULTISIG());

        // set exactly at minimum delay
        bytes memory minimumDelayCallData = abi.encodeCall(Timelock.setDelay, (7 days));

        uint256 operationId = 1234;

        // update to minimum delay
        (bool success, bytes memory returnData) =
            timelock.executeTransaction(address(timelock), minimumDelayCallData, operationId);

        // confirm the transaction failed
        assertEq(success, false);

        // confirm the delay is not set at minimum delay of 7 days
        assertTrue(timelock.delay() != 7 days, "not equal minimum delay");
    }
```
