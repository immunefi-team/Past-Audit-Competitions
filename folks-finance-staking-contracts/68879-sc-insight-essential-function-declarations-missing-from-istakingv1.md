# 68879 sc insight essential function declarations missing from istakingv1&#x20;

**Submitted on Mar 11th 2026 at 19:30:41 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68879
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`IStakingV1` is the interface third-party integrators and users use to interact with `Staking`. It is missing `paused()` — an essential read for any caller before invoking `stake()`, which reverts when the contract is paused — and optionally `activeTotalStaked()` and `activeTotalRewards()`, which track total locked funds. Without these declarations, callers holding an `IStakingV1` reference must bypass the interface, and `supportsInterface` returns `false` for any integrator who adds them to their local copy of the interface — breaking ERC-165 introspection.

### Vulnerability Details

`Staking` inherits `Pausable` and applies `whenNotPaused` to `stake()`:

```solidity
// src/Staking.sol#L48-L55
function stake(uint8 periodIndex, uint256 amount, StakeParams calldata params)
    external
    nonReentrant
    whenNotPaused   // reverts if paused
    returns (uint8)
```

`paused()` is the standard guard a caller checks before submitting a stake. It is not declared in `IStakingV1`, so an integrator holding an `IStakingV1` reference cannot call it:

```solidity
IStakingV1 staking = IStakingV1(stakingAddress);
if (!staking.paused()) {        // compile error: member not found in IStakingV1
    staking.stake(...);
}
```

Similarly, `Staking.sol#L33–L34` declares two `public` state variables whose compiler-generated getters are absent from the interface:

```solidity
// src/Staking.sol#L33-L34
uint256 public activeTotalStaked;
uint256 public activeTotalRewards;
```

The interface currently exposes only user-facing actions and per-period/per-stake queries:

```solidity
// src/IStakingV1.sol — view functions present
function TOKEN() external view returns (IERC20);
function getStakingPeriods() external view returns (StakingPeriod[] memory);
function getStakingPeriod(uint8 periodIndex) external view returns (StakingPeriod memory);
function getUserStakes(address user) external view returns (UserStake[] memory);
function getUserStake(address user, uint8 stakeIndex) external view returns (UserStake memory);

// missing — essential
// function paused() external view returns (bool);

// missing — optional
// function activeTotalStaked() external view returns (uint256);
// function activeTotalRewards() external view returns (uint256);
```

### Impact Details

**`paused()` (essential):** Any integration that needs to gate `stake()` on the paused state must either downcast to the concrete `Staking` type or use a raw `staticcall` — both of which couple the caller to the implementation and defeat the purpose of the interface.

**`activeTotalStaked` / `activeTotalRewards` (optional):** Dashboards, monitoring bots, and manager tooling that hold an `IStakingV1` reference cannot verify the fund-health invariant `TOKEN.balanceOf(address(this)) >= activeTotalStaked + activeTotalRewards` without bypassing the interface.

**On-chain `supportsInterface`:** `type(IStakingV1).interfaceId` is computed as the XOR of every function selector declared in `IStakingV1`. The three missing selectors are excluded from that XOR. If an integrator builds a complete version of `IStakingV1` that includes them, they compute a different `interfaceId`, and `Staking.supportsInterface` returns `false` for it — even though the contract implements every function — breaking ERC-165 introspection.

```solidity
// Integrator's complete interface (adds the missing declarations)
interface IStakingV1Complete is IStakingV1 {
    function paused() external view returns (bool);
    function activeTotalStaked() external view returns (uint256);
    function activeTotalRewards() external view returns (uint256);
}

// type(IStakingV1Complete).interfaceId != type(IStakingV1).interfaceId
staking.supportsInterface(type(IStakingV1Complete).interfaceId); // false
// even though Staking implements every function in IStakingV1Complete
```

**Impact category:** Low

### References

* Missing declarations: `src/IStakingV1.sol`
* `paused()`: inherited from `Pausable` — `src/Staking.sol#L20`
* `activeTotalStaked`, `activeTotalRewards`: `src/Staking.sol#L33–L34`
* `supportsInterface`: `src/Staking.sol#L230–L239`

