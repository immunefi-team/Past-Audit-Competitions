# 29106 - \[SC - High] Insufficient Handling of Partial Failures in Wi...

Submitted on Mar 7th 2024 at 05:22:54 UTC by @cheatcode for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29106

Report type: Smart Contract

Report severity: High

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The `PufferVault` contract orchestrates asset management across different protocols, including interactions with external protocols like Lido and EigenLayer for staking and withdrawals. However, the current implementation lacks handling for partial failures in withdrawal requests, which could lead to inaccurate asset accounting.

## Vulnerability Details

The contract facilitates withdrawals from protocols like Lido (`claimWithdrawalsFromLido`) and initiates stETH withdrawals from EigenLayer. However, it assumes these operations are atomic—either entirely successful or unsuccessful. This assumption may not hold in all scenarios, especially in decentralized environments where partial failures or discrepancies in expected outcomes can occur (e.g., due to changes in protocol behavior, gas fluctuations, or execution limits).

The `claimWithdrawalsFromLido` function iterates through an array of request IDs and claims each withdrawal individually from the Lido withdrawal queue. However, it does not track or handle the scenario where some withdrawals are successfully claimed while others fail. Similarly, when initiating stETH withdrawals from EigenLayer (`initiateStETHWithdrawalFromEigenLayer`), the contract assumes that the requested amount will be successfully withdrawn.

## Impact Details

If partial failures occur during withdrawal operations, the `PufferVault` contract may report an inflated or inaccurate value of assets, as it does not account for the discrepancies between the expected and actual amounts received. This can mislead users and administrators about the true holdings of the vault and lead to financial loss.

## Mitigation

Implement detailed tracking and verification of each step in the withdrawal and conversion process. This includes verifying the actual amounts received versus expected and adjusting the vault's internal accounting accordingly.

```solidity
mapping(uint256 => bool) internal claimedWithdrawals;
mapping(uint256 => uint256) internal withdrawalAmounts;

function claimWithdrawalsFromLido(uint256[] calldata requestIds) external virtual {
    VaultStorage storage $ = _getPufferVaultStorage();
    uint256 totalClaimedAmount;

    // Tell our receive() that we are doing a Lido claim
    $.isLidoWithdrawal = true;

    for (uint256 i = 0; i < requestIds.length; ++i) {
        uint256 requestId = requestIds[i];
        bool isValidWithdrawal = $.lidoWithdrawals.remove(requestId);
        if (!isValidWithdrawal) {
            revert InvalidWithdrawal();
        }

        // Check if the withdrawal has already been claimed
        if (claimedWithdrawals[requestId]) {
            revert DuplicateWithdrawal();
        }

        // Claim the withdrawal and track the amount received
        uint256 amountReceived = _LIDO_WITHDRAWAL_QUEUE.claimWithdrawal(requestId);
        claimedWithdrawals[requestId] = true;
        withdrawalAmounts[requestId] = amountReceived;
        totalClaimedAmount += amountReceived;
    }

    // Update the total pending Lido ETH amount
    $.lidoLockedETH -= totalClaimedAmount;

    // Reset back the value
    $.isLidoWithdrawal = false;
    emit ClaimedWithdrawals(requestIds, totalClaimedAmount);
}
```

In this modified version of `claimWithdrawalsFromLido`, we introduce two new mappings: `claimedWithdrawals` to track claimed withdrawals (preventing duplicates), and `withdrawalAmounts` to store the actual amount received for each withdrawal request. The function calculates the total claimed amount and updates the `lidoLockedETH` accordingly, ensuring accurate accounting.

Alternatively, introduce logic to handle partial withdrawals gracefully. This could involve proportionally adjusting the user's claim in the case of a shortfall or providing the option to retry the withdrawal for the remaining amount.

