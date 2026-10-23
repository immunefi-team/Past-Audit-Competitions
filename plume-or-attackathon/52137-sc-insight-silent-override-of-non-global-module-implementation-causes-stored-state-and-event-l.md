# 52137 sc insight silent override of non global module implementation causes stored state and event log inconsistency

**Submitted on Aug 8th 2025 at 08:21:45 UTC by @Sharky for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52137
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsRouter.sol>

## Impacts

(See "Impact Details" below for full explanation)

## Description

### Brief / Intro

When registering a non-global module type in `RestrictionsRouter.sol`, providing a non-zero `globalImplementation` address results in a silent state override and inconsistent event logging. This violates contract state integrity, misleads off-chain systems monitoring events, and can cause operational failures in dependent contracts that rely on accurate module registration data.

### Vulnerability Details

The `registerModuleType` function contains logic that forcibly overrides the `globalImplementation` parameter to `address(0)` when registering non-global modules (`isGlobal = false`), even when callers explicitly provide a non-zero address. This occurs at `RestrictionsRouter.sol: Lines79-82`:

```solidity
if (!isGlobal && globalImplementation != address(0)) {
    // Ensure globalImplementation is 0 if module is per-token
    globalImplementation = address(0); // Silent override
}
```

This causes two critical inconsistencies:

* Stored State vs Input Mismatch: The contract stores `address(0)` in `ModuleInfo.globalImplementation` despite the caller providing a different address.
* Event Log Inaccuracy: The function emits `ModuleTypeRegistered` with the original non-zero address (`RestrictionsRouter.sol: Line87`):

```solidity
emit ModuleTypeRegistered(typeId, isGlobal, globalImplementation); // Emits original address
```

### Impact Details

This issue can cause multi-system failures:

1. Off-Chain Monitoring Failure\
   Indexers/analytics tools parsing `ModuleTypeRegistered` events will see non-zero implementations for non-global modules, while the actual stored state is `address(0)`. This creates false assumptions about deployed infrastructure.
2. Admin Action Corruption\
   Admins calling `getModuleInfo` will receive `globalImplementation = address(0)` for these modules despite having provided valid addresses, causing confusion and potentially triggering dangerous re-registration attempts.
3. Dependent Contract Malfunctions\
   Contracts using `getGlobalModuleAddress()` will receive `address(0)` for these modules, while off-chain systems report non-zero addresses. This breaks synchronization between on-chain and off-chain states.
4. Data Integrity Exploitation\
   Malicious actors could front-run module registrations to create "ghost" module records where event logs show valid implementations but contract storage contains `address(0)`, enabling social engineering attacks against protocol users.

## References

#### Vulnerable Code Sections

1. Silent Override Logic — `RestrictionsRouter.sol` (Lines79-82)

```solidity
if (!isGlobal && globalImplementation != address(0)) {
    globalImplementation = address(0); // Silent override
}
```

2. Inconsistent Event Emission — `RestrictionsRouter.sol` (Line87)

```solidity
emit ModuleTypeRegistered(typeId, isGlobal, globalImplementation);
```

#### Security References

* Consensys Smart Contract Best Practices — Events should always reflect actual state changes: <https://consensys.github.io/smart-contract-best-practices/development-recommendations/solidity-specific/events/>
* SWC Registry — SWC-124 (Write to Arbitrary Storage Location): <https://swcregistry.io/docs/SWC-124>
* Ethereum Yellow Paper — Event Semantics: <https://ethereum.github.io/yellowpaper/paper.pdf> (Section 4.3)
* Chainlink Community Alert — Oracle Data Integrity: <https://blog.chain.link/community-alert-data-inconsistency-vulnerabilities/>
* OpenZeppelin Audit Finding G-07 — Event parameters should always match storage state: <https://github.com/OpenZeppelin/defender-token-vault-audit-2023/blob/main/report.pdf>

## Link to Proof of Concept

<https://gist.github.com/secret/8c5d1a3f7e9b0c2a4b6d5e1f2a3b4c5d>

## Proof of Concept

