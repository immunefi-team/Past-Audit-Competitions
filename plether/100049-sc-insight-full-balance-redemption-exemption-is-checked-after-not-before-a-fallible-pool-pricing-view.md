# \#100049 [SC-Insight] Full-balance redemption exemption is checked after, not before, a fallible Pool pricing view

**Submitted on Nov 16th 2026 at 06:42:28 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100049
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

# Description

`TrancheVault.requestRedeem` first computes the owner's `maxRequestRedeem` and
then calls `_requireMinimumRedeemSize(shares, maxShares)`. The minimum-size
condition is currently ordered as:

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

The intended full-balance exception is `shares == maxShares`. However, Solidity
evaluates the left operand first, so even an exempt full-balance request calls
`estimateRedeemAssets`. That estimate calls `totalAssets`, which reaches
`HousePool.getPendingTrancheState` and the Pool's Engine/Terminal-NAV view
dependencies. If that external view graph reverts, the full-balance request also
reverts even though its estimated size is irrelevant to the exemption.

The full-balance exemption is a local value condition and should be decided
before consulting fallible Pool-pricing dependencies. Reordering the operands
isolates an exempt full-balance redemption from a pricing-view failure.
Partial redemptions are unchanged: they still call `estimateRedeemAssets`,
compare the result with `POOL.minTrancheDepositAssets()`, and revert when the
estimated value is below the minimum.

## References

All source paths are under `packages/perps/` at commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `src/TrancheVault.sol:606-632` — `requestRedeem`
- `src/TrancheVault.sol:510-513` — `totalAssets`
- `src/TrancheVault.sol:1385-1390` — `estimateRedeemAssets`
- `src/TrancheVault.sol:1611-1618` — minimum-size condition and operand order
- `src/HousePool.sol:1285-1323` — pending-tranche pricing view
- `src/HousePool.sol:1836-1850` — Engine-protocol-lens snapshot dependencies


## Proof of Concept

# Proof of Concept

## Conditions under which this Insight is valuable: 

A full-balance redemption reaches `_requireMinimumRedeemSize`. Whether it is exempt is already known from local values (`shares == maxShares`), so the function should resolve that exemption before consulting the Pool's fallible pricing view. Partial redemptions continue through the same estimate and minimum-value check.

Read-only source demonstration:

1. `TrancheVault.sol:606-624` shows `requestRedeem` computing `maxShares` and calling `_requireMinimumRedeemSize` before transferring shares or enqueuing the request.
2. `TrancheVault.sol:1611-1618` shows `estimateRedeemAssets(shares) < minimum && shares < maxShares`; the external estimate is therefore evaluated before the local full-balance exemption.
3. `TrancheVault.sol:1385-1390` and `:510-513` show the estimate reaching
   `POOL.getPendingTrancheState()`.

Retained differential demonstration:

- On frozen source, forcing `getPendingTrancheState` to revert makes an
  otherwise valid full-balance request bubble that failure.
- Reordering the condition to
  `shares < maxShares && estimateRedeemAssets(shares) < minimum` makes that full-balance request bypass the view and proceed.
- The negative control preserves partial-redemption behavior: a partial
  request still propagates the view failure, a full-balance dust request
  succeeds, and a below-minimum partial request still reverts
  `WithdrawalTooSmall`.

This differential establishes the value condition directly: the exemption is
resolved locally for full-balance redemptions, while every partial redemption
retains the existing minimum-value check.

