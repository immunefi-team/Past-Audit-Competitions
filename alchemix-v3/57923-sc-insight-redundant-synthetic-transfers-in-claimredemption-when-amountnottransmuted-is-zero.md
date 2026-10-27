# 57923 sc insight redundant synthetic transfers in claimredemption when amountnottransmuted is zero

**Submitted on Oct 29th 2025 at 13:34:47 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57923
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**

## Description

## Brief/Intro

When a claim is redeemed after the transmutationTime has fully elapsed, only yield tokens are transferred to the user.

However, the contract still performs synthetic token calculations and transfers, even though the untransmuted amount is zero. This results in unnecessary arithmetic and transfer calls, wasting gas.

## Vulnerability Details

If `amountNottransmuted == 0` (the claim is fully matured), the following code becomes redundant:

```
uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
uint256 syntheticReturned = amountNottransmuted - syntheticFee;

TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);
```

A check `if (amountNottransmuted > 0)` would prevent these unnecessary operations.

## Impact Details

When `amountNottransmuted == 0`, the function still performs two redundant transfer calls, each costing approximately 20,000–45,000 gas without transferring any actual tokens.

## Proof of Concept

## Proof of Concept

1. User calls create redemption with 100e18
2. User calls claimRedemption ( this does 2 unneeded transfer operations)

POC:

Run the existing test `test_target_ClaimRedemption` with the -vvvv flag. In the logs, you can see the redundant synthetic token transfer calls being executed even when amountNottransmuted == 0.

```
    │   ├─ [0] console::log("syntheticReturned", 0) [staticcall]
    │   │   └─ ← [Stop]
    │   ├─ [6095] AlEth::transfer(0x000000000000000000000000000000000000bEEF, 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: 0x000000000000000000000000000000000000bEEF, value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [8095] AlEth::transfer(TransmuterTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0)
    │   │   ├─ emit Transfer(from: Transmuter: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: TransmuterTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 0)
```
