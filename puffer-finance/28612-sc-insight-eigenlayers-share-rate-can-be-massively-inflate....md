# 28612 - \[SC - Insight] EigenLayers share rate can be massively inflate...

Submitted on Feb 22nd 2024 at 14:10:16 UTC by @Shaheen for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28612

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Inadequate Integration

## Description

## Brief/Intro

Griefing attacks can significantly inflate EigenLayer's share rate, potentially leading to a reduction in shares received by the PufferVault.

## Vulnerability Details

Puffer is integrating EigenLayer. The PufferVault contract's `depositToEigenLayer()` function is utilized to deposit users' assets into the Eigenlayer Strategy. However, it lacks a mechanism to ensure that the expected shares are returned, which can result in loss of shares for the users.

```solidity
    function depositToEigenLayer(uint256 amount) external virtual restricted {
        SafeERC20.safeIncreaseAllowance(_ST_ETH, address(_EIGEN_STRATEGY_MANAGER), amount);
        ///@audit-issue slippage protection should be added because "share rate can be massively inflated" ~ EigenLayer
        _EIGEN_STRATEGY_MANAGER.depositIntoStrategy({ strategy: _EIGEN_STETH_STRATEGY, token: _ST_ETH, amount: amount });
    }
```

The EigenLayer's `depositIntoStrategy()`/`deposit()` function is susceptible to griefing attacks, as highlighted in their C4 audit (1). While EigenLayer acknowledges this issue (2), their mitigation approach is to verify that the returned shares are not zero, considering it as a worst-case scenario when the share rate is "massively inflated"

```solidity
->    // extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
      require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");
```

EigenLayer does not implement slippage protection within this function for not so "massive" inflation attacks, leaving it to the discretion of strategy designers. Strategy designers are advised to incorporate slippage protection into their implementations, as discussed in their audit findings (3):

> This is good information for Strategy designers - EigenLayer Dev

To clarify, EigenLayer addresses scenarios where a griefing attack is significant enough to reduce shares to zero. However, it does not handle situations where the attack results in a much smaller number of shares, such as 25-50, when the expected output was 100 shares.

## Impact Details

* Loss of Shares

## Recommended Mitigation

EigenLayer depends on this shares numbers so much (delegation & withdrawals specifically), so it is very important that users/protocol receive maximum/expected shares.

To enhance security for the users'/protocol's funds, it is recommended to add a slippage protection mechanism using `minShares`:

```solidity
    function depositToEigenLayer(uint256 amount, uint256 minShares) external virtual restricted {
        SafeERC20.safeIncreaseAllowance(_ST_ETH, address(_EIGEN_STRATEGY_MANAGER), amount);
        uint256 shares = _EIGEN_STRATEGY_MANAGER.depositIntoStrategy({ strategy: _EIGEN_STETH_STRATEGY, token: _ST_ETH, amount: amount });
        require(shares >= minShares, "Slippage Protected");
    }
```

### References

<https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/343> <https://github.com/Layr-Labs/eigenlayer-contracts/blob/d80482086a8391be7a7bbf3fb1287a68b338cc73/src/contracts/strategies/StrategyBase.sol#L118> <https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/343#issuecomment-1545067086>

## Proof of Concept

Regrettably, due to the nature of this issue being specific to griefing attacks on EigenLayer, I find it challenging to construct a Coded Proof of Concept (PoC). I hope that the protocol team and triagers will comprehend this limitation. Thank you for your understanding.
