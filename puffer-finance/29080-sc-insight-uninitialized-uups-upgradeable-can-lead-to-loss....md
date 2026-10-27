# 29080 - \[SC - Insight] Uninitialized uups upgradeable can lead to loss...

Submitted on Mar 6th 2024 at 19:08:33 UTC by @SAAJ for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29080

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

`PufferDepositor` and `PufferVault` contracts inherit UUPSUpgradeable contract by OZ. These contracts are deployed using a proxy pattern whereby the implementation contract is used by the proxy contract for all its logic. This helps to facilitate future upgrades by pointing the proxy contract to a new and upgraded implementation contract.

However, when the implementation contract is left uninitialized, it is possible for any attacker to gain ownership of the `restricted` role in the implementation contract for both, refer to this [article](https://jordaniza.com/posts/upgradeable-contracts/). This is clearly present due to uninitialized `UUPSUpgradeable` contract in `PufferDepositor` and `PufferVault`.

OpenZeppelin’s clearly provide [guide](https://docs.openzeppelin.com/contracts/5.x/upgradeable#usage) in context of initializing OZ upgradeable contracts in the initialize function.

## Impact

The problem is that that implementation vault and depositor contracts are not initialized, which means that anybody can initialize the contract to become the owner.

The POC clearly shows how the `attack_Contract` contract inherits both `PufferDepositor` and `UUPSUpgradeable`. The constructor initializes both contracts.

The malicious contract contain logic for both `selfdestruct` and upgrade to a new `implementation`.

A selfDestruct function is included for demonstration purposes. This function allows the contract to be destroyed and its funds sent to a specified address.

The `malicious_AuthorizeUpgrade` function will be used to call `_authorizeUpgrade` with a new implementation address afterwards the `selfDestruct` is called which will destroy the malicious `attack_Contract` with leaving no way to trace back the new implementation contract.

Once the attacker has ownership they are able to perform an upgrade of the implementation contract's logic contract and delegate call into any arbitrary contract, allowing them to transfer all asset to themselves leading to direct loss of funds.

The attacker also can then destroy the contract by doing a delegate call (via the execute function) to a function with the `self-destruct` opcode. Once the implementation is destroyed vault will be unusable.

since there's no logic in the proxies to update the implementation - that means this is permanent (i.e. there's no way to call any function on vault anymore, it will be simply dead).

## Code Reference

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferDepositor.sol#L47>

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferVault.sol#294>

## Recommendations

The recommendation is made for initializing the `UUPSUpgradeable` contract in implementation contract of `PufferDepositor` and `PufferVault`.

```
// only needed for UUPS
 __UUPSUpgradeable_init();
```

## Proof of concept

## POC

Here is a POC clearly evident absence of initialization of `UUPSUpgradeable` contract for both `PufferDepositor` and `PufferVault` that can be used by an attacker to upgrade and then destroy the implementation contract.

```

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;


import "openzeppelin-contracts/contracts/proxy/utils/UUPSUpgradeable.sol";
import "../src/PufferDepositor.sol";

contract attack_Contract is PufferDepositor, UUPSUpgradeable {
    constructor() PufferDepositor() UUPSUpgradeable()

    function initialize(address accessManager) public initializer {
        __AccessManaged_init(accessManager);
        __UUPSUpgradeable_init();
    }

    function selfDestruct() public restricted {
        selfdestruct(payable(attacker));
    }

    function test_maliciousAuthorizeUpgrade() public restricted {

        address newImplementation = address(malicious_Contract);
        _authorizeUpgrade(newImplementation);
    }
}

```
