# #46771 \[SC-Insight] Incorrect Collateral Ratio Check Due to Rounding Error

**Submitted on Jun 4th 2025 at 13:34:13 UTC by @TheCarrot for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46771
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_staysAboveCR` function checks whether withdrawing collateral (`_withdrawnNat`) keeps the pool's collateral ratio (CR) above a threshold (`_crBIPS`). However, due to integer division rounding errors, the check can incorrectly return true even when the CR is slightly below the required threshold. This could allow users to exit the pool while leaving it undercollateralized.

## Vulnerability Details

The current implementation checks:

```solidity
return (_assetData.poolNatBalance - _withdrawnNat) * _assetData.assetPriceDiv >=  
       (_assetData.agentBackedFAsset * _assetData.assetPriceMul).mulBips(_crBIPS);
```

Problem:

If `(_assetData.poolNatBalance - _withdrawnNat) * _assetData.assetPriceDiv` is just below the required value (due to rounding), the function still returns true.

* Example:
  * Suppose:
    * `poolNatBalance = 1000`
    * `_withdrawnNat = 500`
    * `assetPriceDiv = 1`
    * `agentBackedFAsset * assetPriceMul * crBIPS = 499` (just below `500`)
  * The check `500 >= 499` passes, even though the pool is undercollateralized by 1 wei.

Fix: Replace `>=` with `>` to add a 1-wei buffer:

```solidity
return (_assetData.poolNatBalance - _withdrawnNat) * _assetData.assetPriceDiv >  
       (_assetData.agentBackedFAsset * _assetData.assetPriceMul).mulBips(_crBIPS);
```

## Impact Details

Impact:

* Users could withdraw slightly more collateral than allowed, gradually pushing the pool into undercollateralization.
* If exploited repeatedly, the pool could become insolvent, causing losses for remaining LPs.

Exploit Scenario:

1. An attacker monitors the pool’s CR.
2. When CR is near exitCR, they trigger small exits that round favorably, draining collateral.
3. The pool eventually becomes undercollateralized, risking bad debt accumulation.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L602>

## Proof of Concept

## Proof of Concept

Attack Scenario: Gradually Draining Pool Collateral via Rounding Exploit Assumptions:

The pool has a collateral ratio (CR) very close to exitCR (e.g., exitCR = 150%, current CR = 150.0001%).

The attacker holds pool tokens and wants to exit while maximizing collateral extraction.

Step-by-Step Attack Path

Step 1: Monitor Pool State

* The attacker tracks the pool’s real-time collateral ratio using:

```solidity
CR = (poolNatBalance * assetPriceDiv) / (agentBackedFAsset * assetPriceMul)
```

* Waits until CR ≈ exitCR + ε (where ε is a tiny buffer, e.g., 1 wei).

Step 2: Craft a Withdrawal Just Below exitCR

* The attacker calculates the maximum \_withdrawnNat such that:

```solidity
(poolNatBalance - _withdrawnNat) * assetPriceDiv == (agentBackedFAsset * assetPriceMul * exitCR) - 1
```

* Due to integer division rounding, the check:

```solidity
(poolNatBalance - _withdrawnNat) * assetPriceDiv >= (agentBackedFAsset * assetPriceMul * exitCR)
```

incorrectly returns true (because 499 >= 499 is true, even though CR is technically below exitCR).

Step 3: Execute Exit Transaction

* The attacker calls:

```solidity
exit(_tokenShare, TokenExitType.MAXIMIZE_FEE_WITHDRAWAL)
```

* The function withdraws \_withdrawnNat while the pool’s true CR is now slightly below exitCR.

Step 4: Repeat to Drain Collateral

* The attacker repeats Steps 1–3 in small increments:
  * Each withdrawal skims off 1–2 wei below exitCR.
  * Over time, the pool’s CR degrades further, risking undercollateralization.

Step 5: Trigger Insolvency

* Once the pool’s CR drops significantly below exitCR, other LPs panic and exit.
* The pool cannot cover all redemptions, leading to bad debt or liquidation.
