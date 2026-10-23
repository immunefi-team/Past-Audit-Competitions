# #46826 \[SC-Medium] transferFeeWei + Transfers.TRANSFER\_GAS\_ALLOWANCE\` when \`CoreVault::transferToCoreVault()\` is called.

**Submitted on Jun 5th 2025 at 00:18:03 UTC by @OxSCSamurai for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46826
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Agents can game the system by ensuring they always have `msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE` when `CoreVault::transferToCoreVault()` is called.

### Summary:

* Instead of paying the full fee of `transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE`, agents can get a gas refund equivalent to `msg.value - transferFeeWei` which refunds them the `Transfers.TRANSFER_GAS_ALLOWANCE` + any surplus, instead of refunding them *only* the surplus or dust amount.
* The fact that the agent only needs his `msg.value` to be a dust amount more than `transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE` in order to get this incorrect refund and pay less gas than the protocol intended, should draw some scrutiny to the implementation of this mechanism in this instance.
* In fact, an agent would be able to ensure 100% of the time that they pay less gas/fees than intended, by simply sending a dust amount of gas more the fee + gas allowance total.
* Yet, if `msg.value` is exactly fee + gas allowance total, the full `msg.value` is sent to `state.nativeAddress`.

Problem section:

```solidity
        // set the active request
        _agent.activeTransferToCoreVault = redemptionRequestId;
        // pay the transfer fee and return overpaid transfer fee when the difference is larger than gas use
        // (all transfers are guarded by nonReentrant in the facet)
        if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
        // send event
        uint256 transferredUBA = Conversion.convertAmgToUBA(transferredAMG);
        emit ICoreVault.TransferToCoreVaultStarted(agentVault, redemptionRequestId, transferredUBA);
    }
```

To demonstrate what I mean:

#### Current code's issue:

```solidity
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);  							// Takes only fee
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);  	// Returns too much
}
```

* Condition checks for `fee + allowance`
* But only takes `fee`
* Returns everything above `fee`
* Protocol loses intended `allowance`

#### Correct implementation:

```solidity
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE);  // Takes both
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - (transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE));  // Returns true excess
}
```

* Takes both `fee + allowance`
* Returns only true excess
* Matches intended behavior
* Protocol keeps what it should

### Impacts:

The ramifications are limited but notable:

* Easy to exploit by sending slightly more than fee + allowance
* The protocol keeps only the `transferFeeWei` instead of `transferFeeWei + TRANSFER_GAS_ALLOWANCE`
* Agents get back more than they should
* Protocol loses the intended gas allowance on each transaction, i.e. Protocol accumulates less gas than intended
* Economic impact is bounded by `TRANSFER_GAS_ALLOWANCE * number of transactions`, i.e. (Protocol loses `TRANSFER_GAS_ALLOWANCE` amount per transfer)
* Violates the intended economic design
* If `TRANSFER_GAS_ALLOWANCE` is small (e.g., 100k gas), impact is minimal
* Could affect protocol's ability to perform operations

Impact: medium Likelihood: high Severity: medium

#### Impacts in scope:

* Low: Contract fails to deliver promised returns, but doesn't lose value

## Proof of Concept

## Proof of Concept (PoC):

Step by step PoC:

```solidity
// 1. Set up
transferFeeWei (e.g., 0.01 ETH)
TRANSFER_GAS_ALLOWANCE (0.001 ETH)
Add a small amount to trigger the bug (e.g., 0.0001 ETH)

// 2. Send transfer with excess
msg.value = 0.0111 ETH  // This is > (0.01 + 0.001) ETH

// 3. Expected behavior:
Protocol keeps: 0.011 ETH (fee + allowance)
Refund: 0.0001 ETH

// 4. Actual behavior:
Protocol keeps: 0.01 ETH (only fee)
Refund: 0.0011 ETH (0.001 ETH extra returned)
```

* This demonstrates how the protocol loses the gas allowance amount in every transfer where `msg.value > transferFeeWei + TRANSFER_GAS_ALLOWANCE`.

#### Recommended Fix:

* This ensures the protocol keeps both the fee and the gas allowance as intended, while only returning the true excess amount to the user.

```solidity
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
   uint256 totalToKeep = transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE;
   Transfers.transferNAT(state.nativeAddress, totalToKeep);
   Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - totalToKeep);
}
```
