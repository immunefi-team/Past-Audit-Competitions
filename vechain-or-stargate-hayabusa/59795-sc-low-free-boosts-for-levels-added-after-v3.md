# 59795 sc low free boosts for levels added after v3

**Submitted on Nov 15th 2025 at 20:36:37 UTC by @OxPrince for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59795
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Starting with V3 the boost fee per level is configured solely in `initializeV3`, which is a one-shot `reinitializer(3)` (packages/contracts/contracts/StargateNFT/StargateNFT.sol:226-243). The only remaining admin entry point for level management is `addLevel` (packages/contracts/contracts/StargateNFT/StargateNFT.sol:301-305), but this helper never touches `boostPricePerBlock`. Because `boostPricePerBlock` is stored in a mapping with the Solidity default of zero and `Levels.updateLevelBoostPricePerBlock` is no longer reachable from any callable function, every level that is added after `initializeV3` permanently inherits a boost price of zero.

`MintingLogic` calculates the boost fee as `(maturityPeriodEndBlock - now) * boostPricePerBlock` (packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol:316-329). When `boostPricePerBlock` is zero, `MintingLogic.boostOnBehalfOf` accepts a zero-fee boost: the balance and allowance checks trivially pass, and the VTHO transfer burns zero tokens (packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol:102-145). Therefore any wallet can immediately mature newly created levels without paying VTHO, bypassing both the intended maturity delay and all boost revenue.

## Vulnerability Details

{% stepper %}
{% step %}

### Governance can add levels that have no boost price

Governance (or any wallet with `LEVEL_OPERATOR_ROLE`) can add new levels by calling `StargateNFT.addLevel` (packages/contracts/contracts/StargateNFT/StargateNFT.sol:301-305). The function delegates to `Levels.addLevel`, which validates metadata/caps and appends the level, but makes no mention of boost pricing.
{% endstep %}

{% step %}

### Boost price is only set during V3 initialization

The only place where `Levels.updateLevelBoostPricePerBlock` is invoked is inside `initializeV3` (packages/contracts/contracts/StargateNFT/StargateNFT.sol:226-243). Because the initializer is marked `reinitializer(3)`, it can run exactly once for the lifetime of the proxy. After deployment, there is no callable function that writes to `boostPricePerBlock[_levelId]`.
{% endstep %}

{% step %}

### Newly added levels keep the default zero boost price

For every newly added level `_levelId`, the mapping entry `boostPricePerBlock[_levelId]` stays at the default `0`. Both `boostPricePerBlock` and `boostAmountOfLevel` now report 0 for that level (packages/contracts/contracts/StargateNFT/StargateNFT.sol:403-411).
{% endstep %}

{% step %}

### Users can boost for free, skipping maturity and revenue

When a user owns an NFT of that level and calls `boost`, the call reaches `MintingLogic.boostOnBehalfOf` through `StargateNFT.boost` (packages/contracts/contracts/StargateNFT/StargateNFT.sol:392-396). `_boostAmount` multiplies the remaining maturity blocks by the zero boost price, so `requiredBoostAmount` is zero (packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol:316-329). The subsequent balance/allowance checks and the burn transfer all succeed without transferring VTHO (packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol:112-145). The NFT’s maturity end block is nevertheless set to the current block, fully skipping the maturity period for free.
{% endstep %}
{% endstepper %}

## Impact Details

{% hint style="warning" %}

* Loss of expected boost-fee revenue for every newly added level.
* Users that watch governance for new level announcements can instantly mature fresh high-value positions, skipping the intended lockup and any reputation/time gating enforced by Stargate.
* The invariants around “time to maturity before delegating” become unenforceable for affected levels, which undermines fairness toward existing delegators.
  {% endhint %}

## Proof of Concept

Boost.test

{% code title="Boost.test (excerpt)" %}

```rust
describe("Boost configuration gaps", () => {
     it("allows boosting a newly added level without paying VTHO", async () => {
         const newLevelArgs = {
             level: {
                 id: 0,
                 name: "FreeBoost",
                 isX: false,
                 vetAmountRequiredToStake: ethers.parseEther("1"),
                 scaledRewardFactor: 100,
                 maturityBlocks: 50,
             },
             circulatingSupply: 0,
             cap: 100,
         };

         await stargateNFTContract.addLevel(newLevelArgs);
         const levelIds = await stargateNFTContract.getLevelIds();
         const newLevelId = levelIds[levelIds.length - 1];
         expect(await stargateNFTContract.boostPricePerBlock(newLevelId)).to.equal(0n);

         await stargateNFTContract.mint(newLevelId, user.address);
         const tokenId = await stargateNFTContract.getCurrentTokenId();
         expect(await stargateNFTContract.boostAmount(tokenId)).to.equal(0n);
         expect(await vthoTokenContract.balanceOf(user.address)).to.equal(0n);

         await stargateNFTContract.connect(user).boost(tokenId);

         const maturityPeriodEndBlock =
             await stargateNFTContract.maturityPeriodEndBlock(tokenId);
         const currentBlock = await stargateContract.clock();
         expect(maturityPeriodEndBlock).to.equal(currentBlock);
         expect(await vthoTokenContract.balanceOf(user.address)).to.equal(0n);
     });
});
```

{% endcode %}

## References

* Target contract: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* Minting logic: packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol
