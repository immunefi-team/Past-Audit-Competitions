# 28689 - \[SC - Medium] incorrect lidoLockedETH value can block full re...

Submitted on Feb 24th 2024 at 00:39:16 UTC by @MahdiKarimi for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28689

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

since the claimed withdrawal amount from Lido is lower than the requested amount ( and lidoLockedETH ), it creates a situation in which lidoLockedETH is increased over time and leading to over-calculation of the vault share price.

## Vulnerability Details

In most cases stETH is always cheaper than ETH, when a withdrawal request is initiated, the requested withdrawal amount is added to lidoLockedETH and claim withdrawal would subtract the claimed amount from lidoLockedETH, however since the claim amount is less than the requested withdrawal, so lidoLockedETH won't be cleared, lidoLockedETH is increased over time and since it's used to calculate total assets it would lead to over calculation of vault shares (puffETH), and would block full redeeming of vault shares since total assets > balance ( assuming there is no stake in eigen layer or withdraw queue ).

## Impact Details

It has two impacts : since lidoLockedETH would increase over time with each withdrawal request it can effect contract calculations. 1 - full redeeming of puffETH gets blocked 2 - over calculation of total assets which inflates share price without backup reserve

## References

Scenario : 1 - Alice deposits 10 ETH and receives 10 puffETH (considering she is the only depositor ) 2 - 10 ETH is staked on lido and after a time we have 11 stETH balance in the vault 3 - a withdraw request is initiated and 11 stETH is added to LidoLockedETH 4 - The lido withdrawal request is claimed and 10.999 ETH will be added to the contract balance and gets subtracted from LidoLockedETH 5 - now total assets = 10.999 ETH + 0.0001 lidoLockedETH = 11 ETH but contract balance is 10.999 ETH 6 - now if Alice decides to withdraw 10 puffETh, 10 vault shares = 11 ETH 7 - since contact doesn't have 11 ETH balance it can't fully repay Alice so she needs to redeem lower shares.

## Proof of Concept

```
function test_lido_withdrawal_lidoLockedETH()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), address(pufferVault), 2000 ether) // Blast got a lot of stETH
    {
        // Queue 2x 1000 ETH withdrawals on Lido
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1000 ether; // steth Amount
        amounts[1] = 1000 ether; // steth Amount
        vm.startPrank(OPERATIONS_MULTISIG);
        uint256[] memory requestIds = pufferVault.initiateETHWithdrawalsFromLido(amounts);

        // Finalize all 2 withdrawals and fast forward to +10 days
        _finalizeWithdrawals(requestIds[1]);
        vm.roll(block.number + 10 days);

        // This one should work
        pufferVault.claimWithdrawalsFromLido(requestIds);

        // since there is no stake amount in lido, eigen layer and withdrawal request we expect total assets = balance 
        assertEq(address(pufferVault).balance, pufferVault.totalAssets();, 'balance of puffer vault is loweer than total assets'));
        
    }
```
