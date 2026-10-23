# #45533 \[SC-Low] Incorrect gas allowance comparison in CoreVault transfer function leads to user fund loss

**Submitted on May 16th 2025 at 09:31:08 UTC by @DSbeX for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45533
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `CoreVault` contract miscalculates gas costs when refunding excess ETH paid by users for transfers, treating gas units as equivalent to wei. This error causes users to lose ETH when overpaying for transfer fees, as the contract retains funds that should be refunded. If exploited, attackers or users overpaying by even small amounts would permanently lose ETH to the contract.

## Vulnerability Details

The vulnerability arises in the `transferToCoreVault` function of the `CoreVault` library, where the contract checks if `msg.value` exceeds the required transfer fee plus a gas allowance: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L83>

```javascript
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
```

`Transfers.TRANSFER_GAS_ALLOWANCE` represents a gas limit (100\_000 gas units), not wei. The code incorrectly adds it to `transferFeeWei` (in wei), leading to a mismatched comparison. This error systematically under-refunds users, as the gas cost in wei (gas units × tx.gasprice) is not calculated. The contract retains ETH that should be returned, resulting in direct fund loss.

## Impact Details

Direct Theft of User Funds: Users overpaying beyond the miscalculated threshold lose ETH permanently to the contract. Exploitability: No attacker needed—users inadvertently lose funds due to normal contract interaction.

## References

`CoreVault::transferToCoreVault` - <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L83>

`Transfers.sol` - <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/utils/lib/Transfers.sol#L8>

## Proof of Concept

## Proof of Concept

1.User Initiates Transfer:

Calls `transferToCoreVault(amountAMG)` with `msg.value = 0.1005 ETH`. The required transfer fee is 0.1 ETH.

2. Flawed Threshold Check:

Contract calculates:

`msg.value (0.1005 ETH) > transferFeeWei (0.1 ETH) + TRANSFER_GAS_ALLOWANCE` (`100,000 wei = 0.0001 ETH`). → `0.1005 ETH > 0.1001 ETH` → true.

3. Fee Payment:

Contract sends 0.1 ETH to the fee address (`state.nativeAddress`).

Remaining contract balance: `0.1005 ETH - 0.1 ETH = 0.0005 ETH`.

4. Refund Attempt:

Contract attempts to refund `msg.value - transferFeeWei = 0.0005 ETH` to the user.

Gas Cost Calculation:

Refund requires \~21,000 gas (standard ETH transfer).

Gas cost in wei: `21,000 gas * 20 gwei = 420,000,000,000 wei = 0.00042 ETH`.

5. Insufficient Contract Balance:

Total ETH needed for refund: `0.0005 ETH (refund) + 0.00042 ETH (gas) = 0.00092 ETH`.

Contract only has 0.0005 ETH after paying the fee.

6. Refund Failure:

The `transferNATAllowFailure` call fails due to insufficient ETH for gas.

Result:

0.0005 ETH remains trapped in the contract.

User loses their overpayment of 0.0005 ETH.
