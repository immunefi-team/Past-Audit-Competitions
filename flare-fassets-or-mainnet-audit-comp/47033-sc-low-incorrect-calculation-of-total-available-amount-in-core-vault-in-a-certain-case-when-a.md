# #47033 \[SC-Low] Incorrect calculation of total available amount in core vault in a certain case when a user redeems from the core vault

**Submitted on Jun 8th 2025 at 08:16:59 UTC by @swarun for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47033
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a user who already has a available non cancelable transfer request again then the total available amount returned is incorrect in that case because it again considers the fee which in reality is not charged again.

## Vulnerability Details

In the case of non cancelable transfer requests fees is only charged when there is a new request added but if a redeemer whose request has already been added again requests a transfer no fees is applied to it but when total available amount in the core vault is calculated it again takens into account the fee which is incorrect.

## Impact Details

It incorrectly charges fees and causes less available amount calculation even when there is certainly enough tokens available.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L220> <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L270>

## Proof of Concept

## Proof of Concept

1. Suppose the user has already called redeem from core vault therefore it will have a transfer request in nonCancelableTransferRequests array

```solidity
 uint256[] private nonCancelableTransferRequests;
```

2. Now suppose a user again calls redeem from core vault with the same destination address so the following calculation for the available amount will be done

```solidity
function getCoreVaultAvailableAmount()
        internal view
        returns (uint256 _immediatelyAvailableUBA, uint256 _totalAvailableUBA)
    {
        State storage state = getState();
        uint256 availableFunds = state.coreVaultManager.availableFunds();
        uint256 escrowedFunds = state.coreVaultManager.escrowedFunds();
        // account for fee for one more request, because this much must remain available on any transfer
        uint256 requestedAmountWithFee =
            state.coreVaultManager.totalRequestAmountWithFee() + getCoreVaultUnderlyingPaymentFee();
        _immediatelyAvailableUBA = MathUtils.subOrZero(availableFunds, requestedAmountWithFee);
        _totalAvailableUBA = MathUtils.subOrZero(availableFunds + escrowedFunds, requestedAmountWithFee);
    }
```

We can clearly see that again fee amount is taken into account while calculating the amount available. Now see in the case of non cancelable requests no new transfer request is added for the available destination address

```solidity
else {
            uint256 index = 0;
            while (index < nonCancelableTransferRequests.length) {
                TransferRequest storage req = transferRequestById[nonCancelableTransferRequests[index]];
==>                if (keccak256(bytes(req.destinationAddress)) == destinationAddressHash) {
                    // add the amount to the existing request
                    req.amount += _amount;
                    _paymentReference = req.paymentReference;   // use the old payment reference when merged
                    break;
                }
                index++;
            }
            nonCancelableTransferRequestsAmount += _amount;
            // if the request does not exist, add a new one
            if (index == nonCancelableTransferRequests.length) {
                nonCancelableTransferRequests.push(nextTransferRequestId);
                newTransferRequest = true;
            }
```

Therefore incorrect calculation of available amount is done. This can also cause less available lots available for example

```solidity
uint256 requestedAmountWithFee =
            state.coreVaultManager.totalRequestAmountWithFee() + getCoreVaultUnderlyingPaymentFee();
        _immediatelyAvailableUBA = MathUtils.subOrZero(availableFunds, requestedAmountWithFee);
        _totalAvailableUBA = MathUtils.subOrZero(availableFunds + escrowedFunds, requestedAmountWithFee);
```

Suppose total available amount after taking into account the fee is less than the lot size then the following function will return zero.

```solidity
 function getCoreVaultAmountLots()
        internal view
        returns (uint64)
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        (, uint256 totalAmountUBA) = getCoreVaultAvailableAmount();
        return Conversion.convertUBAToAmg(totalAmountUBA) / settings.lotSizeAMG;
    }
```

Now hadn't the fees been taken into account it would have returned 1 lot availble which is the correct amount available.
