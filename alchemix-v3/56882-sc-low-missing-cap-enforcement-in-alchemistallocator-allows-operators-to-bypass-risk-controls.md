# 56882 sc low missing cap enforcement in alchemistallocator allows operators to bypass risk controls

**Submitted on Oct 21st 2025 at 13:46:54 UTC by @ibrahimatix0x01 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56882
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The AlchemistAllocator contract calculates allocation caps for strategies but never enforces them. This critical vulnerability allows operators to bypass the entire risk management system and allocate unlimited amounts to strategies, potentially leading to overexposure to high-risk investments and undermining the protocol's risk controls, which could result in significant losses if risky strategies fail.

## Vulnerability Details

In both the `allocate()` and `deallocate()` functions, the AlchemistAllocator calculates an `adjusted` cap value by comparing different cap metrics (absoluteCap, relativeCap, daoTarget), but never actually uses this calculated value to limit the transaction amount:

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
    vault.allocate(adapter, oldAllocation, amount); // Never checks against 'adjusted'
}
```

The vulnerability exists because:

1. The code calculates a cap (adjusted) that should limit allocations for operators
2. There's no comparison between amount and adjusted before proceeding with the allocation
3. The original unchecked amount is passed directly to vault.allocate()
4. The same issue exists in the deallocate() function

The "FIXME" comment suggests awareness that part of the calculation needed improvement, but the fundamental issue of not enforcing the calculated cap was missed entirely.

## Impact Details

This vulnerability completely undermines the cap system designed to limit strategy allocation sizes, which is a critical risk control mechanism in DeFi protocols. The potential impacts include:

1. Risk Control Bypass: Operators can allocate unlimited funds to high-risk strategies, completely bypassing governance-approved risk limits.
2. Portfolio Imbalance: The protocol's portfolio could become severely imbalanced, with excessive allocation to certain strategies.
3. Increased Loss Potential: If strategies receiving excessive allocations fail or underperform, the losses would be magnified.
4. Governance Subversion: The vulnerability allows operators to effectively override governance decisions on allocation caps. While this vulnerability does not directly result in fund loss, it creates conditions where losses could be significantly amplified if risky strategies fail. This represents a failure of the contract to deliver its promised risk management controls, which are essential to the protocol's security model.

## References

Vault V2 documentation on allocation caps and risk management MYT (Meta Yield Token) Strategy documentation Protocol governance docs regarding operator permissions

## Proof of Concept

## Proof of Concept

In `AlchemistAllocator.t.sol` add these imports `import "forge-std/console.sol";` `import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";` and the add the below foundry test function:

```solidity
function testOperatorBypassesCaps() public {
    // Import console for logging
    
    
    // 1. Set up the vault with funds
    _magicDepositToVault(address(vault), user1, 150 ether);
    
    // 2. Check the current caps
    bytes32 allocationId = mytStrategy.adapterId();
    uint256 absoluteCap = vault.absoluteCap(allocationId);
    uint256 relativeCap = vault.relativeCap(allocationId);
    
    // Determine which cap is smaller (the one that should be enforced)
    uint256 effectiveCap = absoluteCap < relativeCap * 150 ether / 1e18 ? absoluteCap : relativeCap * 150 ether;
    
    // 3. Set a small allocation as baseline
    vm.startPrank(operator);
    uint256 smallAmount = 5 ether;
    allocator.allocate(address(mytStrategy), smallAmount);
    
    // Verify initial allocation succeeded
    uint256 allocation = vault.allocation(allocationId);
    assertEq(allocation, smallAmount, "Initial allocation should be set correctly");
    
    // 4. Now allocate a larger amount that would exceed the cap but is less than available funds
    // Use 90% of the vault's funds which should exceed any reasonable cap
    uint256 largeAmount = 140 ether; // 145 ether available but leaving some margin
    
    // This should fail if caps were enforced, but will succeed because the
    // calculated 'adjusted' cap is never compared with the amount
    allocator.allocate(address(mytStrategy), largeAmount);
    
    // Verify the allocation succeeded despite exceeding the cap
    allocation = vault.allocation(allocationId);
    assertEq(allocation, smallAmount + largeAmount, "Allocation exceeds the cap");
    
    // Check that the strategy received the full allocation amount
    uint256 strategyBalance = IMockYieldToken(mockStrategyYieldToken).balanceOf(address(mytStrategy));
    assertEq(strategyBalance, smallAmount + largeAmount, "Strategy balance matches full allocation");
    vm.stopPrank();
}
```

The test can be run by `forge test --mt testOperatorBypassesCaps -vvvv`
