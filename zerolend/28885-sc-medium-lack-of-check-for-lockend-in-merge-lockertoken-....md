# 28885 - \[SC - Medium] Lack of check for Lockend in merge LockerToken ...

Submitted on Feb 29th 2024 at 22:55:06 UTC by @offside0011 for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28885

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

When merging two Locker NFTs, there were no strict restrictions on the start and end times of the two NFTs. This resulted in the final merged NFT having a time range that exceeded the preset maximum time, consequently increasing the locking power.

## Vulnerability Details

In the file BaseLocker.sol, the maximum time that an NFT can be set is externally passed in. For LockerToken, it is set as 4 \* 365 \* 86400, and for LockerLP, it is 365 \* 86400. The staking duration is crucial for the final power calculation. In the \_calculatePower function, the formula is ((lock.end - lock.start) \* lock.amount) / MAXTIME;. Therefore, there are strict restrictions in both the \_createLock and increaseUnlockTime functions to ensure that the time span(lock.end - lock.start) of an NFT does not exceed MAXTIME.

In the merge function, two NFTs can be combined, and the end time is selected from the later time of the two NFTs. In the \_depositFor function, there is no further check on the time span.

Taking LockerLP as an example, suppose an attacker initially stakes an NFT (NFT1) for 1 year, then, after a month, stakes another NFT (NFT2) for 1 year. The attacker then calls the merge function,

```
merge(NFT2, NFT1);
```

In the merge function,

```
uint256 end = _locked0.end >= _locked1.end
    ? _locked0.end
    : _locked1.end;
```

selects the time of the later NFT and assigns it to the second parameter "to", which is NFT1. At this point, the final NFT has a start of NFT1's start and an end of NFT2's end, resulting in a time span of 1 year and 1 month. In the final calculation of \_calculatePower, the returned formula is

```
function _calculatePower(
    LockedBalance memory lock
) internal view returns (uint256) {
     return ((lock.end - lock.start) * lock.amount) / MAXTIME;
}
```

, which amplifies the nft's Power.

## Impact Details

Amplifying the power of a locker in the governance

## References

<https://github.com/zerolend/governance/blob/main/contracts/locker/BaseLocker.sol#L183>

## Proof of Concept (Take LockerLP as Example)

1. call createLockFor, \_lockDuration = 4 \* 365 \* 86400 return nftid=1
2. vm.wrap(block.timestamp + 86400 \* 30)
3. call createLockFor, \_lockDuration = 4 \* 365 \* 86400 return nftid=2
4. merge(2, 1)# 2 is burned and locker(1).end = locker(2).end
