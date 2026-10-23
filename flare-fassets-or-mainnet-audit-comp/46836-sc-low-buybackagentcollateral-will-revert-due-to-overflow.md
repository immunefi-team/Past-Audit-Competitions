# #46836 \[SC-Low] buybackAgentCollateral will revert due to overflow

**Submitted on Jun 5th 2025 at 06:01:52 UTC by @rick137 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46836
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/AgentCollateralFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When f-asset is terminated, agent can burn the market price of backed f-assets with his collateral, to release the remaining collateral, whereas `AgentCollateralFacet::buybackAgentCollateral` will revert due to arithmetic operation overflowed, leading to permanently freezing funds

## Vulnerability Details

```text!
0- agent is created by owner
1- token collaterals and NAT collaterals is deposited by owner and agent is made available
2- f-assets is minted by a minter
3- agent's address will be added to allowed list by governance
4- a request will be submitted by agent's owner to return from CV
5- payment instructions will be emitted by triggering account, which means agent's request cannot be cancel
6- agent will be removed from allowed list , which means payments related to this agent will be skipped by CV multisig
7- minting will be paused by governance
8- f-asset will be terminated by governance after 30 days
9- `buybackAgentCollateral` will revert due to overflow

```

## Impact Details

Permanently freezing funds

## Proof of Concept

## Proof of Concept

consider to add this test to `14-coreVault.ts`

```typescript
     it.only("buybackAgentCollateral will revert due to overflow", async() => {
        const usdcCollateral = context.collaterals.find(c => c.token === context.usdc.address)!;
        const poolCollateral = context.collaterals.find(c => c.token === context.wNat.address)!;
        // 0- agent is created by owner
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1, {
            vaultCollateralToken: usdcCollateral.token,
            mintingVaultCollateralRatioBIPS: usdcCollateral.minCollateralRatioBIPS,
            mintingPoolCollateralRatioBIPS: poolCollateral.minCollateralRatioBIPS,
        });
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(100000000000000));

        // 1- token collaterals and NAT collaterals is deposited by owner and agent is made available
        await agent.depositCollateralLotsAndMakeAvailable(10);

        // 2- f-assets is minted by minter
        const [minted] = await minter.performMinting(agent.vaultAddress, 1);

        // 3-agent's address is added to allowed list by governance
        await coreVaultManager.addAllowedDestinationAddresses([agent.underlyingAddress], { from: governance });


        await prefundCoreVault(minter.underlyingAddress, 10000000e6);

        let agentInfo = await agent.getAgentInfo();

        // 4- a request is submitted by agent's owner to return from CV
        const rres = await context.assetManager.requestReturnFromCoreVault(agent.vaultAddress, agentInfo.freeCollateralLots, { from: agent.ownerWorkAddress });

        // 5- payment instructions is emitted by triggering account, which means agent's request cannot be cancel
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });

        // 6- agent is removed from allowed list , which means payments related to this agent will be skipped by CV multisig
        const tx = await coreVaultManager.removeAllowedDestinationAddresses([agent.underlyingAddress], {
        from: governance,
        });
        // 7- minting is paused by governance
        await context.assetManagerController.pauseMinting([context.assetManager.address], { from: governance });
        assert.isTrue(await context.assetManager.mintingPaused());
        await deterministicTimeIncrease(30 * DAYS);
        // 8- f-asset is terminated by governance after 30 days
        await context.assetManagerController.terminate([context.assetManager.address], {from : governance});

        // 9- `buybackAgentCollateral` will revert due to overflow
        const [,, buybackCost] = await agent.getBuybackAgentCollateralValue();
        await expectRevert(context.assetManager.buybackAgentCollateral(agent.vaultAddress, {from: agent.ownerWorkAddress, value: buybackCost}), "Arithmetic operation overflowed outside of an unchecked block");
    });
```
