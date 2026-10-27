# 58714 sc low pending admin cannot accept ownership in alchemistcurator sol&#x20;

**Submitted on Nov 4th 2025 at 08:17:45 UTC by @Bx4 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58714
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In the AlchemistCurator contract after Admin has transferred Ownership to new Admin the new `pendingAdmin` is not able to accept ownership because of the onlyAdmin modifier on `acceptAdminOwnership`

## Vulnerability Details

Looking at AlchemistCurator transfer owner functionality, it uses the two-step logic which means, the current admin transfers the ownership to the new admin and then the new pending admin has to accept admin privileges by calling `acceptAdminOwnership` to accept ownership.

But here is the case that whenever the new pending owner tries to call `acceptAdminOwnership` it always revert because of the onlyAdmin modifier present on the `acceptAdminOwnership`

`function acceptAdminOwnership() external onlyAdmin {`

```solidity
    modifier onlyAdmin() {
        require(msg.sender == admin, "PD");
        _;
    }
```

This modifier allows only admins to call `acceptAdminOwnership` and as pendingAdmin is not an admin it will revert.

In view of this, The implemented two-step transfer Ownership logic is flawed, and breaks the main pillars of the two-step transfer logic.

## Impact Details

This contract fails to honour the promised return of allowing the new pending admin to accept ownership when `pendingOwner` invokes `acceptAdminOwnership` .

## References

[AlchemistCurator](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L15)

[`AlchemistCurator::transferAdminOwnerShip`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L27)

[`AlchemistCurator::acceptAdminOwnership`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31)

## Proof of Concept

## Proof of Concept

Please add this test into `src/test/AlchemistCurator.t.sol`

```solidity
    function testAdminChangeVuln() public{
        // Current admin transfers ownership to new admin
        vm.startPrank(admin);
        address newAdmin = address(0xabc);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        vm.stopPrank();

        // new Admin accepting new ownership privileges
        vm.prank(newAdmin);
        vm.expectRevert(); // This will revert because of the onlyAdmin modifier which is expecting the current admin to call rather than pending
        mytCuratorProxy.acceptAdminOwnership();
    }
```

We can observe that the test passes and this is due to the revert that happens whenever the new `pendingAdmin` tries to accept Admin ownership.
