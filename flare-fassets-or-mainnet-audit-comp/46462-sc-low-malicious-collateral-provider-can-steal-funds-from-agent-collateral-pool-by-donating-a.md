# #46462 \[SC-Low] Malicious collateral provider can steal funds from agent collateral pool by donating a large amount of native token to the pool (inflation attack)

**Submitted on May 31st 2025 at 01:17:30 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46462
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A malicious collateral provider (attacker) can front-run other providers, depositing small amount of native tokens (will only mention as FLR from now on) in returns for pool collateral token (PCT). Normally, 1 FLR in --> \~1 PCT out. Then attacker donates a large quantity of FLR into the pool to inflate FLR/PCT ratio of the pool (inflation attack). After that, if a provider deposits quantity of FLR less than inflated FLR/PCT ratio, he will receive nothing (0 PCT).

The donation value must be less than 1% of the total pool collateral (TPC) in one Tx. But attacker can send > 100 Txs in the same block to double TPC. In the next block, attacker can double the amount of his donation to double TPC again. After just 10 blocks, attacker can inflate the quantity of FLR (recorded as WFLR, but I will refer it as FLR from now on for simplicity) by 2^10 = 1.024 times.

When the quantity of FLR is 1000 times higher than the quantity of PCT in a pool, any provider who deposits less than 1000 FLR at a time will receive 0 PCT (due to rouding down effect). See more about inflation attack here <https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks>

## Vulnerability Details

In donateNat() function, donation value is only required to be less than 1% total pool collateral. There's no check to limit the ratio between quantity of FLR and PCT. That allows an attacker to call this function many times to inflate the quantity of FLR in pool as much as he wants. <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L855C5-L862C6>

```
function donateNat()
        external payable
    {
        require(msg.value >= MIN_NAT_TO_ENTER && msg.value < totalCollateral / 100,
            "donation must be between 1 NAT and 1% of the total pool collateral");
        _depositWNat();
        emit Donated(msg.sender, msg.value);
    }
```

The quantity of PCT collateral providers receiving is calculated by this function: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L485>. Basically:

```
Quantity of minted PCT = Quantity of deposited FLR x 
    Quantity of current total PCT / Quantity of total collateral (FLR) in the pool
```

Recommendation: add a check to restrict the ratio between quantity of FLR (after adding the donation value) and PCT to be less than minimum deposit quantity. Increase the minimum deposit quantity to be higher value (current value: only 1 Nat)

## Impact Details

* Attacker: pool collateral provider
* Victim: other pool collateral providers (possibly including agent)
* Impact: stealing collateral

## References

<https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks>

## Proof of Concept

## Proof of Concept

1. Attacker (Alice) identifies a new agent collateral pool with 0 (ideally) or very limited collateral.
2. Alice deposits 101 FLR to receive 101 PCT
3. Alice calls donateNat() function 101 times within the same block to donate 101 FLR in total to the pool
4. Alice repeats step 3 for 10 times with increasingly larger FLR quantity, such as: 202, 404,..., 101x2^9 FLR. After that the ratio between quantity of FLR in the pool and quantity of PCT is inflated to \~1000
5. Bob deposits 999 FLR. He receives 0 PCT because 999/1000 is rounded down to 0
