# #45830 \[SC-Medium] Incorrect amount passed to checkMintingCap in self-minting allows bypassing of config minting cap

**Submitted on May 21st 2025 at 04:28:42 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45830
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Vulnerability Details

To enforce system-wide constraints on fAsset issuance (mintingCap), the `checkMintingCap` function validates that the total minted amount (including reservations) does not exceed a configurable minting cap.

```solidity
function checkMintingCap(
    uint64 _increaseAMG
)
    internal view
{
    AssetManagerState.State storage state = AssetManagerState.get();
    AssetManagerSettings.Data storage settings = Globals.getSettings();
    uint256 mintingCapAMG = settings.mintingCapAMG;
    if (mintingCapAMG == 0) return;     // minting cap disabled
    uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
    uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
    require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded");
}
```

When collateral is reserved, this cap is enforced correctly by accounting for both the reservation amount and the corresponding pool fee:

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol#L51>

```solidity
_reserveCollateral(agent, valueAMG + _currentPoolFeeAMG(agent, valueAMG));
```

However, during self-minting operations, specifically in `selfMint` and `mintFromFreeUnderlying`, only the base value (`valueAMG`) is passed to `checkMintingCap`, while the additional fAssets minted as pool fee are excluded:

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol#L89-L92>

```solidity
uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
checkMintingCap(valueAMG);
...
uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
```

**This allows the actual fAsset supply to exceed the cap.**

### Example scenario

Assume the following conditions:

* Minting cap is set to 10 (in AMG units)
* Agent configures the maximum allowed agent and pool fee (100%)
* Lot size is 1 AMG per lot

1. Agent performs a self-mint of 10 lots, triggering minting of:
   * 10 fAssets for the agent
   * +10 fAssets as pool fee (100%)
2. Total minted fAssets: **20**
3. `checkMintingCap` is called with `valueAMG = 10`, so the check passes.
4. Actual minted supply (including pool fee): 20 → **cap exceeded**

This allows the cap to be circumvented in self-minting paths.

## Impact

**Security measure bypass**: The minting cap, intended to limit systemic risk, can be exceeded via self-minting.\
**Denial-of-service vector**: A malicious agent with sufficient capital can consume the entire cap buffer, preventing others from reserving collateral or minting.

For example, if the total minted amount from honest agents is 3 and the cap is set at 10, a malicious agent could self-mint 7 lots with a 100% fee—resulting in a total of 14 new fAssets minted (7 to the agent, 7 as pool fee). This would push the total minted supply to 17, exceeding the cap. Even if some agents later redeem their fAssets, the excess remains, effectively locking out further participation.

Finally, since minted fAssets already in circulation, the protocol may be forced to either raise the cap or disable it entirely.

## Recommended Mitigations

* Account for minted pool fee in `selfMinting` and `mintFromFreeUnderlying`

## Proof of Concept

## Proof-of-Concept

The following test demonstrates the described scenario where minting cap can be exceeded via self-minting.

### Steps

1. Add the following test in `test/integration/fasset-simulation/02-MintAndRedeem.ts`

```typescript
    it("nnez - bypass minting cap", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(100000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        // make agent available
        const fullAgentCollateral = toWei(3e8);
        await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
        // mine some blocks to skip the agent creation time
        mockChain.mine(5);
        // update block
        await context.updateUnderlyingBlock();
        await context.assetManager.currentUnderlyingBlock();
        
        // Set a small minting cap
        await context.assetManagerController.setMintingCapAmg([context.assetManager.address], context.convertLotsToAMG(10), { from: governance });
        // Change setting to maximum fee so that 2x of requested amount is minted
        await agent.changeSettings({feeBIPS: 10_000, poolFeeShareBIPS: 10_000});
        await agent.selfMint(context.convertLotsToUBA(20), 10);

        console.log("Minting cap: ", (await context.assetManager.getSettings()).mintingCapAMG);
        console.log("total fAsset in circulation: ", (await context.fAsset.totalSupply()).toString());

    });
```

2. Run `yarn hardhat test "test/integration/fasset-simulation/02-MintAndRedeem.ts" --grep "nnez - bypass minting cap"`
3. Observe that the total fasset (totalSupply) in circulation exceeds the minting cap after self-minting by agent.
