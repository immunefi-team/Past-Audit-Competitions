# #46081 \[SC-Medium] Wrong check in \`redeemFromCoreVault\` will result in unnecessary revert

**Submitted on May 24th 2025 at 15:35:31 UTC by @aman for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46081
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour
  * Temporary freezing of funds for at least 24 hour
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The CoreVault redemption process checks available lots before fee deduction, which can cause unnecessary reverts. When a user requests redemption, the system checks if there are enough available lots in CoreVault before deducting the redemption fee. This is inefficient because after fee deduction, the actual amount needed would be less, potentially allowing the redemption to proceed. This leads to unnecessary redemption failures.

## Vulnerability Details

The issue occurs in the `redeemFromCoreVault` function where the available lots check happens before fee deduction:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
) internal {
    // Check available lots before fee deduction
    uint64 availableLots = getCoreVaultAmountLots();
@>    require(_lots <= availableLots, "not enough available on core vault"); // @audit: premature check

    // ... other checks ...

    // Fee deduction happens after the check
    uint256 redeemedUBA = Conversion.convertAmgToUBA(redeemedAMG);
    uint256 redemptionFeeUBA = redeemedUBA.mulBips(state.redemptionFeeBIPS);
    uint128 paymentUBA = (redeemedUBA - redemptionFeeUBA).toUint128();

    // Actual transfer request uses reduced amount
    paymentReference = state.coreVaultManager.requestTransferFromCoreVault(
        _redeemerUnderlyingAddress, paymentReference, paymentUBA, false);
}
```

The problem is that the system:

1. Checks available lots against full redemption amount
2. Deducts fee after the check

## Impact Details

The impact is significant because:

1. Users cannot redeem when they should be able to
2. Unnecessary transaction failures

## References

`CoreVault.sol::redeemFromCoreVault`

## Mitigation :

The Best Fix would be to apply the following changes:

```diff
diff --git a/contracts/assetManager/library/CoreVault.sol b/contracts/assetManager/library/CoreVault.sol
index 89cb6188..b9e440e9 100644
--- a/contracts/assetManager/library/CoreVault.sol
+++ b/contracts/assetManager/library/CoreVault.sol
@@ -218,16 +218,17 @@ library CoreVault {
             "underlying address not allowed by core vault");
         AssetManagerSettings.Data storage settings = Globals.getSettings();
         uint64 availableLots = getCoreVaultAmountLots();
-        require(_lots <= availableLots, "not enough available on core vault");
         uint64 minimumRedeemLots = SafeMath64.min64(state.minimumRedeemLots, availableLots);
         require(_lots >= minimumRedeemLots, "requested amount too small");
         // burn the senders fassets
         uint64 redeemedAMG = _lots * settings.lotSizeAMG;
         uint256 redeemedUBA = Conversion.convertAmgToUBA(redeemedAMG);
-        Redemptions.burnFAssets(msg.sender, redeemedUBA);
-        // subtract the redemption fee
         uint256 redemptionFeeUBA = redeemedUBA.mulBips(state.redemptionFeeBIPS);
+        // subtract the redemption fee
         uint128 paymentUBA = (redeemedUBA - redemptionFeeUBA).toUint128();
+        uint64 _lotsConsumed  = Conversion.convertUBAToAmg(paymentUBA)/settings.lotSizeAMG;
+        require(_lotsConsumed <= availableLots, "not enough available on core vault"); // The correct check would be that after fee deduction check that we have this much fund available
+        Redemptions.burnFAssets(msg.sender, redeemedUBA);
```

## Proof of Concept

## Proof of Concept

Add the Following test case to `CoreVault.ts` and run with command `yarn testHH`

```javascript
it.only("Available lots checking before fee deduction will reslt in unnecessary revert", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(101);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 101);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(100);
        await agent.transferToCoreVault(transferAmount);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);

        const lots = 101;
        // redeemer requests direct redemption from CV
        await expectRevert(context.assetManager.redeemFromCoreVault(lots, redeemer.underlyingAddress, { from: redeemer.address }),
            "not enough available on core vault");
    });
```

The above test case will revert , But after apply the fix than run the 2nd test case it will be executed successfully

```javascript
    it.only("Available lots checking before fee deduction will executed", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(101);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 101);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(100);
        await agent.transferToCoreVault(transferAmount);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);

        const lots = 101;
        // redeemer requests direct redemption from CV
        const [paymentAmount1, paymentReference1] = await testRedeemFromCV(redeemer, lots);
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });
        const paymentReqs = filterEvents(trigRes, "PaymentInstructions");
        assertWeb3Equal(paymentReqs[0].args.account, coreVaultUnderlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.destination, redeemer.underlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.amount, paymentAmount1);
        agentInfo1 = await agent.getAgentInfo();
        console.log("minted1 | redeeming1");
        console.log(agentInfo1.mintedUBA, agentInfo1.redeemingUBA);
        const { 0: immediatelyAvailable, 1: totalAvailable } = await context.assetManager.coreVaultAvailableAmount();
        console.log("totalAvailable" , totalAvailable.toString())
        let AvailableLots = Number(context.convertUBAToAmg(totalAvailable)) / Number(context.settings.lotSizeAMG);
        console.log("AvailableLots" , AvailableLots)

    });
```
