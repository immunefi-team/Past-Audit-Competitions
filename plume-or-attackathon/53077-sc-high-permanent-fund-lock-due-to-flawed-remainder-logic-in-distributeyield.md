# 53077 sc high permanent fund lock due to flawed remainder logic in distributeyield

* Submitted on Aug 14th 2025 at 18:58:57 UTC by @Alem for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #53077
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts:
  * Permanent freezing of funds

## Description

### Brief / Intro

A critical vulnerability exists in the `distributeYield` function's remainder-handling logic. If the last holder in the distribution iteration is ineligible to receive yield, the function withholds their share but fails to handle the remaining funds that were calculated for them. This remainder (the "dust" from preceding calculations) remains inside the contract. As there is no function to withdraw these trapped tokens, this results in a permanent and irrecoverable loss of capital for the yield distributor with every affected distribution cycle.

### Vulnerability Details

`distributeYield` distributes yield tokens to eligible holders and avoids precision loss by assigning the leftover amount (the remainder) to the final holder. The implementation checks whether the last holder is eligible (`_isYieldAllowed(lastHolder)`) before transferring the remainder. If the last holder is ineligible, the code simply skips transferring the remainder and does not provide any alternative handling. Since the full `amount` has already been transferred into the contract via `yToken.safeTransferFrom(msg.sender, address(this), amount)`, any undistributed portion remains in the contract, effectively locked.

Vulnerable code snippet:

```solidity
if (holderCount > 0) {
    address lastHolder = $.holders.at(lastProcessedIndex);
    
    // This 'if' statement is the source of the vulnerability.
    if (_isYieldAllowed(lastHolder)) {
        uint256 lastShare = amount - distributedSum;
        if (lastShare > 0) {
            yToken.safeTransfer(lastHolder, lastShare);
            distributedSum += lastShare;
        }
    }
    // MISSING LOGIC: There is no 'else' block here. If the check above
    // fails, the 'lastShare' is never handled. It's simply abandoned.
}

emit YieldDistributed(distributedSum, yieldTokenAddr);
```

`previewYieldDistribution` correctly simulates the distribution (showing a zero share for an ineligible last holder), which creates a discrepancy between preview and actual execution and can mislead a distributor into believing a safe outcome.

## Impact Details

* Permanent and irreversible loss of funds for the yield distributor when the last holder in the iteration is ineligible.
* The locked amount equals the remainder (rounding "dust") accumulated from the share calculations of preceding holders.
* Loss scales with number of holders and repeated distributions: small per-distribution but can become significant with many holders or many distributions.
* This is a recurring, silent loss each time the condition is met.

## Recommendation

Handle the undistributed remainder when the last holder is ineligible. The most robust fix is to refund any undistributed amount back to the original sender (`msg.sender`) so no funds remain locked in the contract.

Suggested handling (conceptual snippet to perform refund of remainder):

```solidity
// ... inside distributeYield, after the main distribution loop and last holder check ...

// After attempting all distributions, calculate what's left.
uint256 remainingInContract = amount - distributedSum;

if (remainingInContract > 0) {
    // Refund the undistributed remainder back to the original caller (the Distributor).
    yToken.safeTransfer(msg.sender, remainingInContract);
}

// The event should emit the amount that was *actually* distributed to holders.
emit YieldDistributed(distributedSum, yieldTokenAddr);
```

(Do not assume this exact snippet fits the contract structure — integrate consistently with existing state updates, access control, and checks.)

## Proof of Concept (PoC)

Actors:

* Distributor: account with YIELD\_DISTRIBUTOR\_ROLE.
* Alice: eligible holder with 400 ArcToken.
* Bob: eligible holder with 500 ArcToken.
* Charlie: ineligible holder with 100 ArcToken (last in holders array).

Holders ordered: \[Alice, Bob, Charlie]. Effective total supply for distribution = 400 + 500 = 900 (Charlie excluded).

{% stepper %}
{% step %}

### Step 1 — Preview

Distributor calls `previewYieldDistribution(10 ether)`. The preview correctly indicates:

* Alice ≈ 4.444... ETH
* Bob ≈ 5.555... ETH
* Charlie = 0 ETH The preview appears balanced.
  {% endstep %}

{% step %}

### Step 2 — Execution: Pull funds

Distributor calls `distributeYield(10 ether)`. Contract pulls 10 WETH from Distributor: `yToken.safeTransferFrom(msg.sender, address(this), amount)` => contract balance increases by 10 WETH.
{% endstep %}

{% step %}

### Step 3 — Distribute to first holders

The distribution loop processes Alice and Bob:

* Alice receives (10 \* 400) / 900 ≈ 4.444... WETH. `distributedSum` ≈ 4.444...
* Bob receives (10 \* 500) / 900 ≈ 5.555... WETH. `distributedSum` ≈ 9.999...
  {% endstep %}

{% step %}

### Step 4 — Final step & trap

The contract handles the last holder (Charlie). It checks `_isYieldAllowed(Charlie)` which returns false. Because of the missing `else`/fallback, the remainder: `lastShare = amount - distributedSum` (≈ 0.000...001 WETH) is not transferred out and remains in the contract. The function ends and emits `YieldDistributed(distributedSum, yieldTokenAddr)`.
{% endstep %}
{% endstepper %}

Result:

* Pulled: 10.0 WETH
* Distributed to holders: ≈ 9.999... WETH
* Permanently locked in contract: ≈ 0.000...001 WETH (the remainder) This remainder, while small per distribution, accumulates across distributions and holders and can become substantial. There is no contract function to withdraw arbitrary tokens, so these funds are irrecoverable.

## Notes

* Do not change links or add external references.
* The root cause: design assumes last holder will always receive remainder; it fails when last holder is ineligible.
* Recommended mitigation: explicitly handle remaining undistributed tokens (refund to sender or allocate to next eligible holder) and align `distributeYield` behavior with `previewYieldDistribution` semantics.
