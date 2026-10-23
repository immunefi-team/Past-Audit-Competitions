# 28779 - \[SC - Insight] Missing sender address check in receive may lea...

Submitted on Feb 26th 2024 at 21:58:19 UTC by @djxploit for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28779

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The `receive` function of `PufferVault.sol` contract, is meant to receive Ether only from Lido. Hence any other ether sent to the contract (accidentally) will be forever locked in the contract, as it will not be accounted for.

## Vulnerability Details

Add an address check in `receive()` of `PufferVault.sol` to ensure the only address sending ETH being received in `receive()` is the Lido contract.

This will prevent stray Ether from being sent accidentally to this contract and getting locked.

## Impact Details

Ethers will get permanently locked in the PufferVault contract, if they are sent from addresses other than Lido contract. Furthermore it will also affect the accounting of the `totalAssets` functions, as it depends on the ether balance of the contract.

## References

<https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72?utm\\_source=immunefi>

## Proof of Concept

Receive function of PufferVault contract

```
    receive() external payable virtual {
        VaultStorage storage $ = _getPufferVaultStorage();
        if ($.isLidoWithdrawal) {
            $.lidoLockedETH -= msg.value;
        }
    }
```

We can fix it by adding an address check like

```
    receive() external payable virtual {
        VaultStorage storage $ = _getPufferVaultStorage();
        require($.isLidoWithdrawal, "Not allowed");
        $.lidoLockedETH -= msg.value;
    }
```
