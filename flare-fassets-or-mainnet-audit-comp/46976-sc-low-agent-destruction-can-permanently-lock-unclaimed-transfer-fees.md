# #46976 \[SC-Low] Agent Destruction Can Permanently Lock Unclaimed Transfer Fees

**Submitted on Jun 7th 2025 at 07:06:56 UTC by @Bluedragon for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46976
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/TransferFeeFacet.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Summary:

When transfer fees are enabled (transferFeeMillionths > 0), agents accumulate transfer fees that can be claimed via `claimTransferFees`. However, when an agent destroys their vault, the system doesn't enforce collection of these accumulated fees before destruction. Since the `claimTransferFees` function depends on agent vault storage that gets deleted during destruction, any unclaimed transfer fees become permanently locked in the AssetManager contract.

## Vulnerability Details:

The issue stems from the transfer fee system's dependency on agent vault storage and the lack of fee collection enforcement during agent destruction.

Transfer fees are tracked and accumulated in `TransferFeeTracking.sol` , where each agent has associated fee tracking data.

Agents can claim their accumulated fees using `TransferFeeFacet.sol`, which requires the agent vault to exist and uses the vault's storage for calculations.

However, when an agent destroys their vault, there's no mechanism to force collection of outstanding transfer fees before the destruction process completes.

## Impact:

* **Permanent Fee Loss**: Unclaimed transfer fees become permanently inaccessible after agent destruction
* **Fund Lock-up**: Transfer fees accumulate in the AssetManager contract with no recovery mechanism
* **Economic Inefficiency**: Agents lose rightful fee earnings if they forget to claim before destruction
* **System Accounting Issues**: Contract holds more funds than can be distributed

**Scenario (step by step):**

1. **Transfer Fee Accumulation**:
   * Transfer fees are enabled with transferFeeMillionths > 0
   * Agent operates normally, accumulating transfer fees over time
   * Fees are tracked in `TransferFeeTracking.sol`
2. **Agent Decides to Exit**:
   * Agent reduces their position to zero (no minted assets, no collateral backing)
   * Agent announces exit from the system
   * Agent has accumulated transfer fees but doesn't claim them
3. **Destruction Process**:
   * Agent calls destroy function after waiting period
   * System allows destruction without checking for unclaimed transfer fees
   * Agent vault storage is deleted/reset
4. **Permanent Fee Loss**:
   * After destruction, `claimTransferFees` can no longer be called for this agent
   * The function requires agent vault storage which fails after vault destruction
   * Transfer fees remain in the contract but become unclaimable

## Recommended Mitigation:

**Option 1: Enforce Fee Collection Before Destruction**

Add a check in the agent destruction process to ensure all transfer fees are claimed:

```solidity
// In agent destruction logic, add:
(uint256 unclaimedFees,) = transferFeeFacet.agentTransferFeeShare(agentVault, type(uint256).max);
require(unclaimedFees == 0, "must claim transfer fees before destruction");
```

**Option 2: Auto-claim During Destruction** Automatically claim and transfer any outstanding fees to the agent owner during the destruction process.

The most user-friendly approach would be Option 2, automatically claiming fees during destruction to prevent accidental loss.

## Proof of Concept

## Proof of Concept:

1. Add the following test to the `12-TrailingFees` in `test/integration/fasset-simulation/` directory
2. Run the test using `yarn testHH`

