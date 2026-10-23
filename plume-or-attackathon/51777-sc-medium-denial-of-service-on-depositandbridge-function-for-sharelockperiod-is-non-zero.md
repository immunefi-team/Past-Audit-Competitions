# 51777 sc medium denial of service on depositandbridge function for sharelockperiod is non zero

**Submitted on Aug 5th 2025 at 18:57:15 UTC by @kaysoft for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51777
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

According to the README file provided on the contest page:\
After deposits all of the depositors shares are locked to their account for the `shareLockPeriod`.\
Link: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/README.md>

The `depositAndBridge(...)` function tries to call the `deposit(...)` and `bridge(...)` function of the `teller` parameter atomically at which time the `shareLockPeriod` of the `deposit(..)` has not elapsed.

This will cause the `depositAndBridge(...)` function to revert because of the `beforeTransfer(...)` validation.

```solidity
File: TellerWithMultiAssetSupport.sol parent  contract of MultiChainLayerZeroTellerWithMultiAssetSupport.sol and MultiChainHyperlaneTellerWithMultiAssetSupport.sol
/**
     * @notice After deposits, shares are locked to the msg.sender's address
     *         for `shareLockPeriod`.
     * @dev During this time all transfers from msg.sender will revert, and
     *      deposits are refundable.
     */
    uint64 public shareLockPeriod;
```

This issue also affects the `depositAndBridgeOneInch(...)` and `depositAndBridgeOkxUniversal(...)` function of `DexAggregatorWrapperWithPredicateProxy.sol`.

## Vulnerability Details

The `depositAndBridge(...)` function of the `TellerWithMultiAssetSupportPredicateProxy.sol` has a `teller` parameter which is a `CrossChainTellerBase` type.

The `MultiChainLayerZeroTellerWithMultiAssetSupport.sol` and `MultiChainHyperlaneTellerWithMultiAssetSupport` contracts can be supplied to the `depositAndBridge(...)` function of `TellerWithMultiAssetSupportPredicateProxy.sol` as the `teller` parameter since it supports the `CrossChainTellerBase` type.

The `depositAndBridge(...)` function calls the `depositAndBridge(...)` function on the teller parameter.

The `depositAndBridge(...)` function on the `teller` parameter calls its `deposit(...)` function first before calling the `bridge(...)` function.

When the deposit function is called, the shares are locked with the `_afterPublicDeposit(...)` function for `shareLockPeriod` before the `bridge(...)` function is called as shown below.

```solidity
File: CrossChainTellerBase.sol
function depositAndBridge(
        ERC20 depositAsset,
        uint256 depositAmount,
        uint256 minimumMint,
        BridgeData calldata data
    )
        external
        payable
        requiresAuth
        nonReentrant
    {
        if (!isSupported[depositAsset]) {
            revert TellerWithMultiAssetSupport__AssetNotSupported();
        }

        uint256 shareAmount = _erc20Deposit(depositAsset, depositAmount, minimumMint, msg.sender);
        _afterPublicDeposit(msg.sender, depositAsset, depositAmount, shareAmount, shareLockPeriod);//@this locks the shares for shareLockPeriod
        bridge(shareAmount, data);
    }
```

In the `bridge(...)` function, there is a `beforeTransfer(...)` function that validates that `sharesLockPeriod` has been exceeded. This check makes it impossible to execute the `depositAndBridge(...)` function atomically because `sharesLockPeriod` means depositors have to wait for the `sharesLockPeriod` before they can bridge.

```solidity
File: CrossChainTellerBase.sol
function bridge(
        uint256 shareAmount,
        BridgeData calldata data
    )
        public
        payable
        requiresAuth
        returns (bytes32 messageId)
    {
        if (isPaused) revert TellerWithMultiAssetSupport__Paused();

        _beforeBridge(data);

        // Since shares are directly burned, call `beforeTransfer` to enforce before transfer hooks.
        beforeTransfer(msg.sender);

        // Burn shares from sender
        vault.exit(address(0), ERC20(address(0)), 0, msg.sender, shareAmount);

        messageId = _bridge(shareAmount, data);
        _afterBridge(shareAmount, data, messageId);
    }


function beforeTransfer(address from) public view {//@audit reverts for non zero shareLockPeriod
        if (shareUnlockTime[from] > block.timestamp) revert TellerWithMultiAssetSupport__SharesAreLocked();
    }
```

## Impact Details

Denial of service on the:

* `depositAndBridge(...)` function of `TellerWithMultiAssetSupportPredicateProxy.sol`
* `depositAndBridgeOneInch(...)` and `depositAndBridgeOkxUniversal(...)` function of `DexAggregatorWrapperWithPredicateProxy.sol`

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/README.md>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L189C5-L192C1>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupport.sol#L47C5-L53C35>

## Proof of Concept

{% stepper %}
{% step %}
Bob calls the `depositAndBridge(...)` function of `TellerWithMultiAssetSupportPredicateProxy.sol` passing the `MultiChainLayerZeroTellerWithMultiAssetSupport.sol` as the `teller` parameter.
{% endstep %}

{% step %}
Bob's transaction reverts because the `teller.depositAndBridge(...)` call in the `depositAndBridge(...)` transaction reverts.
{% endstep %}

{% step %}
This is due to the non-zero `shareLockPeriod` set for the deposit before bridge in the `MultiChainLayerZeroTellerWithMultiAssetSupport.sol` passed as the `teller` parameter.
{% endstep %}
{% endstepper %}
