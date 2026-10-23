# 28650 - \[SC - Insight] Protocol Insolvency due to the over inflated ca...

Submitted on Feb 23rd 2024 at 01:09:56 UTC by @marqymarq10 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28650

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The protocol is at a high risk of becoming insolvent due to an error in accounting. The protocol accepts the deposit of the following three underlying assets in exchange for `puffETH`:

* `Ether`
* `stETH`
* `wstETH`

However, `puffETH` can only be redeemed for `WETH` (according to `PufferVaultMainnet.sol`). The internal accounting does not take into consideration that only one asset can be redeemable for `puffETH`. This leads to an over inflated perception of working capital in the protocol.

Although this issue is not fully exploitable at the moment, this bug is still within the scope of the bug bounty program because it affects assets in scope, fits the criteria of impact in scope, and the bug bounty is marked as [Primacy of Impact](https://immunefisupport.zendesk.com/hc/en-us/articles/12340245635089-Best-Practices-Primacy-of-Impact). Additionally, this bug is directly related to concerns in the bug bounty's description:

```
Which parts of the code are you most concerned about?

- The vault logic of Open Zeppelin’s that we have overridden
  - We are not fully compliant with ERC 4626, because we have overridden some functionality. Namely, we have overridden the maxWithdraw() function, and we are not returning a value of 0, even though withdrawals are currently paused. The specs of ERC 4626 require global and user-specific limits to be factored into the result of this function, which we do not abide by. Similarly, maxRedeem() has the same non-compliance with ERC 4626 specs.
...

What attack vectors are you most concerned about?

- We’ve upgraded the logic dealing with the vault’s total assets, so perhaps the logic to calculate the shares each user has within the vault can be examined closely to ensure the vault still works as intended and there are no attacks to mess with the original accounting code of the vault
```

`maxWithdraw()` is in a contract that is not listed in scope as it is only overridden in `PufferVaultMainnet.sol`, but it is asked to be examined. This is another reason as to why this bug remains in scope.

## Vulnerability Details

As previously mentioned, the protocol accepts three underlying assets in exchange for `puffETH`, but `puffETH` can only be redeemed for `WETH`. `ERC-4626` relies on the balance of the underlying asset to calculate the value of shares. This is evident in Open Zeppelin's implementation of `ERC-4626`:

```
function totalAssets() public view virtual returns (uint256) {
    return _asset.balanceOf(address(this));
}
```

Since Puffer Finance accepts three underlying assets, `PufferVault::totalAssets()` has been overridden:

```
function totalAssets() public view virtual override returns (uint256) {
    return _ST_ETH.balanceOf(address(this)) + getELBackingEthAmount() + getPendingLidoETHAmount()
        + address(this).balance;
}
```

Additionally `totalAssets()` is overridden in `PufferVaultMainnet.sol`:

```
function totalAssets() public view virtual override returns (uint256) {
    uint256 callValue;
    // solhint-disable-next-line no-inline-assembly
    assembly {
        callValue := callvalue()
    }
    return _ST_ETH.balanceOf(address(this)) + getPendingLidoETHAmount() + getELBackingEthAmount()
        + _WETH.balanceOf(address(this)) + (address(this).balance - callValue) + PUFFER_ORACLE.getLockedEthAmount();
}
```

Regardless of which version of `totalAssets()` is used the accounting error still remains. `totalAssets()` is relevant due to its involvement of the calculation of `maxWithdraw()`:

```
function maxWithdraw(address owner) public view virtual override returns (uint256 maxAssets) {
    uint256 remainingAssets = getRemainingAssetsDailyWithdrawalLimit();
    uint256 maxUserAssets = previewRedeem(balanceOf(owner));
    return remainingAssets < maxUserAssets ? remainingAssets : maxUserAssets;
}
```

`maxWithdraw()` calls `previewRedeem()`, which in turn calls `_convertToAssets()`:

```
function _convertToAssets(uint256 shares, Math.Rounding rounding) internal view virtual returns (uint256) {
    return shares.mulDiv(totalAssets() + 1, totalSupply() + 10 ** _decimalsOffset(), rounding);
}
```

As is shown above, `maxUserAssets` uses the inflated `totalAssets()` to calculate if a user can withdraw the underlying asset. This calculation is inherently overinflated, as the vault is accounting assets that are not truly redeemable.

In `PufferVaultMainnet::withdraw()`, the asset that is meant to be sent to the user is `WETH`:

```
function withdraw(uint256 assets, address receiver, address owner)
    public
    virtual
    override
    restricted
    returns (uint256)
{
    uint256 maxAssets = maxWithdraw(owner);
    if (assets > maxAssets) {
        revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
    }

    _updateDailyWithdrawals(assets);

    _wrapETH(assets);

    uint256 shares = previewWithdraw(assets);
    // solhint-disable-next-line func-named-parameters
    _withdraw(_msgSender(), receiver, owner, assets, shares);

    return shares;
}
```

This is further validated by a comment in `PufferVault.sol` above `totalAssets()`:

```
/**
    * @dev See {IERC4626-totalAssets}.
    * Eventually, stETH will not be part of this vault anymore, and the Vault(pufETH) will represent shares of total ETH holdings
    * Because stETH is a rebasing token, its ratio with ETH is 1:1
    * Because of that our ETH holdings backing the system are:
    * stETH balance of this vault + stETH balance locked in EigenLayer + stETH balance that is the process of withdrawal from Lido
    * + ETH balance of this vault
*/
```

In reality, the working capital is only the assets that are held by the contract. Including accounts receivable in the calculation for working capital is misleading to users of the protocol. The over inflated value for the underlying asset will cause insolvency issues as the protocol will not hold enough of the underlying asset to fullfill their debt obligations.

## Impact Details

If this accounting error is left unresolved, the users of the protocol are at risk of having their deposits stuck in the protocol indefinitely. The bug makes `puffETH` an undercollateralized asset. Historically the discovery of an undercollateralized asset can lead to a bank run and a large volume of sell pressure on the open market. This would irreparably harm the reputation of Puffer Finance and along with the price of `puffETH`.

## References

* [totalAssets (PufferVault)](https://github.com/PufferFinance/pufETH/blob/0a345743ec4886735b046164876be32c35e59ebe/src/PufferVault.sol#L142-L153)
* [totalAssets (PufferVaultMainnet)](https://github.com/PufferFinance/pufETH/blob/0a345743ec4886735b046164876be32c35e59ebe/src/PufferVaultMainnet.sol#L64-L85)
* [maxWithdraw (PufferVaultMainnet)](https://github.com/PufferFinance/pufETH/blob/0a345743ec4886735b046164876be32c35e59ebe/src/PufferVaultMainnet.sol#L87-L97)
* [previewRedeem (ERC4626Upgradeable)](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/789ba4f167cc94088e305d78e4ae6f3c1ec2e6f1/contracts/token/ERC20/extensions/ERC4626Upgradeable.sol#L188-L191)
* [\_convertToShares (ERC4626Upgradeable)](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/789ba4f167cc94088e305d78e4ae6f3c1ec2e6f1/contracts/token/ERC20/extensions/ERC4626Upgradeable.sol#L252-L257)
* [withdraw (PufferVaultMainnet)](https://github.com/PufferFinance/pufETH/blob/0a345743ec4886735b046164876be32c35e59ebe/src/PufferVaultMainnet.sol#L111-L137)

## Proof of concept

## PoC

Add the following code snippet to `test/integration/PufferDepositorMainnet.fork.t.sol`:

```
function test_prove_insolvency()
    public
    giveToken(BLAST_DEPOSIT, address(stETH), alice, 200 ether)
    withCaller(alice)
{
    stETH.approve(address(pufferVault), 100 ether);
    uint256 directDepositAmount = pufferVault.depositStETH(100 ether, alice);

    // Notice we are attempting to withdraw one wei not one ether
    pufferVault.withdraw(1, address(alice), address(alice));
    // The following line will also revert 
    // pufferVault.redeem(1, address(alice), address(alice));
}
```

If you run the above test case, you will see the test case fails due to `[FAIL. Reason: EvmError: Revert]`. If you look at the stack traces, it will show the reason for the revert as: `EvmError: OutOfFund`.

Additionally, comment out `PufferVaultMainnet::totalAssets()` to verify the issue still remains with `PufferVault::totalAssets()`.
