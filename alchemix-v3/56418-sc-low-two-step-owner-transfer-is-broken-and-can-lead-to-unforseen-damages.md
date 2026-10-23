# 56418 sc low two step owner transfer is broken and can lead to unforseen damages

**Submitted on Oct 15th 2025 at 19:24:07 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56418
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency
  * Ownership at risk

## Description

## Brief/Intro

The two-step ownership system is designed to protect against accidental ownership transfer or loss of ownership. However, with the way it is implemented in <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol?utm\\_source=immunefi#L27-L35>, ownership can easily be lost, renounced (transferred to address zero) or transferred to an inactive address (i.e being lost)

## Vulnerability Details

The way traditional two-step ownership transfer works is that the admin transfers ownership to a new admin by calling `transferAdminOwnerShip`, and that new admin accepts the ownership by calling `acceptAdminOwnership`. This system prevents the accidental loss of admin privileges.

However, with the way the admin transfer system is designed, calling `acceptAdminOwnership` by the current admin when pendingAdmin is address(0) will lead to loss of admin rights.

See OpenZeppelin implementation here <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol>

```sol
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);//@audit not intended design
    }
```

## Impact Details

1. Accidental renouncing of admin rights: If `acceptAdminOwnership` is called when pendingAdmin is equal to address(0), admin rights will be lost. This can easily happen accidentally if the transaction is sent twice.
2. Loss of admin rights to an invalid address: While not likely to occur, the original `acceptAdminOwnership` requires that the new owner is the "mes.sender"; this condition ensures that the new admin is an active wallet on that chain.
3. Finally, it breaks the whole purpose or design of the two-step ownership transfer, because only one address is involved in the whole process.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol?utm\\_source=immunefi#L27-L35>

## Recommendation

Use the original openzepplin implementation: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol>

## Proof of Concept

## Proof of Concept

I will be providing two POCs. Please add the following test to `test/AlchemistCurator.t.sol`

### Accidental Renounce

```sol
    function testTransferAdminOwnerShipRenouce() public {
        vm.startPrank(admin);
        mytCuratorProxy.acceptAdminOwnership();
    }
```

***Output***

You can see that the admin has been accidentally renouced and set to address zero

```bash
[PASS] testTransferAdminOwnerShipRenouce() (gas: 17031)
Traces:
  [21831] AlchemistCuratorTest::testTransferAdminOwnerShipRenouce()
    ├─ [0] VM::startPrank(0x4444444444444444444444444444444444444444)
    │   └─ ← [Return]
    ├─ [10427] MockAlchemistCurator::acceptAdminOwnership()
    │   ├─ emit AdminChanged(admin: 0x0000000000000000000000000000000000000000)
    │   └─ ← [Return]
    └─ ← [Stop]
```

### New Admin cannot accept ownership

```sol
    function testTransferAdminOwnerShipRevertsWhenNewAdminTriesToClaim() public {
        address newAdmin = address(0x5555555555555555555555555555555555555555);
        vm.startPrank(admin);
        mytCuratorProxy.transferAdminOwnerShip(address(0));
        vm.startPrank(newAdmin);
        mytCuratorProxy.acceptAdminOwnership();

    }
```

***Output*** A new admin or a pending admin cannot accept ownership, thereby breaking the core design of the two-step ownership transfer.

```bash

  [20315] AlchemistCuratorTest::testTransferAdminOwnerShipRevertsWhenNewAdminTriesToClaim()
    ├─ [0] VM::startPrank(0x4444444444444444444444444444444444444444)
    │   └─ ← [Return]
    ├─ [5625] MockAlchemistCurator::transferAdminOwnerShip(0x0000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x5555555555555555555555555555555555555555)
    │   └─ ← [Return]
    ├─ [782] MockAlchemistCurator::acceptAdminOwnership()
    │   └─ ← [Revert] revert: PD
    └─ ← [Revert] revert: PD
```
