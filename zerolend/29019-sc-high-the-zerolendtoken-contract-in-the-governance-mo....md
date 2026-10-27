# 29019 - \[SC - High] The ZeroLendToken contract in the Governance mo...

Submitted on Mar 4th 2024 at 20:56:25 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29019

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The ZeroLendToken contract in the Governance module mishandles the whitelist. It is treated as a blacklist.

## Vulnerability Details

The code below handles updating of the balance when a token is sent by `from` to `to`.

```
function _update(
    address from,
    address to,
    uint256 value
) internal virtual override {
    require(!blacklisted[from] && !blacklisted[to], "blacklisted");
    require(!paused && !whitelisted[from], "paused");
    super._update(from, to, value);
}
```

Note that if `whitelisted` is True, the token should allow the transfer. However the condition is flipped, so it will certainly abort the transfer.

## Impact Details

A whitelisted user will not be able to transfer their tokens, resulting in a temporary freezing of funds.

## Recommended remidiation

Consider refactoring the code as suggestedbelow: `require(!paused || whitelisted[from])`

## References

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/ZeroLendToken.sol#L61>

## Proof of Concept

1. User A is inserted to the whitelist
2. User A wishes to transfer their tokens to User B
3. The transfer() reverts, resulting in lock of funds.
