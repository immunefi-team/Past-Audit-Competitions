# 56583 sc low wrong 2 step transferadminownership logic and insufficient checks in alchemistcurator sol leads to permanent admin ownership loss&#x20;

**Submitted on Oct 17th 2025 at 23:19:22 UTC by @hunter0xweb3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56583
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

AlchemistCurator implements a two step process for transfering ownership of the contract\
However the implementation logic of this process has flaws and lacks of checks that makes the two step process ineffective and could lead to permanent ownership loss as shown next

## Vulnerability Details

The transfer ownership process is implemented in two steps

1. Current admin call to AlchemistCurator::transferAdminOwnerShip(\_newAdmin) to transfer ownership to NewAdmin (NA), this sets the pending admin variable equal to \_newAdmin
2. Call to acceptAdminOwnership()

The issues arises because the current implementation of acceptAdminOwnership requires the caller must be current admin (onlyAdmin modifier) instead of pending admin

```solidity
@>    function acceptAdminOwnership() external onlyAdmin {                       
        admin = pendingAdmin;                                                  
        pendingAdmin = address(0);                                             
        emit AdminChanged(admin);
    }      
```

This makes the 2 step transfer admin ownership ineffective, because pendingAdmin should be the one accepting the admin ownership in acceptAdminOwnership() method

The second issue occurs in acceptAdminOwnership() because it doesnt verify pendingAdmin variable value is not the zero address before changing admin variable

```solidity
    function acceptAdminOwnership() external onlyAdmin {                       
@>      admin = pendingAdmin;                                                  
        pendingAdmin = address(0);                                             
        emit AdminChanged(admin);
    }     
```

If admin calls acceptAdminOwnership() and pendingAdmin hasnt been initialized then ownership of AlchemistCurator is permanently lost

## Impact Details

* Permanent admin ownership lost
* Ineffective two step admin transfership procedure

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L27-L35>

## Proof of Concept

## Proof of Concept

The following PoC shows how the current implementation could lead to permanent admin ownership loss\
Write the following test case in src/test/AlchemistCurator.t.sol

```solidity
function testAdmin2StepOwnershipLoss() public {
    bytes32 ownerRaw;
    address owner;

    console.log("Current owner");
    ownerRaw = vm.load(address(mytCuratorProxy), bytes32(0));
    owner = address(uint160(uint256(ownerRaw)));             
    console.logAddress(owner);

    console.log("Calling acceptAdminOwnership");             
    vm.startPrank(admin);
    mytCuratorProxy.acceptAdminOwnership();

    console.log("Current owner after acceptAdminOwnership"); 
    ownerRaw = vm.load(address(mytCuratorProxy), bytes32(0));
    owner = address(uint160(uint256(ownerRaw)));
    console.logAddress(owner);
}
```

Exec test and observe admin ownership is permanent loss

```bash
reset;time forge test --mt testAdmin2StepOwnershipLoss
Logs:
  Current owner
  0x4444444444444444444444444444444444444444
  Calling acceptAdminOwnership
  Current owner after acceptAdminOwnership
  0x0000000000000000000000000000000000000000
```

**Recomendation**

* Should check pendingAdmin != address(0)
* acceptAdminOwnership function should check msg.sender is pendingAdmin

```solidity
function acceptAdminOwnership() external onlyAdmin { 
    require(msg.sender == pendingAdmin, "Not PAdmin");
    admin = pendingAdmin;                       
    pendingAdmin = address(0);
    emit AdminChanged(admin); 
}
```
