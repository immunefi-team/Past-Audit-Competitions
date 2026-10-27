# #46486 \[SC-Low] Faulty logic in \`transferToCoreVault\` makes users pay more for the refund transaction than the amount being refunded.

**Submitted on May 31st 2025 at 14:38:20 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46486
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`transferToCoreVault` in `CoreVault.sol` incorrectly compares `msg.value` (wei) with `TRANSFER_GAS_ALLOWANCE` (gas units) in the refund threshold check. This creates a meaningless threshold of 100,000 wei (≈0.0000000000001 NAT).

Almost any overpayment triggers a refund that costs users more in gas than they receive back, causing a loss to them.

## Vulnerability Details

In the `transferToCoreVault` function, there is the following logic to refund any excess amount back to the user:

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CoreVault.sol#L83>

```
        if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
```

If `(msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE)` , you refund, otherwise you don't. The comparison with TRANSFER\_GAS\_ALLOWANCE is attempting to ensure there's enough excess value to justify spending the caller's gas on a refund. The logic probably is: "only refund if the amount is large enough that it's worth the caller paying gas to get it back."

But the comparison is incorrect. `msg.value` and `transferFeeWei` are in **wei**, but `Transfers.TRANSFER_GAS_ALLOWANCE` is in **gas units**. This is apparent from how it's being used:

```
    function transferNATAllowFailure(address payable _recipient, uint256 _amount)
        internal
        requireReentrancyGuard
        returns (bool)
    {
        if (_amount > 0) {
            /* solhint-disable avoid-low-level-calls */
            //slither-disable-next-line arbitrary-send-eth
            (bool success, ) = _recipient.call{value: _amount, gas: TRANSFER_GAS_ALLOWANCE}("");
            /* solhint-enable avoid-low-level-calls */
            return success;
        }
        return true;
    }
```

So, we are comparing a value in wei with units of gas. This is incorrect and leads to a loss for the users. The `TRANSFER_GAS_ALLOWANCE` is just 100,000, which is a very small value in wei, and any excess amount will be easily more than this value. Therefore, any slightly excessive amount will end up triggering the refund, even if the gas cost to complete that refund transaction is more than the amount being refunded. So, users end up paying more in gas to support the refund transaction than the refund amount. This is the opposite of what the protocol intends to do.

The protocol needs to use an equivalent value of `TRANSFER_GAS_ALLOWANCE` in wei. Something like `TRANSFER_GAS_ALLOWANCE` \* gas price in wei / gas unit. Then the above comparison becomes correct.

## Impact Details

Every small overpayment triggers a refund call, wasting gas. Users will get refunds even for tiny amounts that cost more in gas than the refund value itself. Hence, users pay more in gas for unnecessary refund attempts than they get back.

Also, if the user believes that the excess amount they pay might be less than the gas cost for a refund, and hence, if they use gas just to complete their transaction, believing that the refund tx will not be included in the function call, will see their transactions being reverted because refund got executed for which they did not pay.

The protocol is harming its own users through economically irrational refund mechanisms. The protocol is essentially **griefing** its own users through faulty logic.

The likelihood becomes high because any excess payment triggers a refund. Instead of just those refunds which are economically viable. The impact is low to medium. Hence, this is a medium severity issue.

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CoreVault.sol#L83>

## Proof of Concept

## Proof of Concept

1. User calls `transferToCoreVault`. Let's say that the `transferFeeWei` for this transaction is 0.01 NAT (10,000,000,000,000,000 wei). User overpays by 0.00001 NAT (10,000,000,000,000 wei). So, the total value sent is equal to 0.01 NAT + 0.00001 NAT = 10010000000000000 wei.
2. Current threshold: 10,000,000,000,000,000 (`transferFeeWei`) + 100,000 (`TRANSFER_GAS_ALLOWANCE`) = 10000000000100000 wei.
3. As the value being sent - 10010000000000000 wei is greater than the threshold 10000000000100000 wei, a refund is triggered. As the refund triggers for overpayment of just 100,001 wei.
4. But, gas cost for refund: \~100,000 gas × gas\_price (likely >> 0.00001 NAT), especially when gas prices are high in the network due to high traffic.
5. Hence, the user actually loses value because of the refund. The protocol is griefing its own users due to faulty logic.