```ts
it.only("Agent destruction can lock the unclaimed transfer fees", async () => {
            const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
            const agent2 = await Agent.createTest(context, agentOwner2, underlyingAgent2);
            const minter = await Minter.createTest(context, userAddress1, underlyingUser1, context.lotSize().muln(100));
            const redeemer = await Redeemer.create(context, userAddress2, underlyingUser2);
            const agentInfo = await agent.getAgentInfo();
            await agent.depositCollateralsAndMakeAvailable(toWei(1e8), toWei(1e8));
            await agent2.depositCollateralsAndMakeAvailable(toWei(1e8), toWei(1e8));
            mockChain.mine(10);
            await context.updateUnderlyingBlock();
            const currentEpoch = await assetManager.currentTransferFeeEpoch();
            // perform minting
            const lots = 3;
            const [minted] = await minter.performMinting(agent.vaultAddress, lots);
            const [minted2] = await minter.performMinting(agent2.vaultAddress, lots);
            // transfer and check that fee is accounted
            const transferAmount = context.lotSize().muln(3);
            // transfer exact dest
            const transferFee = await calculateFee(transferAmount, true);
            await minter.transferFAsset(redeemer.address, transferAmount, true);
            const endBalanceM = await fAsset.balanceOf(minter.address);
            const endBalanceR = await fAsset.balanceOf(redeemer.address);
            // skip 1 epoch and claim
            await deterministicTimeIncrease(epochDuration);
            // Redeemer redeems the minted lots
            const [requests] = await redeemer.requestRedemption(lots);
            await agent.performRedemptions(requests);
            const info = await agent.getAgentInfo();
            console.log("==== Minter mints f-assets ====");
            console.log("Agent total Minted UBA: ", info.mintedUBA.toString()); // pool fee
            console.log("Agent FAsset balance: ", (await context.fAsset.balanceOf(agent.vaultAddress)).toString());
            const poolAmt = toBN("24000000");
            const poolFee = await calculateFee(poolAmt, false);
            console.log("Transfer fee for pool self close:", poolFee.toString()); // needs extra transfer fee to self close
            await minter.transferFAsset(agent.ownerWorkAddress, poolFee, true);
            // Agent is going to be destroyed
            console.log("\n==== Agent is going to be destroyed ====");
            await agent.exitAndDestroy();
            console.log("Agent collateral pool token total supply after destroy: ", (await agent.collateralPoolToken.totalSupply()).toString());
            // announce agent pool token redemption
            let assetManagerFee = await context.fAsset.balanceOf(assetManager.address);
            console.log("Total transfer fee in Asset manager:", assetManagerFee.toString());
            const claimableAmount1 = await agent.transferFeeShare(10);
            console.log("Claimable amount of agent 1:", claimableAmount1.toString());
            const claimableAmount2 = await agent.transferFeeShare(10);
            console.log("Claimable amount of agent 2:", claimableAmount2.toString());
            // Agent 2 claims the transfer fees
            const claimed = await agent2.claimTransferFees(agent2.ownerWorkAddress, 10);
            console.log("Agent 2 claimed transfer fees:", claimed.agentClaimedUBA.toString());
            console.log("Agent 2 pool claimed transfer fees:", claimed.poolClaimedUBA.toString());
            // Agent 1 tries to claim transfer fees
            const claimed2 = agent.claimTransferFees(agent.ownerWorkAddress, 10);
            await expectRevert(claimed2, "invalid agent vault address")
            assetManagerFee = await context.fAsset.balanceOf(assetManager.address);
            console.log("Total transfer fee in Asset manager after agent2 claim:", assetManagerFee.toString());
        });
```

**Logs**

```ts
  Contract: AssetManagerSimulation.sol; test/integration/fasset-simulation/12-TrailingFees.ts; Asset manager simulations - transfer fees

==== Minter mints f-assets ====
Agent total Minted UBA:  24000000
Agent FAsset balance:  0
Transfer fee for pool: 4800

==== Agent is going to be destroyed ====
Agent collateral pool token total supply after destroy:  0

Total transfer fee in Asset manager: 124826
Claimable amount of agent 1: 60012
Claimable amount of agent 2: 60012

Agent 2 claimed transfer fees: 36008
Agent 2 pool claimed transfer fees: 24004

Total transfer fee in Asset manager after agent2 claim: 64814
      ✔ Agent destruction can lock the unclaimed transfer fees (864ms)

```
