# 58215 sc high funds can become permanently stuck in adapter when kill switch is enabled

**Submitted on Oct 31st 2025 at 13:01:14 UTC by @mohitisimmortal for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58215
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary:

When `killSwitch` is enabled in the adapter (MYTStrategy), the adapter does not allocate funds, but the `Morpho vault still transfers collateral to the adapter`. Because no recovery or fallback mechanism exists, these funds become irrecoverably stuck inside the adapter contract. Disabling killSwitch and attempting deallocate does not help because no allocation took place, so there is nothing to unwind, leaving the vault with a permanent loss.

## Description:

1. Allocation flow:

* AlchemistAllocator.allocate() → calls Morpho vault.
* Morpho vault executes allocateInternal().
* allocateInternal() transfers assets to adapter and then calls the adapter’s allocate() function:

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        ...

>>      SafeERC20Lib.safeTransfer(asset, adapter, assets); // Morpho sends assets to adapter 
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
        ...

        }
```

2. Bug condition:

If `killSwitch == true` in the adapter:

```solidity
MYTStrategy::
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0)); // No allocation happens, funds stuck in adapter
        }
        ...
    }

```

* The asset transfer still happened, but the adapter does not allocate into any strategy.
* No recover() or fallback logic exists.
* Funds remain stuck in the adapter contract.
* Later, when attempting deallocate(), it fails to return funds because:
  * Nothing was allocated → nothing to unwind → still stuck.

## Impact:

Permanent loss/stuck of vault asset.

## Likelihood:

* killswitch is rarely be true, but as many strategies are there, so medium-high likelihood is possible.

## Recommendation:

Implement a recovery path for funds when killSwitch is active.

Option A : Add recover() function in adapter:

Option B: Modify allocate() to immediately return transferred funds when killSwitch == true:

Option C: During deallocate(), detect unallocated idle balance and transfer directly back to Morpho vault.

## Proof of Concept

## Proof of Concept

* add below import and testcase into `AlchemistAllocator.t.sol`:

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

function testfundsStuckInAdapter() public {
        // Enable kill switch
        vm.prank(admin);
        mytStrategy.setKillSwitch(true);

        _magicDepositToVault(address(vault), user1, 150 ether);
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), 100 ether);
        bytes32 allocationId = mytStrategy.adapterId();
        uint256 allocation = vault.allocation(allocationId);
        
        // Now, adapter(strategy) have those funds of morpho vault, and no way to recover those from adapter
        uint256 bal = IERC20(mockVaultCollateral).balanceOf(address(mytStrategy));
        assertEq(bal, 100 ether);

        // if try to deallocate those from adapter, then that also fails in both cases(killswitch or non-killswitch)
        // allocator.deallocate(address(mytStrategy), 50 ether);
        vm.stopPrank();
    }
```

* run by `forge test --mt testfundsStuckInAdapter -vvvv`
* output:

```solidity
TestERC20::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   └─ ← [Return] 100000000000000000000 [1e20]
```