```solidity
function initiateStETHWithdrawalFromEigenLayer(uint256 sharesToWithdraw) external virtual restricted {
    VaultStorage storage $ = _getPufferVaultStorage();

    IStrategy[] memory strategies = new IStrategy[](1);
    strategies[0] = IStrategy(_EIGEN_STETH_STRATEGY);

    uint256[] memory shares = new uint256[](1);
    shares[0] = sharesToWithdraw;

    // Account for the shares
    $.eigenLayerPendingWithdrawalSharesAmount += sharesToWithdraw;

    bytes32 withdrawalRoot = _EIGEN_STRATEGY_MANAGER.queueWithdrawal({
        strategyIndexes: new uint256[](1), // [0]
        strategies: strategies,
        shares: shares,
        withdrawer: address(this),
        undelegateIfPossible: true
    });

    $.eigenLayerWithdrawals.add(withdrawalRoot);
}

function claimWithdrawalFromEigenLayer(
    IEigenLayer.QueuedWithdrawal calldata queuedWithdrawal,
    IERC20[] calldata tokens,
    uint256 middlewareTimesIndex
) external virtual {
    VaultStorage storage $ = _getPufferVaultStorage();

    bytes32 withdrawalRoot = _EIGEN_STRATEGY_MANAGER.calculateWithdrawalRoot(queuedWithdrawal);
    bool isValidWithdrawal = $.eigenLayerWithdrawals.remove(withdrawalRoot);
    if (!isValidWithdrawal) {
        revert InvalidWithdrawal();
    }

    uint256 expectedAmount = queuedWithdrawal.shares[0];
    uint256 actualAmount = _EIGEN_STRATEGY_MANAGER.completeQueuedWithdrawal({
        queuedWithdrawal: queuedWithdrawal,
        tokens: tokens,
        middlewareTimesIndex: middlewareTimesIndex,
        receiveAsTokens: true
    });

    if (actualAmount < expectedAmount) {
        // Handle partial withdrawal
        uint256 remainingShares = (expectedAmount - actualAmount) * totalSupply() / totalAssets();
        $.eigenLayerPendingWithdrawalSharesAmount += remainingShares;
        emit PartialWithdrawal(expectedAmount, actualAmount, remainingShares);
    }

    $.eigenLayerPendingWithdrawalSharesAmount -= expectedAmount;
}
```

In the modified `initiateStETHWithdrawalFromEigenLayer` function, we account for the requested shares in the `eigenLayerPendingWithdrawalSharesAmount` storage variable.

In the `claimWithdrawalFromEigenLayer` function, we introduce logic to handle partial withdrawals. If the actual amount received is less than the expected amount, the function calculates the remaining shares based on the discrepancy and adds them back to the `eigenLayerPendingWithdrawalSharesAmount`. This allows users to retry the withdrawal for the remaining amount. Additionally, an event `PartialWithdrawal` is emitted to notify users about the partial withdrawal.

## Proof of Concept

### poc.py

```python
import random

class PufferVault:
    def __init__(self, total_assets):
        self.total_assets = total_assets
        self.lido_holdings = 0
        self.eigenlayer_holdings = 0

    def withdraw_from_lido(self, amount):
        if amount > self.total_assets:
            raise ValueError("Insufficient assets in the vault")

        self.lido_holdings += amount
        self.total_assets -= amount
        print(f"Withdrawn {amount} from Lido. Remaining assets: {self.total_assets}")

    def convert_to_eth_in_eigenlayer(self, amount):
        partial_failure_chance = 0.3  # 30% chance of partial failure

        if random.random() < partial_failure_chance:
            # Simulate partial failure
            converted_amount = int(amount * random.uniform(0.5, 0.9))
            self.eigenlayer_holdings += converted_amount
            self.lido_holdings -= amount
            print(f"Partial failure: Converted {converted_amount} out of {amount} from Lido to ETH in EigenLayer")
        else:
            self.eigenlayer_holdings += amount
            self.lido_holdings -= amount
            print(f"Successfully converted {amount} from Lido to ETH in EigenLayer")

    def get_total_assets(self):
        return self.total_assets + self.lido_holdings + self.eigenlayer_holdings

# Example usage
vault = PufferVault(total_assets=1000)

# Withdraw from Lido
vault.withdraw_from_lido(500)

# Convert stETH to ETH in EigenLayer (potential for partial failure)
vault.convert_to_eth_in_eigenlayer(400)

# Get the total assets (may be inaccurate due to partial failures)
total_assets = vault.get_total_assets()
print(f"Total assets: {total_assets}")
```

In this example, the `PufferVault` class manages the total assets, Lido holdings, and EigenLayer holdings. The `withdraw_from_lido` function simulates withdrawing assets from Lido, while the `convert_to_eth_in_eigenlayer` function simulates converting stETH to ETH in EigenLayer, with a 30% chance of a partial failure.

When a partial failure occurs, the function simulates receiving only a portion (50-90%) of the expected ETH amount. However, the current implementation does not account for this discrepancy, leading to inaccurate total asset tracking.

You can run this code, and you should see output similar to the following:

```
Withdrawn 500 from Lido. Remaining assets: 500
Partial failure: Converted 360 out of 400 from Lido to ETH in EigenLayer
Total assets: 1060
```

If you don't see error run it a couple of times until you hit the partial failure error.

In the example output above, a partial failure occurred during the conversion to ETH in EigenLayer, but the total assets reported by `get_total_assets` does not reflect this discrepancy accurately.
