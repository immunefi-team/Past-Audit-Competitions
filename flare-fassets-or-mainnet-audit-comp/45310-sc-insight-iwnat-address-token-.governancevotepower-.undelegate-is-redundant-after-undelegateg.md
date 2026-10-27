# #45310 \[SC-Insight] \`IWNat(address(token)).governanceVotePower().undelegate()\` is redundant after \`undelegateGovernance()\`

**Submitted on May 12th 2025 at 13:21:52 UTC by @magtentic for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45310
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**

## Description

## Brief/Intro

The `destroy()` function unnecessarily calls `IWNat(address(token)).governanceVotePower().undelegate()` even when the delegation has already been removed via a prior call to `undelegateGovernance()`. This leads to redundant operations and suboptimal gas usage.

## Vulnerability Details

The flag TOKEN\_DELEGATE\_GOVERNANCE is set to `tokenUseFlags[token]` via `_tokenUsed()` during a call to `delegateGovernance()`, but it is not cleared when `undelegateGovernance()` is invoked. This causes destroy() to interpret the token as still being actively delegated, resulting in a second, redundant call to undelegate().

### Example:

#### Flag is set:

```
function delegateGovernance(IVPToken _token, address _to) external override onlyOwner {
    _token.governanceVotePower().delegate(_to);
    _tokenUsed(_token, TOKEN_DELEGATE_GOVERNANCE);
}
```

#### Flag is not cleared:

```
function undelegateGovernance(IVPToken _token) external override onlyOwner {
    _token.governanceVotePower().undelegate();
}
```

#### Redundant check in `destroy()`:

```
if ((useFlags & TOKEN_DELEGATE_GOVERNANCE) != 0) {
    IWNat(address(token)).governanceVotePower().undelegate();
}
```

## Impact Details

* Redundant logic: `undelegate()` is called twice for the same token, increasing unnecessary gas costs.
* Potential confusion: It creates a mismatch between internal state (tokenUseFlags) and actual delegation state.
* Maintainability issue: Future contributors may misinterpret this logic or fail to reason correctly about token state.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Refer to Vulnerability Details for guidance on why the code is redundant. Runnable PoCs are not required for Smart Contract & Blockchain/DLT bug reports
