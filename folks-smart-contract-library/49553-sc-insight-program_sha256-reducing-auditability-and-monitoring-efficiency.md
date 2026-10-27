# #49553 \[SC-Insight] program\_sha256\`, Reducing Auditability and Monitoring Efficiency

**Submitted on Jul 17th 2025 at 07:36:39 UTC by @perseverance for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49553
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/Upgradeable.py>
* **Impacts:**

## Description

## Summary

The `UpgradeCancelled` event in `Upgradeable.py` currently logs only the cancellation timestamp. This omission makes it impossible for off-chain monitoring systems, indexers, and auditors to directly identify *which* scheduled upgrade was cancelled. Including the `program_sha256` in the event would significantly enhance traceability and align its data structure with the `UpgradeScheduled` and `UpgradeCompleted` events. This change improves the efficiency of off-chain monitoring and contributes to the system's long-term operational sustainability.

## Enhancement Details

The `Upgradeable.py` contract provides a robust mechanism for scheduling and executing contract upgrades. The process is accompanied by events for key actions: `UpgradeScheduled`, `UpgradeCompleted`, and `UpgradeCancelled`.

While `UpgradeScheduled` and `UpgradeCompleted` both include the `program_sha256`, providing a clear identifier for the contract code, `UpgradeCancelled` does not.

**Current Implementation:**

**File:** `contracts/library/Upgradeable.py`

```python
@abimethod
def cancel_contract_upgrade(self) -> None:
    # ... checks ...

    # delete scheduled upgrade
    self._check_upgrade_scheduled()
    del self.scheduled_contract_upgrade.value

    # The event only contains the timestamp
    emit(UpgradeCancelled(ARC4UInt64(Global.latest_timestamp)))
```

This implementation creates an operational inefficiency for any off-chain system. To determine which upgrade was cancelled, a monitoring service would need to:

1. Listen for an `UpgradeCancelled` event.
2. Query the historical state of the blockchain *before* the cancellation transaction to find the `scheduled_contract_upgrade` state value.
3. Extract the `program_sha256` from that historical state.

This process is computationally expensive, complex, and reduces the efficiency of indexing and monitoring tools.

## Severity Assessment

* **Severity:** Code Optimizations and Enhancements
* **Impact:** This recommendation is aimed at improving system performance and reducing operational costs for ecosystem tools. By including the `program_sha256` directly in the `UpgradeCancelled` event, the contract provides complete, context-rich data. This optimization eliminates the need for expensive historical state lookups by off-chain services, thereby reducing their computational overhead and resource utilization. This directly contributes to the long-term sustainability and operational efficiency of the monitoring infrastructure built around the protocol.

## Recommendation

To improve auditability and monitoring efficiency, the `UpgradeCancelled` event should be updated to include the `program_sha256`.

1. **Modify Event Structs:** Update the `UpgradeCancelled` struct definition in `contracts/library/interfaces/IUpgradeable.py`.

   **Recommended Change in `IUpgradeable.py`:**

   ```python
   class UpgradeCancelled(Struct):
       program_sha256: Bytes32
       timestamp: ARC4UInt64
   ```
2. **Update `cancel_contract_upgrade` Method:** Modify the implementation in `contracts/library/Upgradeable.py` to fetch the `program_sha256` before deleting the state and emit it in the event.

   **Recommended Change in `Upgradeable.py`:**

   ```python
   @abimethod
   def cancel_contract_upgrade(self) -> None:
       """Cancel the scheduled upgrade
       ...
       """
       self._only_initialised()
       self._check_sender_role(self.upgradable_admin_role())

       # Check for and copy the scheduled upgrade before deleting
       self._check_upgrade_scheduled()
       scheduled_upgrade = self.scheduled_contract_upgrade.value.copy()
       del self.scheduled_contract_upgrade.value

       # Emit the event with the program hash and timestamp
       emit(
           UpgradeCancelled(
               scheduled_upgrade.program_sha256,
               ARC4UInt64(Global.latest_timestamp)
           )
       )
   ```

This change makes the contract's event data more robust and efficient for external consumers without altering the core on-chain logic.

## Proof of Concept

## Proof of Concept

Not needed for insights
