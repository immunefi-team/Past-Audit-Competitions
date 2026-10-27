# 29054 - \[SC - Medium] Lido discounted withdrawals are not accounted for

Submitted on Mar 5th 2024 at 22:27:05 UTC by @OxDEADBEEF for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29054

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Protocol insolvency
* Permanent freezing of funds

## Description

## Brief/Intro

The `PufferVault` stakes `ETH` into `LIDO` to gain yield. The operator can request withdrawals from LIDO and anyone can claim them. The current implementation assumes the amount requested is the amount that will be claimable and updates the vaults accounting respectively.

However - `LIDO` can provide a lower (discounted) amount from the requested amount. This will cause the vault to calculate as if it has more assets then it really has - leading to an inflated share/asset ratio.

## Vulnerability Details

`pufferVault` calculates the `totalAssets()` based on all floating ETH value. This includes stETH that is locked in `LIDO` for withdrawing by checking `$.lidoLockedETH`:

```solidity
     * @dev See {IERC4626-totalAssets}.
     * Eventually, stETH will not be part of this vault anymore, and the Vault(pufETH) will represent shares of total ETH holdings
     * Because stETH is a rebasing token, its ratio with ETH is 1:1
     * Because of that our ETH holdings backing the system are:
     * stETH balance of this vault + stETH balance locked in EigenLayer + stETH balance that is the process of withdrawal from Lido
     * + ETH balance of this vault
     */
    function totalAssets() public view virtual override returns (uint256) {
        return _ST_ETH.balanceOf(address(this)) + getELBackingEthAmount() + getPendingLidoETHAmount()
            + address(this).balance;
    }

    function getPendingLidoETHAmount() public view virtual returns (uint256) {
        VaultStorage storage $ = _getPufferVaultStorage();
        return $.lidoLockedETH;
    }
```

This appears fine because it is expected that LIDO will release ETH from stETH at a 1:1 value. However - this is not true. In slashing events on LIDO - when claiming withdrawals, there can be a "discounted" amount.

<https://stake.lido.fi/withdrawals/request>

```
Why is the claimable amount may differ from my requested amount?

The amount you can claim may differ from your initial request due to a slashing occurrence and penalties. For these reasons, the total claimable reward amount could be reduced.
```

Withdraw request and claim:

```solidity
    function initiateETHWithdrawalsFromLido(uint256[] calldata amounts)
        external
        virtual
        restricted
        returns (uint256[] memory requestIds)
    {
        VaultStorage storage $ = _getPufferVaultStorage();

        uint256 lockedAmount;
        for (uint256 i = 0; i < amounts.length; ++i) {
            lockedAmount += amounts[i];
        }
        $.lidoLockedETH += lockedAmount;

        SafeERC20.safeIncreaseAllowance(_ST_ETH, address(_LIDO_WITHDRAWAL_QUEUE), lockedAmount);
        requestIds = _LIDO_WITHDRAWAL_QUEUE.requestWithdrawals(amounts, address(this));

        for (uint256 i = 0; i < requestIds.length; ++i) {
            $.lidoWithdrawals.add(requestIds[i]);
        }
        emit RequestedWithdrawals(requestIds);
        return requestIds;
    }

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

    receive() external payable virtual {
        // If we don't use this pattern, somebody can create a Lido withdrawal, claim it to this contract
        // Making `$.lidoLockedETH -= msg.value` revert
        VaultStorage storage $ = _getPufferVaultStorage();
        if ($.isLidoWithdrawal) {
            $.lidoLockedETH -= msg.value;
        }
    }
```

Notice the above behavior where:

1. `$.lidoLockedETH` is increased by ***REQUESTED*** withdraw value
2. `$.lidoLockedETH` is deducted by ***RECEIVED*** amount.

As explained above - in slashing events, the ***RECEIVED*** amount will be less then the ***REQUESTED*** amount.

This means that `$.lidoLockedETH` will hold a value that is not retrievable. `totalAssets()` will be higher then the actual value the contract has.

## Impact Details

Since `totalAssets()` will be inflated - the share/asset ratio will be incorrect. Currently withdrawals are not allowed.

1. If withdrawals are allowed - there can be an insolvency issue where users cannot withdraw/redeem all their shares.
2. Deposits will be minted an incorrect share amount

## Proof of Concept

Here is proof showing that `totalAssets` does not change after taking the loss and of the claim.

Add the following test to `test/Integration/PufferTest.integration.t.sol`

```solidity
    function testBug() external giveToken(BLAST_DEPOSIT, address(stETH), alice, 1000 ether) {
        bytes32 _VAULT_STORAGE_LOCATION = 0x611ea165ca9257827fc43d2954fdae7d825e82c825d9037db9337fa1bfa93100;

        // Deposit to vault 1000 ether
        vm.startPrank(alice);
        SafeERC20.safeIncreaseAllowance(IERC20(stETH), address(pufferVault), type(uint256).max);
        uint256 aliceMinted = pufferVault.deposit(1000 ether, alice);
        vm.stopPrank();

        // Operator request to withdraw 1000 ether from LIDO 
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1000 ether; // steth Amount
        vm.prank(OPERATIONS_MULTISIG);
        uint256[] memory requestIds = pufferVault.initiateETHWithdrawalsFromLido(amounts);
        
        // Save total ETH backing before claim
        uint256 backingETHAmountBeforeLoss= pufferVault.totalAssets();

        // MOCK pufferVault.claimWithdrawalsFromLido(requestIds[0]);
        // Lets assume LIDO only returns 900 discounted ether instead of 1000
        // The funds will be sent to the receive function therefore we need to
        // send 900 ether to the contract and decrease $.lidoLockedETH by 900.

        // Loads current lidoLockedEth (should be 1000 ether)
        uint256 lidoLockedEth = uint256(vm.load(address(pufferVault), _VAULT_STORAGE_LOCATION));
        assertEq(lidoLockedEth, 1000 ether);

        // Reduce lidoLockedEth by 900 ether
        vm.store(address(pufferVault), _VAULT_STORAGE_LOCATION, bytes32(lidoLockedEth - 900 ether));

        // Validate lidoLockedEth is 100 ether
        lidoLockedEth = uint256(vm.load(address(pufferVault), _VAULT_STORAGE_LOCATION));
        assertEq(lidoLockedEth, 100 ether);
        
        // Send pufferVault the discounted eth (900 ether)
        deal(address(pufferVault), 900 ether);

        // Get totalAssets after claim and loss of 100 ether.
        uint256 backingETHAmountAfterLoss = pufferVault.totalAssets();
        
        // See that even though there is a 100 ether loss (funds not backed by vault anymore) - totalAssets stays the same
        // As if it still holds the lost value.
        assertEq(backingETHAmountBeforeLoss, backingETHAmountAfterLoss);
    }
```

To run the test:

```
forge test --match-test "testBug()"
```
