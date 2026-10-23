# #45517 \[SC-Insight] Partial Documentation for Self-Close Exit Fee Handling and Redemption Workflow in 'CollateralPool.sol'

**Submitted on May 16th 2025 at 00:33:39 UTC by @rusalka711 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45517
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**

## Description

## Brief/Intro

The `_selfCloseExitTo` and `fAssetRequiredForSelfCloseExit` functions lack critical explanations of their redemption logic, fee lifecycle management, and parameter constraints. Missing documentation increases the risk of user errors during self-close exits and complicates protocol maintenance.

***

## Vulnerability Details

#### **A. Undocumented Parameters**

1. **`_redeemToCollateral`**:
   * No explanation of when/why to use direct collateral redemption vs. standard chain redemption.
2. **`_redeemerUnderlyingAddress`**:
   * Missing examples (e.g., BTC/XRP address formats) and validation requirements.
3. **`_executor`**:
   * Unclear role (e.g., receives fees for manual redemptions on non-smart contract chains).

#### **B. Fee Lifecycle Ambiguity**

1. **Fee Debt Conversion**:

   ```solidity
   freeFAssetFeeShare -= spentFreeFAssetFeeShare;  
   debtFAssetFeeShare += spentFreeFAssetFeeShare;  
   ```

   * No context on how this affects future fee withdrawals or user balances.
2. **Allowance Checks**:

   ```solidity
   require(fAsset.allowance(...) >= ..., "f-asset allowance too small");  
   ```

   * Fails to specify that `transferFee` is paid by the user, not the contract.

#### **C. Collateral Ratio Logic**

1. **`_getFAssetRequiredToNotSpoilCR`**:
   * No documentation of its formula or impact on pool solvency.
2. **Agent Redemption Limits**:
   * Undefined consequences of `maxAgentRedemption < requiredFAssets` (e.g., partial exits).

#### **D. Event Emissions**

1. **`IncompleteSelfCloseExit`**:
   * No explanation of when/why this event is emitted.
2. **`Exited` Parameters**:
   * Undocumented fields (e.g., `spentFAssetFees` vs. `requiredFAssets`).

***

## Impact Details

**Category**: Documentation Improvements Insight

**Impact Analysis**:

* **No Direct Exploit**: The code operates correctly as written.
* **Operational Risks**:
  1. **Insufficient Allowances**: Users may underestimate `transferFee`, causing transaction reverts.
  2. **Collateral Misconfigurations**: Misunderstanding `_redeemToCollateral` could lead to failed redemptions.
  3. **Accounting Errors**: Poorly tracked fee debt may cause incorrect user balance calculations.

***

## References

* **Code File**: `CollateralPool.sol` (Lines 312–412).
* **Key Functions**:
  * `_selfCloseExitTo`
  * `fAssetRequiredForSelfCloseExit`
* **Critical Parameters**:
  * `_redeemToCollateral`
  * `_redeemerUnderlyingAddress`
  * `_executor`

## Proof of Concept

## Proof of Concept

#### **1. `_selfCloseExitTo` Function Documentation**

**Location**: Insert **above** the `_selfCloseExitTo` function.\
**Code**:

```solidity
/**
 * @notice Executes a self-close exit, redeeming pool tokens while preserving collateral ratios.
 * @param _tokenShare Pool tokens to liquidate (must be ≤ caller's balance).
 * @param _redeemToCollateral If true, bypasses lot size checks for direct collateral redemption.
 * @param _recipient Receives withdrawn NAT and FAsset fees.
 * @param _redeemerUnderlyingAddress Valid underlying chain address (e.g., BTC: bc1q..., XRP: r...).
 * @param _executor Address paid via `msg.value` to execute redemptions on non-smart contract chains.
 * @dev
 * - **Collateral Checks**: Post-exit balances must meet `MIN_TOKEN_SUPPLY_AFTER_EXIT`/`MIN_NAT_BALANCE_AFTER_EXIT`.
 * - **Agent Limits**: If `requiredFAssets > maxAgentRedemption`, reduces `_tokenShare` and `natShare` to stay within limits.
 * - **Fees**: Uses the caller’s FAsset fees first. Additional FAssets are transferred (caller covers fees).
 * - **Reentrancy**: Guarded by `nonReentrant` modifier.
 */
// slither-disable-next-line reentrancy-eth         // guarded by nonReentrant
function _selfCloseExitTo(...) private {
    // ... (existing code)
}
```

***

#### **2. `fAssetRequiredForSelfCloseExit` Function Documentation**

**Location**: Insert **above** the `fAssetRequiredForSelfCloseExit` function.\
**Code**:

```solidity
/**
 * @notice Computes FAssets needed for a self-close exit after deducting the user's available fees.
 * @param _tokenAmountWei Pool tokens to exit (in wei).
 * @return FAssets required to preserve collateral ratio.
 * @dev
 * - **Formula**: `requiredFAssets = max(0, (poolCR * (poolNatBalance - natWei)) / assetPrice - feeBalance)`.
 * - **Edge Case**: Returns 0 if user’s fees cover the full requirement.
 */
function fAssetRequiredForSelfCloseExit(...) external view returns (uint256) {
    // ... (existing code)
}
```

***

#### **3. Inline Comment for `_redeemerUnderlyingAddress`**

**Location**: Insert **above** the `_redeemerUnderlyingAddress` parameter usage (inside `_selfCloseExitTo`).\
**Code**:

```solidity
// For non-collateral redemptions: Provide a valid underlying chain address (e.g., BTC: bc1q..., XRP: r...).
string memory _redeemerUnderlyingAddress,
```

***

#### **4. Event Emission Comment**

**Location**: Insert **above** the `emit IncompleteSelfCloseExit` line.\
**Code**:

```solidity
// Emitted when agent's redemption capacity is exceeded, forcing partial exit.
emit IncompleteSelfCloseExit(_tokenShare, requiredFAssets);
```

***

#### **5. Parameter Validation Comment**

**Location**: Insert **above** the `requiredFAssets < assetManager.lotSize()` check.\
**Code**:

```solidity
// Direct collateral redemption bypasses lot size checks. Use for small amounts.
if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
    // ... (existing code)
}
```

***

### **Summary of Changes**

| Code Section                     | Documentation Added                                                     |
| -------------------------------- | ----------------------------------------------------------------------- |
| `_selfCloseExitTo`               | NatSpec for parameters, collateral checks, agent limits, and fee logic. |
| `fAssetRequiredForSelfCloseExit` | Formula explanation and edge-case handling.                             |
| `_redeemerUnderlyingAddress`     | Example address formats for clarity.                                    |
| `IncompleteSelfCloseExit`        | Context for partial exits due to agent limits.                          |
| `requiredFAssets` check          | Clarifies `_redeemToCollateral` usage for small amounts.                |

***

### **Result**

These comments:

1. **Prevent User Errors**: Clarify redemption paths, fee allowances, and address formats.
2. **Improve Auditability**: Document formulas and edge cases for critical logic.
3. **Enhance Maintainability**: Explain parameter interactions and event triggers.
