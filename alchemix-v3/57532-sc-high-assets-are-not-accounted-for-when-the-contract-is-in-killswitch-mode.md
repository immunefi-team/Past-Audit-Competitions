# 57532 sc high assets are not accounted for when the contract is in killswitch mode

**Submitted on Oct 27th 2025 at 00:38:34 UTC by @ayden for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57532
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In killSwitch mode, the allocation step is skipped without reverting. As a result, assets can be transferred from the Morpho V2 Vault to the MYTStrategy contract. However, these transferred assets are not recorded in the Morpho V2 Vault’s allocation, nor are they included in the total assets of the MYTStrategy contract.

## Vulnerability Details

```solidity
    function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0)); <<<<<
        }
```

## Impact Details

If user transfer assets to a killSwitch mode Strategy , the assets become stuck

## References

## Proof of Concept

## Proof of Concept

add following test to AlchemistAllocator.t.sol contract.

```solidity
    function test_KillSwitchMode() public {
        address user = makeAddr("user");

        _magicDepositToVault(address(vault), user, 10e18);

        vm.startPrank(admin);
        bytes32 allocationId = mytStrategy.adapterId();

        //admin allocate 5e18 assest to mytStrategy contract.
        allocator.allocate(address(mytStrategy), 5e18);

        //admin user set mytStrategy to killSwitch Mode.
        mytStrategy.setKillSwitch(true);

        //admin allocate another 5e18 assest to mytStrategy contract.
        allocator.allocate(address(mytStrategy), 5e18);

        uint256 allocation = vault.allocation(allocationId);

        //the allocation is still 5e18.
        assertEq(allocation, 5e18);
        assertEq(TestERC20(mockVaultCollateral).balanceOf(address(vault)),0);
        assertEq(TestERC20(mockVaultCollateral).balanceOf(address(mytStrategy)),5e18);
    }

```
