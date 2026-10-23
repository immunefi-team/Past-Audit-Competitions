# #55208 \[SC-Low] executors receive a greater reward than the assigned value

**Submitted on Sep 24th 2025 at 11:33:14 UTC by @rick137 for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #55208
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/commit/7dd1ddd574989c44b3057ce426ff188bc69743d1>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Root Cause

Minters are required to pay a reservation fee to mint new `fAsset`. Furthermore, minters may designate an address as an executor to carry out their minting process, and the executor receives a reward determined by the minter. However, the executor may receive a reward exceeding the allocated value, resulting in a loss of funds for the minter.

The reservation fee is defined as `valueAMG` in terms of NAT token times `collateralReservationFeeBIPS`. This means NAT's price has an impact on the final payout by the minter.

Issue path (example scenario):

* Assume NAT's price is $1 and `collateralReservationFeeBIPS` is 1%.
* The minter establishes a reservation collateral for 1 lot with 150 NAT as `msg.value`, allocating 100 NAT for the reservation fee and 50 NAT for the executor.
* The price of NAT rises to $1.1 upon the execution of the minter's transaction, resulting in a reservation fee of 90 NAT and an executor fee of 59 NAT, which exceeds the amount the minter designated for the executor. The final payout should be 90 NAT plus 50 NAT, totaling 140 NAT, not 150 NAT.

## Proof of Concept

```typescript
it.only("reserve-collateral", async() => {
    const fAssetImpl = await FAsset.new();
    const fAssetProxy = await FAssetProxy.new(fAssetImpl.address, "fEthereum", "fETH", "Ethereum", "ETH", 18, { from: governance });
    fAsset = await FAsset.at(fAssetProxy.address);

    let wNat = await ERC20Mock.new("wNative", "wNat");
    assetManager = await AssetManager.new(wNat.address);
    await fAsset.setAssetManager(assetManager.address, { from: governance });
     // 1- agent is created and agent become available
    const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
    const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(1e8));
    const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
    // make agent available one lot worth of pool collateral
    const requiredCollateral = await agent.requiredCollateralForLots(100);
    await agent.depositCollateralsAndMakeAvailable(requiredCollateral.vault, requiredCollateral.pool);

    const poolTokenSupplyBefore = await agent.collateralPoolToken.totalSupply();
    // minter mints
    const lots = 1;
    let agentInfo = await agent.getAgentInfo();
    let snapshotId = await network.provider.send("evm_snapshot", []);
    await context.priceStore.setCurrentPrice("NAT", 110000, 0);
    let res = await context.assetManager.reserveCollateral(agent.vaultAddress, lots, agentInfo.feeBIPS, redeemer.address, {from: minter.address, value : toBNExp(150, 18)});
    let reserveColl = requiredEventArgs(res, 'CollateralReserved');
    console.log("executor paying out when NAT price raises to $1.1 upon execution of the minter's transaction-----------------------------------");
    console.log("executorFeeNat:", reserveColl.executorFeeNatWei.toString() / 1e18);
    console.log("reservationFeeNat:", Number(toBNExp(150, 18).sub(reserveColl.executorFeeNatWei)) / 1e18);


    console.log("executor paying out when NAT price is $1-----------------------------------");
    await network.provider.send("evm_revert", [snapshotId]);
    res = await context.assetManager.reserveCollateral(agent.vaultAddress, lots, agentInfo.feeBIPS, redeemer.address, {from: minter.address, value : toBNExp(150, 18)});
    reserveColl = requiredEventArgs(res, 'CollateralReserved');
    console.log("executorFeeNat:", reserveColl.executorFeeNatWei.toString() / 1e18);
    console.log("reservationFeeNat:", Number(toBNExp(150, 18).sub(reserveColl.executorFeeNatWei)) / 1e18);
});
```

Console output:

```console
  Contract: AssetManager.sol; test/integration/assetManager/AttackScenarios.ts; Asset manager integration tests
challenger's reward:300 deprecated collateral
    ✔ challenger receives deprecated collateral (105ms)
executor paying out when NAT price raises to $1.1 upon execution of the minter's transaction-----------------------------------
executorFeeNat: 59.090909090000004
reservationFeeNat: 90.90909091
executor paying out when NAT price is $1-----------------------------------
executorFeeNat: 50
reservationFeeNat: 100
    ✔ reserve-collateral (78ms)


  2 passing (3s)
```

(End of report)
