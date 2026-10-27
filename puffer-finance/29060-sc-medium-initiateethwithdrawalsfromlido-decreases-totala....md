# 29060 - \[SC - Medium] initiateETHWithdrawalsFromLido decreases totalA...

Submitted on Mar 6th 2024 at 02:04:33 UTC by @OxDEADBEEF for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29060

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

When a user deposits stETH into the PufferVault - he will receive shares based on the total assets the vault holds.

The assets amount calculation adds all the floating eth values that are deposited into other platforms for yield and current stETH and eth balance.

When an operator calls `initiateETHWithdrawalsFromLido` - the vault transfers stETH to LIDO and increases an internal counter (`lidoLockedETH`).

However - the amount transferred and the amount the counter is increment can be different.

## Vulnerability Details

The main issue is with how `stETH` balance is calculated. There is a known 1-2 wei corner case (<https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case>) which impacts the exact number of tokens moved.

Therefore when `initiateETHWithdrawalsFromLido` is called `lidoLockedETH` can be incremented to a higher value then the actual amount of `stETH` moved.

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
```

## Impact Details

the call to `initiateETHWithdrawalsFromLido` will decrease totalAssets().

Depositors that deposit before `initiateETHWithdrawalsFromLido` will receive more shares then depositors that deposit after `initiateETHWithdrawalsFromLido`

## Proof of Concept

This POC shows that a depositor before `initiateETHWithdrawalsFromLido` will earn more shares then a deposit after `initiateETHWithdrawalsFromLido`.

Add the following test to `test/Integration/PufferTest.integration.t.sol`

```solidity
    function testBug() external giveToken(BLAST_DEPOSIT, address(stETH), alice, 1000 ether) {
        // Deposit to vault 1000 ether
        vm.startPrank(alice);
        SafeERC20.safeIncreaseAllowance(IERC20(stETH), address(pufferVault), type(uint256).max);
        uint256 aliceMinted = pufferVault.deposit(1000 ether, alice);
        vm.stopPrank();

        // Calculate shares to receive before withdrawal request
        uint256 sharesBeforeWithdrawalRequest = pufferVault.previewDeposit(1000 ether);

        // Operator request to withdraw 1000 ether from LIDO 
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1000 ether; // steth Amount
        vm.prank(OPERATIONS_MULTISIG);
        uint256[] memory requestIds = pufferVault.initiateETHWithdrawalsFromLido(amounts);
        
        // Calculate shares to receive after withdrawal request
        uint256 sharesAfterWithdrawalRequest= pufferVault.previewDeposit(1000 ether);

        // Validate that sharesBeforeWithdrawalRequest is smaller then sharesAfterWithdrawalRequest
        assertGt(sharesBeforeWithdrawalRequest, sharesAfterWithdrawalRequest);
    }
```
