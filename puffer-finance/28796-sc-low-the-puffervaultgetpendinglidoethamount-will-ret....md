# 28796 - \[SC - Low] The PufferVaultgetPendingLidoETHAmount will ret...

Submitted on Feb 27th 2024 at 16:52:00 UTC by @aman for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28796

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Contract `PufferVault:: $.lidoLockedETH` does not store correct value in case of deposit and withdrawal will leads to return wrong value `PufferVault::getPendingLidoETHAmount`.

## Vulnerability Details

Offer a detailed explanation of the vulnerability itself. Do not leave out any relevant information. Code snippets should be supplied whenever helpful, as long as they don’t overcrowd the report with unnecessary details. This section should make it obvious that you understand exactly what you’re talking about, and more importantly, it should be clear by this point that the vulnerability does exist.

The Protocols integrate Lido staking for earning staking rewards on depositing user ETH at Lido. Here I assume that the assets are staked at Lido correctly. For withdrawal Lido uses Withdrawal queue. The users must initiate Withdrawal request from Lido withdrawal queue. The request code : it stored the requested amount in `$.lidoLockedETH` state varaible.

```javascript
function initiateETHWithdrawalsFromLido(uint256[] calldata amounts)
        [...]
    {
        VaultStorage storage $ = _getPufferVaultStorage();

        uint256 lockedAmount;
        for (uint256 i = 0; i < amounts.length; ++i) {
            lockedAmount += amounts[i];
        }
 @:263       $.lidoLockedETH += lockedAmount;

        SafeERC20.safeIncreaseAllowance(_ST_ETH, address(_LIDO_WITHDRAWAL_QUEUE), lockedAmount);
        requestIds = _LIDO_WITHDRAWAL_QUEUE.requestWithdrawals(amounts, address(this));
   [...]
    }
```

when user claim request from Lido The contract checks if `$.isLidoWithdrawal` is true. than in default `receive()` it decrease the `$.lidoLockedETH` by `msg.value` received. following are the relevant code snippet:

```javascript
  receive() external payable virtual {
        // If we don't use this pattern, somebody can create a Lido withdrawal, claim it to this contract
        // Making `$.lidoLockedETH -= msg.value` revert
        VaultStorage storage $ = _getPufferVaultStorage();
        if ($.isLidoWithdrawal) {
            $.lidoLockedETH -= msg.value;
        }
    }

....

function claimWithdrawalsFromLido(uint256[] calldata requestIds) external virtual {
        VaultStorage storage $ = _getPufferVaultStorage();

        // Tell our receive() that we are doing a Lido claim
        $.isLidoWithdrawal = true;

        for (uint256 i = 0; i < requestIds.length; ++i) {
            bool isValidWithdrawal = $.lidoWithdrawals.remove(requestIds[i]);
            if (!isValidWithdrawal) {
                revert InvalidWithdrawal();
            }

            // slither-disable-next-line calls-loop
            _LIDO_WITHDRAWAL_QUEUE.claimWithdrawal(requestIds[i]);
        }

        // Reset back the value
        $.isLidoWithdrawal = false;
        emit ClaimedWithdrawals(requestIds);
    }

```

## Impact Details

The issue is That STETH is Re-basing token, due to which the the amount requested and amount claimed will not be the same if slashing occurs at Lido. Due to which the `lidoLockedETH` will report a value which in reality is not present or locked.

## References

<https://github.com/PufferFinance/pufETH/blob/main/src/PufferVault.sol#L83> <https://github.com/PufferFinance/pufETH/blob/main/src/PufferVault.sol#L262>

## Proof of Concept
