# 29069 - \[SC - Medium] Ability to deny users from repaying and supplyi...

Submitted on Mar 6th 2024 at 14:21:15 UTC by @OceanAndThunders for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29069

Report type: Smart Contract

Report severity: Medium

Target: <https://explorer.zksync.io/address/0x54d6F91bE4509826559ad12E1Ca6CA3A6C3811e0>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Hello team,

## Brief/Intro

The repayWithPermit and supplyWithPermit functions on pool contract "0x54d6F91bE4509826559ad12E1Ca6CA3A6C3811e0" uses the permit function for external ERC20 contract, an attacker can front run the permit function , thus the pool.repayWithPermit and pool.supplyWithPermit will automatically reverts, allows a chain call/transaction all be griefed

## Vulnerability Details

The pool contract "0x54d6F91bE4509826559ad12E1Ca6CA3A6C3811e0" allows the user to supply with permit to external contract by providing the permit hash to pool.repayWithPermit, this later will call external token with the given permit bytes, the functions repayWithPermit and supplyWithPermit will revert if the attacker front runs the token.permit with the victim's bytes, the first call will succeed and consumes the nonce, thus the pool.repayWithPermit (the original call) will revert then, the call/transaction will be reverted all the way

## Impact Details

Denying users from using the pool.repayWithPermit and pool.supplyWithPermit properly

## References

Example explanation of the issue : <https://www.trust-security.xyz/post/permission-denied>

## Proof of concept :

In this test "<https://github.com/zerolend/core-contracts/blob/146f7bc50b8b9850cf9048989cd734e51c400b48/test-suites/pool-l2.spec.ts#L382>" include this line at 411 (as the attacker called the permit before the protocols call usdc.permit) :

```
// extracteting v, r, s from the mempool and calling the usdc.permit before pool.repayWithPermit
//the usedc.permit call should succeed, thus it will rsults for the pool.repayWithPermit call failure
await usdc.permit(deployer.address,l2Pool.address,amount.toString(),highDeadline,v,r,s);
```

This will makes the function reverts !

Regards,

Adam
