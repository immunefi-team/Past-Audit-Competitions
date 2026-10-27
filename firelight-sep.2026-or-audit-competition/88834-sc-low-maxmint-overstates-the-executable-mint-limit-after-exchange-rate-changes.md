# 88834 sc low maxmint overstates the executable mint limit after exchange rate changes

**Submitted on Aug 15th 2026 at 05:51:28 UTC by @QED for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88834
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

FirelightVault can return a positive ERC-4626 `maxMint` quote whose exact share amount cannot fit within the remaining asset capacity, causing compliant integrations to submit reverting mints.

## Vulnerability Details

**Affected target:** `FirelightVault` at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

**Severity:** Low

**Asset:** [`contracts/core/FirelightVault.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol)

### Root Cause

[`maxMint`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L392) computes `convertToShares(depositLimit) - totalSupply()` instead of deriving the executable share amount from remaining asset capacity. The vault's [virtual-asset/share conversion](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1074) is not additive, while [`mint`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L704) ceil-rounds the asset cost and [`_depositFunds`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989) rejects the resulting total when it exceeds `depositLimit`.

### Attack Path

{% stepper %}
{% step %}

## Deposit one asset

A depositor deposits one asset and receives one share.
{% endstep %}

{% step %}

## Transfer an additional asset to the vault

Any underlying holder directly transfers one more asset to the vault, changing the state to one share backed by two assets under a three-asset limit.
{% endstep %}

{% step %}

## Read the maximum mint quote

An ERC-4626 integration reads `maxMint(receiver) = 1`.
{% endstep %}

{% step %}

## Preview the mint

`previewMint(1)` ceil-rounds the cost to two assets, although only one asset of capacity remains.
{% endstep %}

{% step %}

## Attempt to mint

The integration calls `mint(1, receiver)`, and the cap check reverts with `DepositLimitExceeded`.
{% endstep %}
{% endstepper %}

## Impact Details

ERC-4626 routers, allocators, frontends, and batched transactions can suffer reliable availability failures when they mint the advertised maximum after exchange-rate changes, but the defect does not enable theft or bypass the deposit limit.

## References

* [`FirelightVault.maxMint`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L380-L393)
* [`FirelightVault.mint`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L676-L707)
* [`FirelightVault._depositFunds`](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989)
* [`FirelightVault` conversion formulas](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1074)
* [ERC-4626 specification](https://eips.ethereum.org/EIPS/eip-4626)

## Recommended Fix

Compute `maxMint` from remaining asset capacity with `mint`-equivalent rounding and enforce with regression tests that `previewMint(maxMint(receiver)) <= maxDeposit(receiver)` across exchange-rate changes.

## Link to Proof of Concept

<https://gist.github.com/w-qedaudit/b8c9f3d72bf1c47f73955da7adaf1ed2>

## Proof of Concept

**PoC gist:** <https://gist.github.com/w-qedaudit/b8c9f3d72bf1c47f73955da7adaf1ed2>

Prerequisite: Git, Node.js, and npm must be installed with network access available for cloning and dependency installation.

The supplied `poc.js` is a temporary focused reproducer copied into the checked-out repository at runtime; it is not an in-tree project test or modification.

```sh
cd firelight/igtok1p3-maxmint-unexecutable
chmod +x run.sh
./run.sh
```

Expected successful output includes:

```
STATE totalSupply=1 totalAssets=2 depositLimit=3
QUOTE maxDeposit=1 maxMint=1 previewMint(maxMint)=2
RESULT mint(maxMint) reverted error=DepositLimitExceeded
POST totalSupply=1 totalAssets=2
```

The complete output from the verified execution is included in `poc_output.txt`.
