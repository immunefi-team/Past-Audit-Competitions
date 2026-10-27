# 57975 sc low broken admin rotation in acceptadminownership causes permanent governance lockout

**Submitted on Oct 29th 2025 at 18:33:46 UTC by @al0x23 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57975
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Smart contract unable to operate (governance liveness failure)

## Description

## Brief/Intro

The acceptAdminOwnership() function in the AlchemistCurator contract uses the onlyAdmin modifier, which restricts its execution to the current admin. As a result, the pendingAdmin — who is supposed to accept ownership — cannot complete the handover. This design flaw prevents any future admin rotation and can permanently lock governance operations if the current admin loses access to their key. Because the curator contract is itself timelocked and controls configuration of the main VaultV2 contract, this bug introduces a long-term liveness failure in governance.

## Vulnerability Details

The AlchemistCurator implements a two-step admin transfer mechanism:

address public admin; address public pendingAdmin;

modifier onlyAdmin() { require(msg.sender == admin, "PD"); \_; }

function transferAdminOwnership(address newAdmin) external onlyAdmin { pendingAdmin = newAdmin; }

function acceptAdminOwnership() external onlyAdmin { admin = pendingAdmin; pendingAdmin = address(0); emit AdminChanged(admin); }

The onlyAdmin modifier is applied to both functions. This means:

Only the current admin can call both transferAdminOwnership() and acceptAdminOwnership().

The intended pendingAdmin can never call acceptAdminOwnership() to finalize the role transfer.

Expected behavior:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "NOT_PENDING_ADMIN");
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

Actual behavior: The current admin must call acceptAdminOwnership() to complete the transfer, which defeats the purpose of having a two-step ownership mechanism. If the current admin loses access or is offboarded, the protocol cannot complete the rotation, and the admin role remains forever locked.

Timelock Considerations

The AlchemistCurator interacts with the timelocked functions in the main VaultV2 contract. Because curator actions (like setting caps, allocators, and fees) are executed via timelocks, a working admin key is required to submit and approve transactions over time.

If the admin is lost or the transfer is stuck:

No new timelocked operations can be initiated.

No governance actions (e.g., adjusting caps, adding adapters, or modifying fees) can be executed.

Critical updates requiring curator/admin actions become permanently unavailable.

Even though the VaultV2.owner could technically deploy a new curator and assign it via setCurator, this is a heavy manual recovery that breaks timelock continuity and governance integrity.

## Impact Details

Impact:

Governance and curator operations are permanently frozen once the current admin key is not available, and the changeAdmin intended functionality is 100% broken in any scenario. No new admin can take control, even though the pendingAdmin variable is correctly set. The bug doesn’t cause loss of funds, but causes governance paralysis, blocking timelocked configuration updates (e.g., cap adjustments, gate settings, or allocator role changes).

## Proof of Concept

## Proof of Concept

paste this in src/test/AlchemistCurator.t.sol

```solidity

 function test_PoC_changeAdmin() external {
        address newAdmin = makeAddr("newAdmin");
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        vm.prank(newAdmin);
        vm.expectRevert(bytes("PD"));
        mytCuratorProxy.acceptAdminOwnership();
    }

```
