# 69420 sc insight avoid the use of floating pragma to ensure same compiler version used for testing is also used for deployment

Submitted on Mar 14th 2026 at 20:44:55 UTC by @kaysoft for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69420
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

### Brief/Intro

The `Staking.sol`, `IMigrator.sol` and `IStakingV1` all use unspecific solidity pragma `^0.8.23`.

This may result in a situation where the pragma version used for test is different from the one used for deployment because the `^0.8.23` means any compiler from `0.8.23` and above.

### Vulnerability Details

The smart contracts use floating compiler version and this may result in using a different version of compiler during deployment. It can also allow the use of latest compiler version that have unfixed issues to be used.

```solidity
File: Staking.sol
pragma solidity ^0.8.23;
```

### Impact Details

Complier used during deployment may be different from the one used during testing which may lead to other security issues.\
Newer compiler versions that have not been battle tested may be used to compile the code during deployment since the compiler version is not fixed.

## Recommendation

Instead of `pragma solidity ^0.8.23;`, use `pragma solidity 0.8.23`
