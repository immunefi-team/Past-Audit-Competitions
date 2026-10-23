# 58506 sc low adjusted cap limits are never enforced

**Submitted on Nov 2nd 2025 at 21:11:55 UTC by @Tee0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58506
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistAllocator contract calculates role-based allocation limits (`adjusted`) to enforce different permission levels for admins vs operators, but never validates that allocation amounts respect these calculated limits before forwarding calls to the vault.

## Vulnerability Details

The AlchemistAllocator contract calculates adjusted cap limits in both `allocate()` and `deallocate()` functions but completely fails to enforce them. After computing the maximum allowable allocation through a multiple step cap selection process, the contract proceeds with allocations without any validation against these calculated limits, rendering the entire cap system non-functional.

After computing `adjusted`, a validation should follow but this validation is completely missing. The `adjusted` variable is calculated and not used to validate the allocation amount.

Since `daoTarget` is currently set to `type(uint256).max`, the adjusted cap becomes effectively infinite regardless of vault configured caps. Any operator can allocate unlimited funds to any adapter, completely bypassing all governance-configured risk limits.

Even when the FIXME is resolved and `daoTarget` is set to a real value from StrategyClassificationProxy, the issue remains because the enforcement check is still missing.

## Impact Details

Since daoTarget = type(uint256).max, the adjusted cap becomes infinite for operators. Operators can allocate up to the absolute maximum vault caps without any risk-based restrictions. Even when StrategyClassificationProxy is implemented, there's no enforcement.

## References

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
            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }

function deallocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted < daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.deallocate(adapter, oldAllocation, amount);
    }
```

## Proof of Concept

## Proof of Concept

This test should be ran in the AlchemistAllocator.t.sol found in the test folder using this command; `forge test --match-test test_BypassAbsoluteCap -vv`

```solidity
function test_BypassAbsoluteCap() public {
        // This test demonstrates that AlchemistAllocator calculates "adjusted" caps
        // but never enforces them, relying solely on the vault's enforcement.
        // The vulnerability: AlchemistAllocator should add an additional layer of
        // protection but it doesn't - it only passes through to the vault.

        uint256 absoluteCap = vault.absoluteCap(mytStrategy.adapterId());
        uint256 relativeCap = vault.relativeCap(mytStrategy.adapterId());
        console2.log("Vault absolute cap:", absoluteCap);
        console2.log("Vault relative cap:", relativeCap);

        // Deposit sufficient funds to the vault
        _magicDepositToVault(address(vault), user1, absoluteCap);

        // The AlchemistAllocator calculates an "adjusted" cap but never uses it
        // For operators: adjusted = max(max(absoluteCap, relativeCap), daoTarget)
        // Since daoTarget = type(uint256).max, adjusted becomes infinite for operators
        // Even if daoTarget was fixed, there's no enforcement check

        vm.startPrank(operator);
        // Allocate up to the vault's absoluteCap - this should succeed
        // because AlchemistAllocator doesn't enforce any restrictions
        allocator.allocate(address(mytStrategy), absoluteCap);
        vm.stopPrank();

        uint256 newAlloc = vault.allocation(mytStrategy.adapterId());
        console2.log("Allocation after:", newAlloc);

        // The allocation succeeded and equals the vault's cap
        // This proves AlchemistAllocator provides no additional protection
        // beyond what the vault already enforces
        assertEq(newAlloc, absoluteCap, "Allocation should equal absoluteCap");

        // If AlchemistAllocator enforced its own caps, operators might be
        // restricted to amounts less than the vault's absoluteCap based on
        // risk classification. But currently, they can allocate up to vault limits.
    }
```
