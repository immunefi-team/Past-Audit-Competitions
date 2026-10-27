# #39243 \[SC-Insight] Misleading Comment in merge Function Regarding Token Transfers to wrapped1155Fac

**Submitted on Jan 25th 2025 at 17:32:48 UTC by @huntercheto for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39243
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The merge function in the ConditionalScalarMarket contract contains a misleading comment that suggests only the Long and Short ERC20 tokens are being transferred to the wrapped1155Factory, while it also involves the Invalid, leading to confusion.

## Vulnerability Details

In the merge function, the comment inaccurately states that the contract only transfers Long/Short ERC20 tokens to the wrapped1155Factory, while the Invalid ERC20 tokens are also being transferred. This could cause a misunderstanding of the token flow in the contract. The relevant code is as follows:

```solidity
// Contract transfers Long/Short ERC20 to wrapped1155Factory and gets
// back Long/Short ERC1155.
wrapped1155Factory.unwrap(
    conditionalTokens, wrappedCTData.shortPositionId, amount, address(this), wrappedCTData.shortData
);
wrapped1155Factory.unwrap(
    conditionalTokens, wrappedCTData.longPositionId, amount, address(this), wrappedCTData.longData
);
wrapped1155Factory.unwrap(
    conditionalTokens, wrappedCTData.invalidPositionId, amount, address(this), wrappedCTData.invalidData
);

```

## Impact Details

This is an INSIGHT, as the misleading comment could lead to confusion and misinterpretation of the code.

But this is a CRITICAL issue if the intent was to work only with the Long and Short tokens, as the Invalid tokens are also being transferred, which could lead to unintended behavior.

## Proof of Concept

## Proof of Concept

```solidity
A proof of code is not needed as the issue is straightforward and can be easily identified in the code itself.
```
