# 57704 sc high missing global state update in forcerepay leads to permanent freezing of unclaimed yield

**Submitted on Oct 28th 2025 at 10:07:53 UTC by @Diavol0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57704
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

### Summary

The `_forceRepay()` function in AlchemistV3 contains a critical accounting bug where it correctly decreases the user's `account.earmarked` during liquidation but **fails to decrease the global `cumulativeEarmarked` variable**. This synchronization failure causes the global earmarked accounting to become permanently inflated with each liquidation event, progressively restricting the protocol's ability to earmark new debt and ultimately freezing users' unclaimed yield in the Transmuter redemption system.

### Root Cause

AlchemistV3's earmark mechanism operates with two distinct synchronization models:

**1. Asynchronous Allocation (earmark increases):**

* Global `cumulativeEarmarked` updates immediately when `_earmark()` is called
* User `account.earmarked` updates later when `_sync()` is triggered
* Creates temporary state: `cumulativeEarmarked >= sum(account.earmarked)`

**2. Synchronous Operations (earmark decreases):**

* For direct repayment operations (`repay()` and `_forceRepay()`), the critical invariant is:

  ```
  Delta(cumulativeEarmarked) == sum(Delta(account.earmarked))
  ```
* When users directly repay earmarked debt, **user-level and global-level reductions must occur in lockstep**

The bug violates the synchronization invariant in `_forceRepay()`:

**Correct Implementation in `repay()` (Lines 521-526):**

```solidity
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;

// ✓ Correctly updates global state
uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= earmarkPaidGlobal;
```

**Buggy Implementation in `_forceRepay()` (Lines 760-762):**

```solidity
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;

// ✗ Missing global state update
// cumulativeEarmarked should decrease here but doesn't
```

### Impact Analysis

**Immediate Impact:**

1. **Global Accounting Inflation**: `cumulativeEarmarked` becomes inflated by the earmarked amount repaid in each liquidation
2. **Earmark Capacity Reduction**: `liveUnearmarked = totalDebt - cumulativeEarmarked` is understated, limiting future earmark operations
3. **Transmuter Redemption Impairment**: Users' yield cannot be properly earmarked and redeemed through Transmuter

**Cumulative Effect:**

* Each liquidation of earmarked debt exacerbates the problem
* Inflation accumulates: after N liquidations with total earmarked E, global state is inflated by E
* No self-healing mechanism exists

**Ultimate Consequence:** When `cumulativeEarmarked` approaches `totalDebt`:

* `liveUnearmarked ≈ 0`
* New earmark operations become impossible
* **Unclaimed yield is permanently frozen** - users cannot redeem through Transmuter
* Requires protocol upgrade to restore functionality

## Link to Proof of Concept

<https://gist.github.com/6newbie/698cbf05723ff5686c31b3d60d171813>

## Proof of Concept

### Step-by-Step Reproduction

**Prerequisites:**

* AlchemistV3 contract deployed and initialized
* Transmuter contract operational
* User positions with earmarked debt exist

**Steps:**

1. **Setup Initial State**

```

- User creates position with collateral and borrows debt

- Transmuter redemption is created to trigger earmarking

- Advance blocks to earmark 60% of user's debt

- Initial state:

* totalDebt = 180e18

* cumulativeEarmarked = 108e18

* User earmarked = 108e18

```

2. **Trigger Liquidation**

```

- Manipulate yield token price to make position undercollateralized

- Call liquidate(accountId)

- This internally calls _forceRepay(accountId, 108e18)

```

3. **Observe Bug**

```

After liquidation:

- User earmarked: 108e18 → 0 (✓ correctly reduced)

- totalDebt: 180e18 → 72e18 (✓ correctly reduced via _subDebt)

- cumulativeEarmarked: 108e18 → 72e18 (✗ SHOULD BE 0!)


Bug Evidence:

- Expected cumulativeEarmarked: 0 (108e18 - 108e18)

- Actual cumulativeEarmarked: 72e18

- Inflation: 72e18

```

4. **Verify Impact on Future Earmarks**

```

- liveUnearmarked = totalDebt - cumulativeEarmarked

- With bug: 72e18 - 72e18 = 0

- Expected: 72e18 - 0 = 72e18

- Result: Future earmarks are completely blocked (0 capacity)

```

### Automated Test Verification

A comprehensive test case has been implemented at:

* **File**: `src/test/AlchemistV3.t.sol`
* **Function**: `testBug_ForceRepay_Missing_CumulativeEarmarked_Update()` (Lines 3936-4066)

**Run test:**

```bash

forge test --match-test testBug_ForceRepay_Missing_CumulativeEarmarked_Update -vv

```

**Expected output:**

```

[FAIL: BUG: cumulativeEarmarked should decrease by earmarked amount but it doesn't!]

  

Logs:

=== Before Liquidation ===

User earmarked: 108000000000000000010800

Global cumulativeEarmarked: 108000000000000000010800

  

=== After Liquidation ===

User earmarked: 0

Global cumulativeEarmarked: 72000000000000000007200 ← Should be 0

  

=== Bug Evidence ===

Expected: 0

Actual: 72000000000000000007200

Inflation: 72000000000000000007200

```

The test demonstrates:

* ✅ User-level earmarked correctly reduced to 0
* ✅ Total debt correctly reduced by 108e18
* ❌ **Global cumulativeEarmarked incorrectly retains 72e18 inflation**

### Comparison with Correct Implementation

**Correct behavior (repay function):**

```

User calls repay():

- account.earmarked -= X ✓

- cumulativeEarmarked -= X ✓

- Both synchronized

```

**Buggy behavior (\_forceRepay function):**

```

Liquidation calls _forceRepay():

- account.earmarked -= X ✓

- cumulativeEarmarked unchanged ✗

- Synchronization broken

```
