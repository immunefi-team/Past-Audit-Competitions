# 29081 - \[SC - Insight] No constructor should be used to set in upgrade...

Submitted on Mar 6th 2024 at 19:13:58 UTC by @SAAJ for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29081

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* wrong usage of OpenZeppelin library in contract

## Description

## Vulnerability Details

As per OZ [docs](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat), values set in constructor in upgradeable is not safe as constructors are only called during the contract deployment, proxies can't access that information as it's stored on the original contract not on the proxy, meaning, proxies are completely oblivious to the existence of constructors.

All the initialization will be lost since it was run in the context of the logic implementation contract and not the Proxy contract.

The exception is given for `immutable` variable but it is also not effective as both the `PufferDepositor` and `PufferVault` are upgradeable, meaning the implementation stores the value passed in constructor and not in proxy storage, refer to OZ’S [documentation](https://docs.openzeppelin.com/upgrades-plugins/1.x/faq#why-cant-i-use-immutable-variables) which clearly states this issue.

Since implementation address can be changed meaning when upgrade will be carried out the values stored through constructor will be lost.

## Impact

The addresses stored in implementation contract of `PufferDepositor` and `PufferVault` through constructor will be lost when upgrade will be processed.

This can cause unexpected behavior like loss of funds if depositing or reverting when withdrawal is made as call made to these contract will be actually made to `address(0)`.

## Code Reference

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferDepositor.sol#L41>

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferVault.sol#57>

## Recommendations

The recommendation is made to avoid using constructor in upgradeable contract and move the immutable variables passed in constructor for both contracts of `PufferDepositor` and `PufferVault` to the `initialize` function.

## Proof of concept

## POC

The POC for the issue is visible in context of not following OZ guideline in terms of not using constructor in upgradeable and using immutable with caution.

```

File: PufferVault.sol

constructor(PufferVault pufferVault, IStETH stETH) payable {
        PUFFER_VAULT = pufferVault;
        _ST_ETH = stETH;
        _disableInitializers();
    }

File: PufferDepositor.sol 

constructor(
        IStETH stETH,
        ILidoWithdrawalQueue lidoWithdrawalQueue,
        IStrategy stETHStrategy,
        IEigenLayer eigenStrategyManager
    ) payable {
        _ST_ETH = stETH;
        _LIDO_WITHDRAWAL_QUEUE = lidoWithdrawalQueue;
        _EIGEN_STETH_STRATEGY = stETHStrategy;
        _EIGEN_STRATEGY_MANAGER = eigenStrategyManager;
        _disableInitializers();
    }



```
