# #43971 \[SC-Insight] Incorrect NatSpec Tag in removeRateOracle() Misrepresents Function Reference

**Submitted on Apr 15th 2025 at 06:24:49 UTC by @Glitch\_Hunter for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43971
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/RateOracleRegistry.sol>
* **Impacts:**

## Description

## Brief/Intro

The removeRateOracle() function in RateOracleRegistry.sol includes a NatSpec comment that incorrectly references IRateOracleRegistry-addRateOracle. This can mislead readers or automated tools, such as doc generators and IDEs, when tracing inheritance or interface implementation, resulting in misdocumentation or confusion.

## Vulnerability Details

The function in question contains this NatSpec tag:

```solidity
/** @dev See {IRateOracleRegistry-addRateOracle}. */
function removeRateOracle(address _pt, address _rateOracle) external override restricted {
```

However, this tag incorrectly refers to the addRateOracle function. It should point to the corresponding function in the interface:

```solidity
/** @dev See {IRateOracleRegistry-removeRateOracle}. */
```

This is particularly important for automated documentation tooling, interface verification, and clarity when reviewing inherited functionality or permissions in AccessManagedUpgradeable contracts.

## Impact Details

While this issue does not directly affect contract execution or security, it falls under documentation improvements that impact:

* Developer experience
* Audit trail clarity
* IDE integration and NatSpec tracing
* Public documentation consistency

Mistakenly referencing the wrong function misrepresents what behavior is inherited or overridden and may confuse future maintainers or auditors

## References

None

## Proof of Concept

## Proof of Concept

RateOracleRegistry.sol

```diff
--- /** @dev See {IRateOracleRegistry-addRateOracle}. */  // ❌ Incorrect reference
+++ /** @dev See {IRateOracleRegistry-removeRateOracle}. */  // ✅ Correct reference
function removeRateOracle(address _pt, address _rateOracle) external override restricted {
```
