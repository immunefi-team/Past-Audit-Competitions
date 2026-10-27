# 28663 - \[SC - Low] Deposit of stETH fails due to LIDOs - wei corno...

Submitted on Feb 23rd 2024 at 10:22:59 UTC by @codesentry for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28663

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`depositStETH` method of `PufferDepositor` contract transfer stETH from `msg.sender` to `PufferDepositor` and then `PufferVault` transfer it from `PufferDepositor`. Overall `depositStETH` may fails randomly because of random 1 wei cornor issue in LIDO's stETH.

## Vulnerability Details

stETH balance calculation includes integer division, and there is a common case when the whole stETH balance can't be transferred from the account while leaving the last 1-2 wei on the sender's account. The same thing can actually happen at any transfer or deposit transaction. This issue is documented here(<https://github.com/lidofinance/lido-dao/issues/442>) and still an valid issue. Same is documented in LIDO's official document(<https://docs.lido.fi/guides/lido-tokens-integration-guide/>) also.

Below is the code snippet that has bug.

```
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

        SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);

        return PUFFER_VAULT.deposit(permitData.amount, msg.sender);
    }
```

Assume user is depositing 2stETH. `safeTransferFrom` transfers 2 stETH but `PufferDepositor` contract get 2stETH minus 1 wei.

`PUFFER_VAULT.deposit(2e18, msg.sender)` would fail because contract don't have enough stETH due to 1 wei corner issue .

This issues is an rounding issue and can happens randomly.

## Impact Details

depositStETH() fails . Contract fails to perform intended functionality.

## References

<https://github.com/lidofinance/lido-dao/issues/442> <https://docs.lido.fi/guides/lido-tokens-integration-guide/>

## Proof of concept

##

Bug in this line of code

```
   SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);

        return PUFFER_VAULT.deposit(permitData.amount, msg.sender);
```

As documented in LIDO, this is rounding issue . This may happen randomly. POC would be successful only when rounding issue occurs. I believe this is straight forward issue and hence POC may not require . Still
