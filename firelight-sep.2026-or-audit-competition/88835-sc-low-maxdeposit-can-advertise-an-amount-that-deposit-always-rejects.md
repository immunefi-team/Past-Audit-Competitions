# 88835 sc low maxdeposit can advertise an amount that deposit always rejects

**Submitted on Aug 15th 2026 at 05:52:22 UTC by @QED for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88835
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

FirelightVault can return a positive ERC-4626 `maxDeposit` quote even when depositing that exact amount deterministically reverts, causing maximum-amount integrations to fail.

### Vulnerability Details

**Affected target:** `FirelightVault` at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

**Severity:** Low

**Asset:** [`contracts/core/FirelightVault.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol)

#### Root Cause

[`maxDeposit`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L371-L377) returns the unused asset capacity without checking whether it converts to a nonzero number of shares, but [`deposit`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L671) floor-converts the assets and reverts with `InvalidAmount` when the result is zero shares. The conversion includes virtual terms in [`_convertToSharesTotals`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1065), so a positive remaining capacity can legitimately convert to zero.

#### Attack Path

{% stepper %}
{% step %}

## Transfer underlying units

An unprivileged account transfers 10 underlying units directly to an empty vault whose deposit limit is 11.
{% endstep %}

{% step %}

## Leave one unit of capacity

The vault has `totalAssets = 10`, `totalSupply = 0`, and one unit of remaining capacity.
{% endstep %}

{% step %}

## Receive an invalid quote

`maxDeposit(receiver)` returns 1, while `previewDeposit(1)` returns zero shares.
{% endstep %}

{% step %}

## Deposit the advertised maximum

A router or user follows the advertised maximum and calls `deposit(1, receiver)`.
{% endstep %}

{% step %}

## Transaction reverts

The zero-share guard reverts with `InvalidAmount` before any asset transfer.
{% endstep %}
{% endstepper %}

### Impact Details

ERC-4626 routers, frontends, or batched operations that deposit the advertised maximum can reliably revert and deposits can remain unavailable until the cap or vault state changes, although no assets are stolen.

## References

* [`FirelightVault.maxDeposit`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L365-L378)
* [`FirelightVault.deposit`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L674)
* [`FirelightVault` conversion formula](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1030-L1074)
* [ERC-4626 specification](https://eips.ethereum.org/EIPS/eip-4626)

## Recommended Fix

After calculating remaining capacity, return zero whenever `previewDeposit(remaining)` is zero and add regression tests for donated empty vaults and high share-price rounding edges.

## Link to Proof of Concept

<https://gist.github.com/w-qedaudit/91f523cdc78670872597603aeb7742de>

## Proof of Concept

**PoC gist:** <https://gist.github.com/w-qedaudit/91f523cdc78670872597603aeb7742de>

Prerequisite: Git, Node.js, and npm must be installed with network access available for cloning and dependency installation.

The supplied `poc.js` is a temporary focused reproducer copied into the checked-out repository at runtime; it is not an in-tree project test or modification.

```sh
cd firelight/qhf6dkc3-maxdeposit-unexecutable
chmod +x run.sh
./run.sh
```

Expected successful output includes:

```
state totalAssets=10 totalSupply=0 depositLimit=11
maxDeposit(receiver)=1
previewDeposit(maxDeposit)=0
deposit(maxDeposit, receiver)=reverted InvalidAmount; transaction=rejected by eth_sendTransaction
after depositorAssets=1 depositorShares=0
```

The complete output from the verified execution is included in `poc_output.txt`.
