# 29099 - \[SC - Insight] Actual amount of stETH deposited is less than t...

Submitted on Mar 7th 2024 at 02:56:41 UTC by @kaysoft for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29099

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Due to stETH's [1-2 corner case](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case) the actual amount transfered from user is when the `depositStETH()` function is less than the specified amount in the parameter.

## Vulnerability Details

In the function below, `permitData.amount` of `stETH` is transfered from the `msg.sender` to the the `PufferDepositor` contract. The issue is that the actual amount transferred from msg.sender to the contract is less than the specified `permitData.amount`.

The reason for this is described in [1-2 wei stETH corner case](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case)

```
File: PufferDeposit.sol
function depositStETH(Permit calldata permitData) external restricted returns (uint256 pufETHAmount) {
        try ERC20Permit(address(_ST_ETH)).permit({
            owner: msg.sender,
            spender: address(this),
            value: permitData.amount,
            deadline: permitData.deadline,
            v: permitData.v,
            s: permitData.s,
            r: permitData.r
        }) { } catch { }

192:        SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);

        return PUFFER_VAULT.deposit(permitData.amount, msg.sender);//@audit use transfershares: https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case
    }
```

For example is you specify `permitData.amount` as `1000000000000000000001` the actual amount that will be pulled from the user is `1000000000000000000000`.

This leaves 1 wei of stETH that is not pulled from msg.sender.

Why this is an issue is that there are 2 transfers:

1. First from msg.sender to the `PufferDepositor` contract
2. And secondly from the `PufferDepositor` to the PufferVault contract.

1 wei of stETH difference may seem small but the [doc](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case) further stated that:

> The same thing can actually happen at any transfer or deposit transaction. >In the future, when the stETH/share rate will be greater, the error can >become a bit bigger.

## Impact Details

Possible Denial of service due to transfer amount difference.

## Recommendation

Short Term: Consider using the `transferShares()` function to transfer stETH as recommended by [Lido docs](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case)

Long Term: Instead of `stEth`, consider integration the non-rebasable value-accruing counterpart `wstETH` as recommeded by the [Lido docs](https://docs.lido.fi/guides/lido-tokens-integration-guide/#wsteth)

## References

<https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case>

## Proof of Concept

1. Create a file in the `test/integration` directory and name is `POC.t.sol`
2. Copy and paste the code below to the new file: `POC.t.sol`.
3. Run `forge test --match-path test/Integration/POC.t.sol -vvv`
4. The test should fail with some `logs` of amounts
5. This fails because `depositAmount` is not equal to `actualAmount`.

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

import { TestHelper } from "../TestHelper.sol";
import { Permit } from "../../src/structs/Permit.sol";
import { IERC20 } from "openzeppelin/token/ERC20/IERC20.sol";
import "forge-std/console.sol";


contract PufferDepositorV2ForkTest is TestHelper {
    
    function test_stETH_deposit_pulls_less_stETH()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), alice, 35000 ether)
        withCaller(alice)
    {

        uint256 depositAmount = 1000000000000000000001;
        Permit memory permit = _signPermit(
            _testTemps(
                "alice",
                address(pufferDepositor),
                depositAmount,
                block.timestamp,
                hex"260e7e1a220ea89b9454cbcdc1fcc44087325df199a3986e560d75db18b2e253"
            )
        );

        uint256 balanceBefore = stETH.balanceOf(alice);
        pufferDepositor.depositStETH(permit, alice);
        uint256 balanceAfter = stETH.balanceOf(alice);
        

        console.log("Alice's Deposit Amount:::", depositAmount);
        uint256 actualTransfer = balanceBefore - balanceAfter;
        console.log("Actual stETH transfered from Alice:::", actualTransfer );
        console.log("Amount of stETh not transfered:::", depositAmount - actualTransfer);
        
        // This will revert since not all depositAmount was transfered from Alice.
        // assertEq(actualTransfer, depositAmount, "despositAmount and actual transfer should be same");
    }
}
```
