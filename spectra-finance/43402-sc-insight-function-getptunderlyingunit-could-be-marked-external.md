# #43402 \[SC-Insight] Function \`getPTUnderlyingUnit\` could be marked external

**Submitted on Apr 5th 2025 at 15:48:56 UTC by @PotEater for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43402
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/util/RouterUtil.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `getPTUnderlyingUnit` is marked as public, however it is not used internally anywhere, so it could be marked external to save gas.

## Vulnerability details

Public functions can be called internally and externally at the same time. In such case, marking a function public would make sense. However, the mentioned functions are not used internally within the contract.

This means the functions can be marked external to save gas.

## Impact Details

Functions marked `public` cost more gas than external functions. This is gas/code optimization.

Changing this function to external will save gas.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/util/RouterUtil.sol#L831>

## Proof of Concept

## Proof of Concept

```solidity
    function getPTUnderlyingUnit(address _pt) public view returns (uint256) { // @audit-issue function could be marked external
        return getUnit(IPrincipalToken(_pt).underlying());
    }
```
