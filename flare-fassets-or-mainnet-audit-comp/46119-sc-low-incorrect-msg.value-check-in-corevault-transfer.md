# #46119 \[SC-Low] Incorrect \`msg.Value\` check in \`CoreVault\` Transfer

**Submitted on May 25th 2025 at 06:53:02 UTC by @aman for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46119
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `transferToCoreVault` function in `CoreVault.sol` incorrectly assumes that the `TRANSFER_GAS_ALLOWANCE` should be retained by the contract, when in fact the gas costs are paid by the agent (msg.sender) directly. This leads to unnecessary retention of funds that should be returned to the agent.

## Vulnerability Details

The vulnerable code is in `CoreVault.sol`:

```solidity
if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
}
```

The issue is that the code:

1. Checks if `msg.value` is greater than `transferFeeWei + TRANSFER_GAS_ALLOWANCE`
2. But the `TRANSFER_GAS_ALLOWANCE` is unnecessary because:
   * Gas costs are paid by the agent (msg.sender) directly through the transaction
   * The contract doesn't need to retain any additional gas allowance
   * The only fee that should be retained is the `transferFeeWei`

## Impact Details

* **Severity**: Medium
* **Impact**: Unnecessarily retains funds that should be returned to the agent
* **Likelihood**: High - Affects every transfer to core vault where `msg.value > transferFeeWei` and `msg.value <= transferFeeWei + TRANSFER_GAS_ALLOWANCE`

## References

* File: `contracts/assetManager/library/CoreVault.sol`
* Function: `transferToCoreVault`

## Recommendations

1. Remove the `TRANSFER_GAS_ALLOWANCE` check since gas costs are paid by the msg.sender:

```solidity
if (msg.value > transferFeeWei) {
    Transfers.transferNAT(state.nativeAddress, transferFeeWei);
    Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
}
```

## Proof of Concept

## Proof of Concept

To proof my point I have added few changed to `CoreVault.sol` file :

```diff
+    uint256 balNativeAddressBefore=  payable(address(state.nativeAddress)).balance;
        if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
+        uint256 balNativeAddressAfter=  payable(address(state.nativeAddress)).balance;
+        console.log("state.nativeAddress balance After balance " , balNativeAddressAfter - balNativeAddressBefore - transferFeeWei);
```

And also update the `Agent.ts`:

```diff
diff --git a/test/integration/utils/Agent.ts b/test/integration/utils/Agent.ts
index 02980151..6ae8697a 100644
--- a/test/integration/utils/Agent.ts
+++ b/test/integration/utils/Agent.ts
@@ -529,7 +537,7 @@ export class Agent extends AssetContextClient {

     async transferToCoreVault(transferAmount: BNish) {
         const cbTransferFee = await this.assetManager.transferToCoreVaultFee(transferAmount);
-        const res = await this.assetManager.transferToCoreVault(this.vaultAddress, transferAmount, { from: this.ownerWorkAddress, value: cbTransferFee });
+        const res = await this.assetManager.transferToCoreVault(this.vaultAddress, transferAmount, { from: this.ownerWorkAddress, value: cbTransferFee.add(toBN(100_000)) });
         const rdreqs = filterEvents(res, "RedemptionRequested").map(evt => evt.args);
         assert.isAtLeast(rdreqs.length, 1);
```

And than add following test case to `CoreVault.ts` file:

```javascript
    it.only("extra amount not return correctly", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const agent2 = await Agent.createTest(context, agentOwner2, underlyingAgent2);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        let agentInfo1 = await agent.getAgentInfo();

        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(50);

        // mint
        let [minted] = await minter.performMinting(agent.vaultAddress, 50);


        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(49);
        await agent.transferToCoreVault(transferAmount);
    });
```

Run with command `yarn testHH` console.logs of this test :

```shell
state.nativeAddress balance After balance  100000 # extra amount nativeAddress receives
```