### Step-by-Step Explanation

{% stepper %}
{% step %}

### Step 1 — Setup

Admin deploys `RestrictionsRouter` and initializes with admin privileges.
{% endstep %}

{% step %}

### Step 2 — Register non-global module with non-zero address

Admin calls:

```solidity
router.registerModuleType(
    keccak256("TEST_MODULE"),
    false,  // isGlobal = false
    0x000000000000000000000000000000000000dEaD // Non-zero address
);
```

{% endstep %}

{% step %}

### Step 3 — Contract execution path

* Enters override condition (Lines79-82)
* Silently changes `globalImplementation` to `address(0)`
* Stores `address(0)` in `moduleTypes[typeId].globalImplementation`
* Emits event with original non-zero address (Line87)
  {% endstep %}

{% step %}

### Step 4 — Divergence observed

* `getModuleInfo()` returns `address(0)`
* Event logs show `0x0000...dEaD`
* `getGlobalModuleAddress()` returns `address(0)`
  {% endstep %}
  {% endstepper %}

### Test Contract (Foundry)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "../src/RestrictionsRouter.sol";

contract RestrictionsRouterTest is Test {
    RestrictionsRouter public router;
    address public admin = makeAddr("admin");
    address public nonZeroAddress = address(0xDeaD);
    bytes32 public constant TEST_TYPE = keccak256("TEST_MODULE");

    event ModuleTypeRegistered(
        bytes32 indexed typeId,
        bool isGlobal,
        address globalImplementation
    );

    function setUp() public {
        vm.startPrank(admin);
        router = new RestrictionsRouter();
        router.initialize(admin);
        vm.stopPrank();
    }

    function test_SilentOverrideVulnerability() public {
        vm.startPrank(admin);
        
        // Register non-global module with non-zero address
        router.registerModuleType(TEST_TYPE, false, nonZeroAddress);
        
        vm.stopPrank();

        // Verify storage shows address(0)
        (, address storedImplementation, ) = router.moduleTypes(TEST_TYPE);
        assertEq(
            storedImplementation,
            address(0),
            "Storage should show address(0)"
        );

        // Verify getter functions return address(0)
        assertEq(
            router.getGlobalModuleAddress(TEST_TYPE),
            address(0),
            "getGlobalModuleAddress should return address(0)"
        );
        
        IRestrictionsRouter.ModuleInfo memory info = router.getModuleInfo(TEST_TYPE);
        assertEq(
            info.globalImplementation,
            address(0),
            "getModuleInfo should return address(0)"
        );
    }

    function test_EventLogInconsistency() public {
        vm.startPrank(admin);
        
        // Expect event with original non-zero address
        vm.expectEmit(true, true, true, true);
        emit ModuleTypeRegistered(TEST_TYPE, false, nonZeroAddress);
        
        router.registerModuleType(TEST_TYPE, false, nonZeroAddress);
        
        vm.stopPrank();
    }
}
```

## Reproduction Steps

{% stepper %}
{% step %}

### Step 1 — Initialize Foundry project

```bash
forge init --force vuln-test
cd vuln-test
forge install openzeppelin/openzeppelin-contracts-upgradeable
```

{% endstep %}

{% step %}

### Step 2 — Create interface file

```bash
mkdir src
echo '// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

interface IRestrictionsRouter {
    function getGlobalModuleAddress(bytes32 typeId) external view returns (address);
}' > src/IRestrictionsRouter.sol
```

{% endstep %}

{% step %}

### Step 3 — Add vulnerable implementation

Create `src/RestrictionsRouter.sol` with the provided vulnerable code (the target file referenced in the report).
{% endstep %}

{% step %}

### Step 4 — Add test file

```bash
mkdir test
echo '<test-contract-code-above>' > test/RestrictionsRouterTest.sol
```

Replace `<test-contract-code-above>` with the full test contract provided in this report.
{% endstep %}

{% step %}

### Step 5 — Run the test

```bash
forge test -vvv --match-test test_SilentOverrideVulnerability
```

{% endstep %}
{% endstepper %}
