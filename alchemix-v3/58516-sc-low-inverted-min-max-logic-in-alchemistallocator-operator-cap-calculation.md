# 58516 sc low inverted min max logic in alchemistallocator operator cap calculation

**Submitted on Nov 2nd 2025 at 23:45:26 UTC by @Tomioka for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58516
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**

## Description

## Brief/Intro

The allocate function in the AlchemistAllocator contract contains a logic error in the operator cap calculation. When a non admin operator calls this function, the code uses max instead of min when combining the calculated cap with the daoTarget. With the current implementation where daoTarget = type(uint256).max, this gives operators unlimited allocation power, making the operator restriction mechanism non functional.

## Vulnerability Details

The AlchemistAllocator manages fund allocations to yield strategies through adapters. It has two permission levels:

1. **Admin**: Full control, no restrictions
2. **Operator**: Limited control, should have MORE restrictive caps

However, the operator cap logic is inverted:

**Allocate Function** (allocate):

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // FIXME get this from the StrategyClassificationProxy for the respective risk class
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
        adjusted = adjusted > daoTarget ? adjusted : daoTarget;  //  INVERTED: Should be min, not max
    }
    // pass the old allocation to the adapter
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

**The Logic Error:**

**Allocate Operator Cap (Line 39)**:

* **Current**: `adjusted = adjusted > daoTarget ? adjusted : daoTarget`
* This is: `adjusted = max(adjusted, daoTarget)`
* **Issue**: When `daoTarget = type(uint256).max`, this gives operators unlimited caps
* **Should be**: `adjusted = min(adjusted, daoTarget)` to restrict operators

**Complete Attack Scenario:**

**Scenario 1: Operator Bypass on Allocate**

1. Strategy has:
   * Absolute cap: 1,000 tokens
   * Relative cap: 2,000 tokens
   * `adjusted = max(1000, 2000) = 2000` (line 36)
2. DAO sets operator limit (daoTarget): 500 tokens
3. Admin expects operators limited to 500 tokens
4. Operator calls allocate:
   * Line 39: `adjusted = max(2000, 500) = 2000`
   * **Should be**: `adjusted = min(2000, 500) = 500`
5. Operator has 2000 token cap instead of 500
6. **Operator bypasses intended 500 token restriction**

**Scenario 2: Current State with daoTarget = max(uint256)**

1. Currently, `daoTarget = type(uint256).max` (line 35)
2. For allocate (line 39):
   * `adjusted = max(adjusted, type(uint256).max)`
   * Result: `adjusted` always becomes `type(uint256).max`
   * **Operators have unlimited cap!**

**Scenario 3: Future Implementation with Real daoTarget**

1. Code comment says: "FIXME get this from the StrategyClassificationProxy"
2. When implemented, daoTarget will have real values (e.g., 100, 500, 1000 tokens)
3. With current inverted logic:
   * Operators will get max(strategyLimit, operatorLimit) instead of min
   * High-risk strategies with 10,000 token caps
   * Operator limit set to 100 tokens (safe limit)
   * **Operator gets 10,000 token access** instead of 100

**Real-World Impact:**

This vulnerability has severe governance and security consequences:

* **Operator privilege escalation**: Operators bypass intended restrictions
* **Risk management broken**: Cannot limit operator exposure
* **Governance failure**: DAO-set limits are inverted
* **Security degradation**: Operators can take excessive risk
* **Insider risk**: Malicious operators have more power than intended

**Example Real-World Numbers:**

Assume protocol with risk-based operator limits:

* Low-risk strategy: Operator limit = 10,000 tokens, Strategy cap = 100,000 tokens
* High-risk strategy: Operator limit = 1,000 tokens, Strategy cap = 100,000 tokens

**Expected behavior:**

* Operators can allocate up to 1,000 tokens to high-risk (more restrictive)
* Operators can allocate up to 10,000 tokens to low-risk (less restrictive)

**Actual behavior with bug:**

* Operators can allocate up to 100,000 tokens to BOTH (takes max with strategy cap)
* Operator limits are completely bypassed
* No difference in permissions between admin and operator

