# 57028 sc insight wrong amount variable in repay event

**Submitted on Oct 22nd 2025 at 19:37:09 UTC by @Anirruth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57028
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The event emitted for repay function is wrong.

## Vulnerability Details

The event emitted when a user calls repay function is the following:

```
emit Repay(msg.sender, amount, recipientTokenId, creditToYield);
```

The following is the event Repay:

```
/// @notice Emitted when `amount` of `underlyingToken` are repaid to grant credit to account owned by `recipientId`.
    ///
    /// @param sender          The address which is repaying tokens.
    /// @param amount          The amount of the underlying token that was used to repay debt.
    /// @param recipientId     The id of account that received credit for the repaid tokens.
    /// @param credit          The amount of debt that was paid-off to the account owned by owner.
    event Repay(address indexed sender, uint256 amount, uint256 indexed recipientId, uint256 credit);
```

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L517-L519>

In the repay function, when emitting the event, in the amount column its supposed to be the credit/creditToYeild variable which gets reduced and not the inputted amount variable, Since the amount variable gets converted or reduced according to the debt and the credit/creditToYield variable is the one which gets subtracted from user. Therefore the variable to be used either the credit or the creditToYield variable. Similar to the burn function : <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L492>

## Impact Details

Wrong amount in the event is emitted.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L543>

## Proof of Concept

## Proof of Concept

Not required for insights.
