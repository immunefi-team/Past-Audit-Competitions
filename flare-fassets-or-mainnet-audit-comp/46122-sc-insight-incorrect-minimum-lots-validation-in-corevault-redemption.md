# #46122 \[SC-Insight] Incorrect Minimum Lots Validation in CoreVault Redemption

**Submitted on May 25th 2025 at 07:18:51 UTC by @aman for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46122
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CoreVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `redeemFromCoreVault` function in `CoreVault.sol` incorrectly validates the minimum lots requirement when the available lots in the core vault are less than the `coreVaultMinimumRedeemLots` setting. This allows redeemers to create redemption requests with fewer lots than the minimum requirement.

## Vulnerability Details

The vulnerable code is in `CoreVault.sol`:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
)
    internal
    onlyEnabled
{
    State storage state = getState();
    require(state.coreVaultManager.isDestinationAddressAllowed(_redeemerUnderlyingAddress),
        "underlying address not allowed by core vault");
    AssetManagerSettings.Data storage settings = Globals.getSettings();
    uint64 availableLots = getCoreVaultAmountLots();
    uint64 minimumRedeemLots = SafeMath64.min64(state.minimumRedeemLots, availableLots);
    require(_lots >= minimumRedeemLots, "requested amount too small");
    // ... rest of the function
}
```

The issue is that the code:

1. Gets the available lots in the core vault
2. Sets `minimumRedeemLots` as the minimum of `state.minimumRedeemLots` and `availableLots`
3. This means if `availableLots` is less than `minimumRedeemLots`, the requirement is lowered
4. This contradicts the documentation which states that lots must be larger than `coreVaultMinimumRedeemLots`

## Impact Details

* **Impact**: Allows redemption requests with fewer lots than the minimum requirement
* **Likelihood**: Low - Only occurs when core vault has insufficient funds

## References

* File: `contracts/assetManager/library/CoreVault.sol`
* Function: `redeemFromCoreVault`

## Recommendations

1. Fix the minimum lots validation to always enforce the minimum requirement:

```solidity
function redeemFromCoreVault(
    uint64 _lots,
    string memory _redeemerUnderlyingAddress
)
    internal
    onlyEnabled
{
    State storage state = getState();
    require(state.coreVaultManager.isDestinationAddressAllowed(_redeemerUnderlyingAddress),
        "underlying address not allowed by core vault");
    require(_lots >= state.minimumRedeemLots, "requested amount too small");
    uint64 availableLots = getCoreVaultAmountLots();
    require(_lots <= availableLots, "not enough available on core vault");
    // ... rest of the function
}
```

## Proof of Concept

## Proof of Concept

Add the Following test Case to `CoreVault.ts` :

```javascript
it.only("request direct redemption from core vault less than min Redeem lots", async () => { // @audit-issue : redeem less than min lots
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        await prefundCoreVault(minter.underlyingAddress, 1e6);
        // allow CV manager addresses
        await coreVaultManager.addAllowedDestinationAddresses([redeemer.underlyingAddress], { from: governance });
        // make agent available
        await agent.depositCollateralLotsAndMakeAvailable(100);
        // mint
        const [minted] = await minter.performMinting(agent.vaultAddress, 2);
        await minter.transferFAsset(redeemer.address, minted.mintedAmountUBA);
        // agent requests transfer for some backing to core vault
        const transferAmount = context.convertLotsToUBA(2);
        await agent.transferToCoreVault(transferAmount);
        let res = await context.assetManager.setCoreVaultMinimumRedeemLots(3, { from: context.governance });
        expectEvent(res, "SettingChanged", { name: "coreVaultMinimumRedeemLots", value: "3" })
        const minLots = await context.assetManager.getCoreVaultMinimumRedeemLots();
        assertWeb3Equal(minLots, 3);
        const lots = 2;
        // redeemer requests direct redemption from CV
        const [paymentAmount1, paymentReference1] = await testRedeemFromCV(redeemer, lots);
        const trigRes = await coreVaultManager.triggerInstructions({ from: triggeringAccount });
        const paymentReqs = filterEvents(trigRes, "PaymentInstructions");
        assertWeb3Equal(paymentReqs[0].args.account, coreVaultUnderlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.destination, redeemer.underlyingAddress);
        assertWeb3Equal(paymentReqs[0].args.amount, paymentAmount1);
        expect(toBN(minLots).gt(toBN(lots))).to.be.true;
    });
```

Run with command `yarn testHH`.
