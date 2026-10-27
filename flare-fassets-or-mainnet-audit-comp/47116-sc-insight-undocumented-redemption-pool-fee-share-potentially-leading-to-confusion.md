# #47116 \[SC-Insight] Undocumented Redemption Pool Fee Share potentially leading to confusion

**Submitted on Jun 9th 2025 at 03:04:13 UTC by @a090325 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47116
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Agents.sol>
* **Impacts:**

## Description

## Brief/Intro

It's possible for agents to set/update Redemption Pool Fee Share. This feature is implemented in several files (listed in "references" section). But the docs (<https://dev.flare.network/fassets/redemption#redemption-fee>) said:

> This fee is meant only to cover the agent's transaction fee on the underlying chain, so it is not shared with the collateral pool.

So there's a clear mismatch between the docs and the implemented feature.

## Vulnerability Details

* This feature is exposed through AgentSettingsFacet (<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/AgentSettingsFacet.sol>) but ultimately this function will be called if an agent decide to set/update Redemption Pool Fee Share: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L67>

```
 function setRedemptionPoolFeeShareBIPS(
        Agent.State storage _agent,
        uint256 _redemptionPoolFeeShareBIPS
    )
        internal
    {
        require(_redemptionPoolFeeShareBIPS <= SafePct.MAX_BIPS, "value too high");
        _agent.redemptionPoolFeeShareBIPS = _redemptionPoolFeeShareBIPS.toUint16();
    }
```

* Besides above functions, there're several other related functions as well. Digging into those functions, I conclude that this (Redemption Pool Fee Share) is a fully implemented feature.

## Impact Details

This undocumented feature could cause confusion to:

* Agents: could use this feature the wrong way.
* Auditors: could spend time on an unused (but implemented) feature.

## References

* <https://dev.flare.network/fassets/redemption#redemption-fee>
* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/AgentSettingsUpdater.sol#L15>
* <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L67> <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L356>

## Proof of Concept

## Proof of Concept

Please check those files (listed in references) and also the docs (<https://dev.flare.network/fassets/redemption#redemption-fee>).