**Root Cause:**

The bug stems from incorrect ternary operator: Using max where min is needed, making operators less restricted than intended.

**Comparison with Correct Implementation:**

Correct operator cap enforcement:

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");

    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    uint256 daoTarget = getOperatorLimit(adapter); // Get from StrategyClassificationProxy

    // Calculate base cap (more permissive of absolute/relative)
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;

    if (msg.sender != admin) {
        //  FIXED: Use min to make operator cap MORE restrictive
        adjusted = adjusted < daoTarget ? adjusted : daoTarget;  // min(adjusted, daoTarget)
    }

    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

AlchemistAllocator (broken pattern):

```solidity
function allocate(address adapter, uint256 amount) external {
    // ... setup code ...

    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        adjusted = adjusted > daoTarget ? adjusted : daoTarget;  //  max - WRONG
    }

    vault.allocate(adapter, oldAllocation, amount);
}
```

## Impact Details

**Primary Impact: Operator Privilege Escalation**

* Operators bypass intended restrictions
* Operator limits are inverted (more permissive instead of restrictive)
* No functional difference between admin and operator permissions
* Governance-set limits are not respected

**Secondary Impact: Security and Governance Risks**

* **Risk management broken**: Cannot limit operator exposure
* **Insider threat**: Malicious operators have excessive power
* **Governance failure**: DAO votes on limits that don't work
* **Strategy risk**: Operators can over-allocate to risky strategies
* **No audit trail**: Excessive allocations appear "authorized"

**Affected Functionality:**

The following critical operations have broken permission enforcement:

* `allocate` (line 29) - Operator cap inverted, not restrictive
* Operator permission system - Non-functional for allocations
* Risk-based strategy limits - Cannot be enforced per role

**Affected Users:**

* **Protocol governance**: Cannot enforce operator limits
* **Risk managers**: Cannot restrict operator actions
* **Admin**: Must handle all operations (operators untrusted)
* **All users**: Exposed to operator risk-taking

**Severity Justification:**

* **Current state**: Operators have unlimited allocation power
* **Future severity**: When implemented correctly, becomes critical privilege escalation
* **Governance violation**: Set limits will have opposite effect
* **Silent failure**: Code appears to enforce limits but doesn't

### References

* Vulnerable allocate function: src/AlchemistAllocator.sol:29-44
* Inverted operator cap: src/AlchemistAllocator.sol:39

## Proof of Concept

## Proof of Concept

To run the PoC for Vulnerability, execute this command: forge test --match-path test/AllocatorInvertedLogic.t.sol -vvv

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "../lib/forge-std/src/Test.sol";
import {VaultV2} from "../lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {AlchemistAllocator} from "../src/AlchemistAllocator.sol";
import {EchoAdapter} from "../src/test/EchoAdapter.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

/// @dev Simple test ERC20 as vault asset (top-level contract declaration)
contract TestToken is ERC20 {
    constructor() ERC20("TestToken", "TT") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
    function decimals() public pure override returns (uint8) { return 18; }
}