## Recommendation

Add the missing declarations to `IStakingV1`:

```solidity
// essential
function paused() external view returns (bool);

// optional but recommended
function activeTotalStaked() external view returns (uint256);
function activeTotalRewards() external view returns (uint256);
```

No implementation change is needed — `paused()` is already provided by the inherited `Pausable` contract, and the `public` state variables in `Staking.sol` already satisfy the getter signatures automatically.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {Staking} from "./src/Staking.sol";
import {IStakingV1} from "./src/interfaces/IStakingV1.sol";

// Complete IStakingV1 — adds the essential and optional missing declarations.
interface IStakingV1Complete is IStakingV1 {
    function paused() external view returns (bool);            // essential
    function activeTotalStaked() external view returns (uint256);  // optional
    function activeTotalRewards() external view returns (uint256); // optional
}

contract MockERC20 {
    function balanceOf(address) external pure returns (uint256) { return type(uint256).max; }
    function transfer(address, uint256) external pure returns (bool) { return true; }
    function transferFrom(address, address, uint256) external pure returns (bool) { return true; }
    function allowance(address, address) external pure returns (uint256) { return type(uint256).max; }
    function approve(address, uint256) external pure returns (bool) { return true; }
}

contract MissingGettersTest is Test {
    Staking staking;

    function setUp() public {
        MockERC20 token = new MockERC20();
        staking = new Staking(address(this), address(this), address(this), address(token));
    }

    function test_InterfaceIdMismatch() public view {
        bytes4 incompleteId = type(IStakingV1).interfaceId;
        bytes4 completeId   = type(IStakingV1Complete).interfaceId;

        console.log("IStakingV1 interfaceId (declared)  :");
        console.logBytes4(incompleteId);
        console.log("IStakingV1Complete interfaceId     :");
        console.logBytes4(completeId);

        // The contract supports the declared (incomplete) interface.
        assertTrue(staking.supportsInterface(incompleteId), "should support declared interface");

        // The contract does NOT support the complete interface — even though Staking
        // implements paused(), activeTotalStaked(), and activeTotalRewards().
        assertFalse(staking.supportsInterface(completeId), "complete interfaceId should not be registered");

        console.log("supportsInterface(IStakingV1)         :", staking.supportsInterface(incompleteId));
        console.log("supportsInterface(IStakingV1Complete) :", staking.supportsInterface(completeId));
    }

    function test_MissingFunctionsAccessibleDirectly() public view {
        // All three functions exist on the concrete Staking contract...
        assertEq(staking.paused(),             false);
        assertEq(staking.activeTotalStaked(),  0);
        assertEq(staking.activeTotalRewards(), 0);
        console.log("paused()           (direct call):", staking.paused());
        console.log("activeTotalStaked  (direct call):", staking.activeTotalStaked());
        console.log("activeTotalRewards (direct call):", staking.activeTotalRewards());

        // ...but none are reachable through an IStakingV1 reference.
        // IStakingV1(address(staking)).paused();             // compile error: member not found
        // IStakingV1(address(staking)).activeTotalStaked();  // compile error: member not found
        // IStakingV1(address(staking)).activeTotalRewards(); // compile error: member not found
    }
}
```

**Command:**

```bash
forge test --match-path test/MissingGetters.t.sol -vv
```

**Output:**

```
Ran 2 tests for test/MissingGetters.t.sol:MissingGettersTest
[PASS] test_InterfaceIdMismatch() (gas: 19000)
Logs:
  IStakingV1 interfaceId (declared)  :
  0x6449774f
  IStakingV1Complete interfaceId     :
  0x7f8e7b77
  supportsInterface(IStakingV1)         : true
  supportsInterface(IStakingV1Complete) : false

[PASS] test_MissingFunctionsAccessibleDirectly() (gas: 23613)
Logs:
  paused()           (direct call): false
  activeTotalStaked  (direct call): 0
  activeTotalRewards (direct call): 0

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
