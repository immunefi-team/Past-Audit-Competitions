# #45405 \[SC-Insight] Insufficient Documentation for Governance-Controlled Functions and Critical Parameters in 'CoreVaultManager.sol'

**Submitted on May 14th 2025 at 00:34:54 UTC by @rusalka711 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45405
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**

## Description

## Brief/Intro

Governance functions like `updateSettings`, `addPreimageHashes`, and account management utilities lack explanations for parameter constraints, role definitions, and system dependencies. Missing context increases the risk of misconfiguration during governance actions, potentially destabilizing escrow mechanics or access control.

## Vulnerability Details

**Affected Components**:

1. **`addTriggeringAccounts`/`removeTriggeringAccounts`**:
   * No definition of "triggering accounts" or their privileges.
   * Example:

     ```solidity
     function addTriggeringAccounts(address[] calldata _triggeringAccounts)  
     // What permissions do these accounts have? How do they interact with `triggerInstructions`?  
     ```
2. **`updateCustodianAddress`**:
   * Undocumented impact of custodian changes on escrow lifecycle.

     ```solidity
     function updateCustodianAddress(string calldata _custodianAddress)  
     // How does this affect existing escrows or payment instructions?  
     ```
3. **`updateSettings`**:
   * Lack of rationale for constraints (e.g., `_escrowEndTimeSeconds < 1 days`).
   * No explanation of parameter interplay:

     ```solidity
     require(_escrowEndTimeSeconds < 1 days, "invalid end time");  
     // Why enforce <1 day? How does this align with UTC cycles?  
     ```
4. **`addPreimageHashes`**:
   * Missing context on preimage hash lifecycle (e.g., usage in escrows, redemption).

     ```solidity
     require(_preimageHashes[i] != bytes32(0), "invalid preimage hash");  
     // What is a preimage hash? How is it generated/used?  
     ```

## Impact Details

**Category**: Insight - Documentation Improvements

**Impact Analysis**:

* **No Direct Exploit**: Functions operate as intended.
* **Operational Risks**:
  * Incorrect escrow configurations due to misunderstood time parameters.
  * Invalid preimage hashes disrupting redemption workflows.
  * Privilege escalation if triggering accounts are misconfigured.

## References

* **Code File**: `CoreVaultManager.sol` (link: <https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>)
* **Key Functions**:
  * `updateSettings` (Lines 504-520).
  * `addPreimageHashes` (Lines 527-538).
  * Role management functions (Lines 449-460, 467-478 and 485-494).

## Proof of Concept

## Proof of Concept

#### **1. `updateSettings` Function**

**Improved Documentation**:

```solidity
/**
 * @notice Updates critical protocol parameters.  
 * @param _escrowEndTimeSeconds End time for escrows (in seconds within a UTC day, e.g., 79200 = 22:00 UTC). Must be < 86400.  
 * @param _escrowAmount Amount to escrow per batch. Set to 0 to disable escrows.  
 * @param _minimalAmount Minimum reserve balance retained in the vault after escrowing to prevent insolvency.  
 * @param _fee Per-request fee (deducted from available funds). Must be > 0.  
 * @dev Misconfiguration (e.g., setting _minimalAmount = 0) may risk vault insolvency.  
 */  
function updateSettings(...)  
```

***

#### **2. `addTriggeringAccounts` and `removeTriggeringAccounts`**

**Improved Documentation**:

```solidity
/**
 * @notice Authorizes addresses to trigger escrow/transfer processing.  
 * @param _triggeringAccounts List of addresses to grant triggering privileges.  
 * @dev Triggering accounts can call `triggerInstructions()`. Restrict to trusted actors.  
 * @dev Batch size limited to 50 addresses to avoid gas overflows.  
 */  
function addTriggeringAccounts(...)  

/**
 * @notice Revokes triggering privileges from addresses.  
 * @param _triggeringAccounts List of addresses to remove.  
 * @dev Removed accounts can no longer call `triggerInstructions()`.  
 */  
function removeTriggeringAccounts(...)  
```

***

#### **3. `addPreimageHashes` Function**

**Improved Documentation**:

```solidity
/**
 * @notice Adds cryptographic commitments (preimage hashes) for escrow redemptions.  
 * @param _preimageHashes List of unique, non-zero hashes (use keccak256(secret) for secure generation).  
 * @dev Preimage hashes must never be reused. Leaked secrets allow unauthorized redemptions.  
 */  
function addPreimageHashes(...)  
```

***

#### **4. `updateCustodianAddress` Function**

**Improved Documentation**:

```solidity
/**
 * @notice Updates the custodian address for future escrow redemptions.  
 * @param _custodianAddress New custodian address (must be non-empty).  
 * @dev Changes apply only to new escrows. Existing escrows retain the previous custodian.  
 */  
function updateCustodianAddress(...)  
```

***

#### **5. `addTriggeringAccounts` Function**

**Improved Documentation**:

```solidity
/**  
 * @notice Grants triggering privileges to specified addresses.  
 * @param _triggeringAccounts List of addresses to authorize.  
 * @dev  
 * - Authorized accounts can call `triggerInstructions()`, which processes escrows and transfers.  
 * - Batch size must be ≤ 50 addresses to avoid gas limits.  
 * - Triggering accounts are highly privileged; audit and restrict to trusted entities.  
 * @emit TriggeringAccountAdded: Successfully added addresses.  
 */  
function addTriggeringAccounts(address[] calldata _triggeringAccounts) external onlyGovernance {  
    // ...  
}  
```

***

#### **6. `removeTriggeringAccounts` Function**

**Improved Documentation**:

```solidity
/**  
 * @notice Revokes triggering privileges from specified addresses.  
 * @param _triggeringAccounts List of addresses to revoke.  
 * @dev  
 * - Removed addresses can no longer call `triggerInstructions()`.  
 * - Use this to mitigate risks from compromised or deprecated accounts.  
 * @emit TriggeringAccountRemoved: Successfully removed addresses.  
 */  
function removeTriggeringAccounts(address[] calldata _triggeringAccounts) external onlyGovernance {  
    // ...  
}  
```