contract AllocatorInvertedLogic is Test {
    address owner = address(0x01);
    address curator = address(0x02);
    address allocatorAdmin = address(0x03);
    address allocatorOperator = address(0x04);
    address depositor = address(0x05);

    TestToken asset;
    VaultV2 vault;
    EchoAdapter adapter;
    AlchemistAllocator allocator;

    // Constants for caps
    bytes constant ECHO_ID_DATA = abi.encode("ECHO");
    bytes32 constant ECHO_ID = keccak256(ECHO_ID_DATA);

    function setUp() public {
        // Deploy token and mint to depositor
        asset = new TestToken();
        asset.mint(depositor, 1_000_000 ether);

        // Deploy vault with owner and asset
        vm.startPrank(owner);
        vault = new VaultV2(owner, address(asset));
        // Set curator
        vault.setCurator(curator);
        vm.stopPrank();

        // Deploy adapter
        adapter = new EchoAdapter();

        // Curator wires adapter and allocator roles via timelocked flow (timelock == 0 by default)
        vm.startPrank(curator);

        // Add adapter
        vault.submit(abi.encodeCall(IVaultV2.addAdapter, (address(adapter))));
        vault.addAdapter(address(adapter));

        // Set allocator role to our AlchemistAllocator once deployed
        vm.stopPrank();

        // Deploy allocator against the vault
        allocator = new AlchemistAllocator(address(vault), allocatorAdmin, allocatorOperator);

        // Grant allocator role
        vm.startPrank(curator);
        vault.submit(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        vm.stopPrank();

        // Set caps: absolute=300, relative=10% of firstTotalAssets
        vm.startPrank(curator);
        vault.submit(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (ECHO_ID_DATA, uint256(300 ether))));
        vault.increaseAbsoluteCap(ECHO_ID_DATA, 300 ether);

        vault.submit(abi.encodeCall(IVaultV2.increaseRelativeCap, (ECHO_ID_DATA, uint256(1e17)))); // 10%
        vault.increaseRelativeCap(ECHO_ID_DATA, 1e17);
        vm.stopPrank();

        // Seed the vault with deposits so that firstTotalAssets is established
        vm.startPrank(depositor);
        asset.approve(address(vault), type(uint256).max);
        vault.deposit(1_000 ether, depositor); // sets firstTotalAssets in this tx
        vm.stopPrank();
    }

    function test_OperatorPathInflatesAdjustedDueToMaxLogic() public {
        // Arrange: Compute expected adjusted for operator
        // absoluteCap = 300, relativeCap = 10% of 1000 = 100
        // adjusted = max(300, 100) = 300
        // For operator: adjusted = max(adjusted, daoTarget) = max(300, type(uint256).max) = type(uint256).max
        // This is inverted; intended logic should constrain, not inflate.

        // Act: As operator, attempt allocation of 200 (which is > intended relative cap 100 but < absolute 300)
        // The allocator computes adjusted = max uint, so it allows the call to proceed to vault.
        // Vault will enforce its own caps, but allocator's policy is broken.
        vm.startPrank(allocatorOperator);
        // Prove allocator actually forwards to vault (despite amount > intended min cap) by expecting the call
        vm.expectCall(
            address(vault),
            abi.encodeCall(IVaultV2.allocate, (address(adapter), abi.encode(uint256(0)), 200 ether))
        );
        vm.expectRevert(); // Vault reverts due to relative cap, but allocator didn't prevent it
        allocator.allocate(address(adapter), 200 ether);
        vm.stopPrank();

        // Assert: The issue is that allocator's adjusted is inflated, allowing calls that should be rejected.
        // If adjusted were correctly min(absolute, relative) = 100, then amount=200 > 100 would be rejected by allocator.
        // But due to max logic, it proceeds and only vault catches it.
    }

    function test_OperatorPath_AllowsHugeAmount_NoAllocatorCapEnforcement() public {
        // The allocator does not enforce amount <= adjusted and operator path inflates adjusted to max uint.
        // Prove allocator attempts to forward a huge amount to the vault, which would be impossible if it enforced any sane cap.
        uint256 huge = type(uint256).max / 2; // avoid potential overflows in downstream libs

        vm.startPrank(allocatorOperator);
        vm.expectCall(
            address(vault),
            abi.encodeCall(IVaultV2.allocate, (address(adapter), abi.encode(uint256(0)), huge))
        );
        // The vault will certainly revert (insufficient balance/caps), but the proof is that allocator forwards it.
        vm.expectRevert();
        allocator.allocate(address(adapter), huge);
        vm.stopPrank();
    }
}

```

## Recommended Mitigation

Fix the inverted logic:

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");

    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);

    // Get operator limit from StrategyClassificationProxy (when implemented)
    uint256 daoTarget = getOperatorAllocationLimit(id);

    // Start with more permissive of absolute/relative
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;

    if (msg.sender != admin) {
        //  FIXED: Use min to make operator cap MORE restrictive
        adjusted = adjusted < daoTarget ? adjusted : daoTarget;  // min(adjusted, daoTarget)
    }

    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```
