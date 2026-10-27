# 29116 - \[SC - Low] Using deposit results in more shares for the sa...

Submitted on Mar 7th 2024 at 13:58:54 UTC by @LokiThe5th for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29116

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `PufferVault` is an ERC4626 vault which mints shares to a user. However, the underlying asset is `stETH`, which always leaves some amount of shares over when using the normal `transferFrom` function. By using the standard `ERC4626::deposit()` and `ERC4626:mint()` functionality the user is always minted slightly more shares than expected.

## Vulnerability Details

`stETH` has a known property that when using the `transferFrom` or `transfer` functions it always leaves some small amount of `stETH` behind.

In the `PufferVault` the `deposit` and `mint` functions which are inherited from `ERC4626` both use the `transferFrom` method via `safeTransferFrom` within the `ERC4626::_deposit` function.

This has the effect that when a user deposits via the `PufferVault` there is always slightly less `stETH` transferred than expected, but the user is still minted shares according the initial amount.

This also allows a user to mint free shares for dust amounts. As the stETH amount left over rom `transferFrom` increases, so too does the amount of free shares the user receives. This extra amount can become larger as stETH/share rate grows. See there docs here: <https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case>.

## Impact Details

A user can mint 1 share in exchange for the rounding amount (1-2 wei). This may become larger in the future.

In this case a user should always use `deposit` instead of mint, as the rounding differences means they will always receive slightly more shares when using `deposit`.

## References

Lido: <https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case>

## Proof of Concept

The below files should be placed within the `puffEth` repo.

<https://drive.google.com/drive/folders/1-h\\_HIsiwwqv2L80TIOZd9x-HwvD0Yuwu?usp=sharing>

Place the `OverSharePoC.sol` file in the `test` directory.

The `OverSharePoc.sol` file mimics `stETH`'s property of leaving dust amounts after normal `transferFrom` calls.

Place the `MockAccessManager.sol`, `stETHMockDust.sol`, `LidoWithdrawalQueueMockExpanded.sol` and the `stETHStrategyMock.sol`files in the `test/mocks` directory.

Run the test with `forge test --match-contract OverSharePoC -vvv`.

The console output:

```
[PASS] test_PoC_deposit() (gas: 281393)
Logs:
  Shares received:  1000000000000000001
  steth used:  999999999999999999

[PASS] test_PoC_mint() (gas: 281591)
Logs:
  Shares received:  1000000000000000000
  steth used:  999999999999999999
```

This shows Alice receiving slightly more shares when using `deposit` vs `mint`.
