# 59615 sc high off by one error in period boundary check allows theft of unclaimed yield after delegation exit

**Submitted on Nov 14th 2025 at 06:09:59 UTC by @csanuragjain for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59615
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

The `_claimableDelegationPeriods()` function in `Stargate.sol` contains an off-by-one error in the boundary condition check. When a user's `endPeriod` equals their `nextClaimablePeriod`, the comparison `endPeriod > nextClaimablePeriod` evaluates to FALSE, causing the function to return `completedPeriods` instead of `endPeriod` as the upper claimable bound. This allows users to claim VTHO rewards for periods after their delegation has ended, resulting in theft of unclaimed yield from legitimate delegators.

### Vulnerability Details

Location: `Stargate.sol` - `_claimableDelegationPeriods()` function (around line \~812)

Relevant snippet (simplified):

```solidity
function _claimableDelegationPeriods(
    StargateStorage storage $,
    uint256 _tokenId
) private view returns (uint32, uint32) {

    // ... code ...

    uint32 currentValidatorPeriod = completedPeriods + 1;
    uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;

    // BUG: Line ~812
    if (
        endPeriod != type(uint32).max &&
        endPeriod < currentValidatorPeriod &&
        endPeriod > nextClaimablePeriod  //  Should be >=
    ) {
        return (nextClaimablePeriod, endPeriod);
    }

    // Falls through when endPeriod == nextClaimablePeriod
    if (nextClaimablePeriod < currentValidatorPeriod) {
        return (nextClaimablePeriod, completedPeriods); //  Returns wrong upper bound
    }

    return (0, 0);
}
```

The bug:

* When `endPeriod == nextClaimablePeriod` (e.g., both equal 10):
  * `endPeriod > nextClaimablePeriod` is `10 > 10 = FALSE`
  * Code falls through to the second condition and returns `(nextClaimablePeriod, completedPeriods)` instead of `(nextClaimablePeriod, endPeriod)`
  * This allows claiming rewards for periods after the delegation ended.

Example scenario:

* User state:
  * `lastClaimedPeriod = 9`
  * `nextClaimablePeriod = 10`
  * `delegationEndPeriod = 10` (user exited at period 10)
* Validator state:
  * `completedPeriods = 11`
  * `currentValidatorPeriod = 12`

Buggy behavior:

* Check: `10 > 10 = FALSE`
* Falls through
* Returns: `(10, 11)` (incorrect)

Expected behavior (with fix `>=`):

* Check: `10 >= 10 = TRUE`
* Returns: `(10, 10)` (correct)

## Impact Details

**Severity:** HIGH — Theft of unclaimed yield

Direct impact:

1. Users can receive VTHO rewards for periods when they were not delegated.
2. Legitimate delegators receive diluted rewards.
3. The protocol loses VTHO tokens.
4. Exploit is repeatable for each period after exit.

Financial impact example (illustrative):

* Assume 100 VTHO per period for validator rewards.
* User exited at period 10.
* Validator completes N additional periods (11, 12, ...).
* If there are 9 legitimate delegators remaining:
  * Without bug: Period 11 -> 100 VTHO / 9 = 11.11 VTHO each
  * With bug: Period 11 -> 100 VTHO / 10 (including exited user) = 10 VTHO each
  * Exited user takes 10 VTHO; each legitimate delegator loses \~1.11 VTHO.
* If the user delays claiming for N periods after exit, they steal N × 10 VTHO in this example.

Real-world note: Project history includes prior rewards calculation issues that required reimbursement; this vulnerability could create a similar situation requiring manual remediation.

## References

* Code Location:
  * File: `packages/contracts/contracts/Stargate.sol`
  * Function: `_claimableDelegationPeriods`
  * Lines: 788-820 (bug at line \~812)
* Related Functions:
  * `_claimRewards()` — calls `_claimableDelegationPeriods()`
  * `_claimableRewards()` — uses returned period range for calculations
  * `claimRewards()` — public function users call to exploit

## Link to Proof of Concept

<https://gist.github.com/mariana617/d80b31fd1d63e06c8cab3bcc84f170dc>

## Proof of Concept

Reproduction steps:

{% stepper %}
{% step %}

### Clone repository

Clone the repository.
{% endstep %}

{% step %}

### Navigate to contracts

Change to the contracts package:

```
cd packages/contracts
```

{% endstep %}

{% step %}

### Create test file

Create test file `test/unit/poc.ts` with the provided code in: <https://gist.github.com/mariana617/d80b31fd1d63e06c8cab3bcc84f170dc>
{% endstep %}

{% step %}

### Run test

Run:

```
npx hardhat test test/unit/poc.ts
```

{% endstep %}
{% endstepper %}

### Expected Output

```
Period Boundary Bug
User delegated until period: 10
User can claim periods: 10 to 11
Bug: User claims period 11 despite exiting at 10

 demonstrates endPeriod == nextClaimablePeriod allows claiming extra periods
User delegated until period: 10
User can claim periods: 10 to 10
Fixed: User correctly claims only until 10
     fix: using >= instead of >

2 passing
```

## Suggested Fix

Change the comparison from `endPeriod > nextClaimablePeriod` to `endPeriod >= nextClaimablePeriod` so that when `endPeriod == nextClaimablePeriod`, the function correctly returns `(nextClaimablePeriod, endPeriod)`.

(Do not modify any other logic or links — the only needed change is the boundary comparison operator as described.)
