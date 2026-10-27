# #47380 \[SC-Insight] Incorrect token\_assets\_value in AccountLiquidated Event

**Submitted on Jun 13th 2025 at 06:49:47 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47380
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/registry>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `account_liquidate()` function in the `account_component.cairo` file contains a flaw where the `token_assets_value` field in the emitted `AccountLiquidated` event is incorrectly set to the **liquidator’s** initial token balance rather than any value related to the **liquidated account**. This misrepresentation undermines the integrity of on-chain event data and misleads off-chain consumers such as analytics tools, dashboards, and auditors.

## Vulnerability Details

In the `account_liquidate()` function, the protocol emits an `AccountLiquidated` event after transferring a share of the liquidated account’s positions and collateral. However, the `token_assets_value` field is populated using the liquidator's **pre-liquidation** balance, which is not relevant to the liquidation outcome and contradicts the field’s intended meaning.

The event documentation clearly states that `token_assets_value` refers to "The total value of token assets in the account" (i.e., the **liquidated account**). Logging the liquidator’s balance at this field is logically incorrect, misleading, and potentially harmful for off-chain systems relying on accurate data.

```rust
let initial_liquidator_token_balance = self
    .token
    .get_asset_balance_or_empty(liquidator, settlement_token_asset);

...

self.emit(
    AccountComponent::Event::AccountLiquidated(
        AccountLiquidated {
            account: account,
            liquidator: liquidator,
            token_assets_value: initial_liquidator_token_balance.amount,  // ← Incorrect source
            margin_requirement: margin_requirement.into(),
            unrealized_pnl: total_unrealized_pnl.into(),
            liquidation_penalty: liq_penalty,
            liquidation_share: liquidation_share_d.try_into().unwrap(),
            is_partial_liquidation: (!full_liquidation).into(),
            oracle_snapshot_id: oracle_snapshot_id,
        },
    ),
);
```

## Impact Details

Off-chain systems monitoring liquidation value per account will retrieve incorrect data.

## Proof of Concept

## Proof of Concept

This is no need POC
