# #37661 \[SC-High] Incorrect \`total\_active\_stake\` reduction causes loss of funds for the users and exc

**Submitted on Dec 11th 2024 at 22:09:38 UTC by @holydevoti0n for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37661
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The burn operation incorrectly reduces `total_active_stake` by including rewards in the reduction amount. This leads to inflated reward calculations, causing the protocol to collect excessive fees from user funds.

## Vulnerability Details

When users burn xALGO, the contract incorrectly reduces `total_active_stake` by the full withdrawal amount (stake + `rewards`): <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L824>

```solidity
def burn(send_xalgo: abi.AssetTransferTransaction, min_received: abi.Uint64) -> Expr:
    burn_amount = send_xalgo.get().asset_amount()
    algo_balance = ScratchVar(TealType.uint64)
    algo_to_send = ScratchVar(TealType.uint64)

    return Seq(
        # calculate algo amount to send
@>        algo_balance.store(
            App.globalGet(total_active_stake_key)
@>            + App.globalGet(total_rewards_key)
            - App.globalGet(total_unclaimed_fees_key)
        ),
@>        algo_to_send.store(
            mul_scale(
                burn_amount,
                algo_balance.load(),
                get_x_algo_circulating_supply() + burn_amount
            )
        ),
       ...
        # update total active stake
@>        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
        ...
    )
```

1. First burn distorts `total_active_stake` by including `rewards` in reduction.
2. Subsequent burns use this distorted `total_active_stake` value.
3. Creates compounding error where each burn further distorts the rate.

The second problem is that the `total_active_stake` is used to calculate the protocol fees and the total in rewards: <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L144-L147>

```solidity
def update_total_rewards_and_unclaimed_fees():
    old_total_rewards = ScratchVar(TealType.uint64)
    new_total_rewards = ScratchVar(TealType.uint64)

    return Seq(
        # update total rewards
        old_total_rewards.store(App.globalGet(total_rewards_key)),
        new_total_rewards.store(get_proposers_algo_balance(Int(0)) - App.globalGet(total_pending_stake_key) - App.globalGet(total_active_stake_key)),
@>        App.globalPut(total_rewards_key, new_total_rewards.load()),
        # update unclaimed fees
@>        App.globalPut(total_unclaimed_fees_key, App.globalGet(total_unclaimed_fees_key) + mul_scale(
            new_total_rewards.load() - old_total_rewards.load(),
            App.globalGet(fee_key),
            ONE_4_DP
        )),
    )
```

The artificially low `total_active_stake` causes:

* Inflated new\_total\_rewards calculation
* Excessive fee collection since fees are taken as a percentage of rewards
* Compounding effect as more burns occur over time

## Impact Details

* Loss of user funds through excessive fee collection(especially for the users that will keep the funds after several `burn` operations)
* Rewards and fees are calculated on stake that is incorrectly classified as rewards
* Impact compounds over time with each burn operation

## Recommendation

The burn operation should reduce `total_active_stake` proportionally based only on the staked amount. i.e:

```solidity
stake_reduction = mul_scale(
    burn_amount,
    App.globalGet(total_active_stake_key),
    get_x_algo_circulating_supply()
)

App.globalPut(total_active_stake_key, 
    App.globalGet(total_active_stake_key) - stake_reduction)
```

## Proof of Concept

The PoC shows the tracking of how `total_active_stake` decreases more than it should, which causes the protocol to misclassify staked funds as rewards.

Another observation is that the helper functions of the PoC is merely copy/paste code from the original test suite. I've done this so I could separate the test in a new file but still preserve the state and use the same logic.

Create a new test file called `incorrect-burning.test.ts` and paste the following code into it:

```javascript
import {
    ABIContract,
    Account,
    Algodv2,
    AtomicTransactionComposer,
    decodeAddress,
    decodeUint64,
    encodeUint64,
    generateAccount,
    getApplicationAddress,
    getMethodByName,
    IntDecoding,
    makeApplicationCreateTxn,
    makeApplicationUpdateTxn,
    makeBasicAccountTransactionSigner,
    modelsv2,
    OnApplicationComplete,
  } from "algosdk";
  import { mulScale, mulScaleRoundUp, ONE_16_DP, ONE_4_DP } from "folks-finance-js-sdk";
  import { sha256 } from "js-sha256";
  import { prepareOptIntoAssetTxn } from "./transactions/common";
  import {
    parseXAlgoConsensusV1GlobalState,
    prepareAddProposerForXAlgoConsensus,
    prepareBurnFromXAlgoConsensusV2,
    prepareClaimDelayedMintFromXAlgoConsensus,
    prepareClaimXAlgoConsensusV2Fee,
    prepareDelayedMintFromXAlgoConsensusV2,
    prepareImmediateMintFromXAlgoConsensusV2,
    prepareInitialiseXAlgoConsensusV1,
    prepareInitialiseXAlgoConsensusV2,
    preparePauseXAlgoConsensusMinting,
    prepareRegisterXAlgoConsensusOffline,
    prepareRegisterXAlgoConsensusOnline,
    prepareScheduleXAlgoConsensusSCUpdate,
    prepareUpdateXAlgoConsensusAdmin,
    prepareUpdateXAlgoConsensusV2Fee,
    prepareUpdateXAlgoConsensusPremium,
    prepareUpdateXAlgoConsensusMaxProposerBalance,
    prepareUpdateXAlgoConsensusSC,
    prepareSetXAlgoConsensusProposerAdmin,
    prepareSubscribeXAlgoConsensusProposerToXGov,
    prepareUnsubscribeXAlgoConsensusProposerFromXGov,
    prepareXAlgoConsensusDummyCall,
    parseXAlgoConsensusV2GlobalState,
    prepareCreateXAlgoConsensusV1,
    prepareMintFromXAlgoConsensusV1,
  } from "./transactions/xAlgoConsensus";
  import { getABIContract } from "./utils/abi";
  import { getAlgoBalance, getAssetBalance } from "./utils/account";
  import {
    compilePyTeal,
    compileTeal,
    enc,
    getAppGlobalState,
    getParsedValueFromState,
    parseUint64s
  } from "./utils/contracts";
  import { fundAccountWithAlgo } from "./utils/fund";
  import { privateAlgodClient, startPrivateNetwork, stopPrivateNetwork } from "./utils/privateNetwork";
  import { advanceBlockRounds, advancePrevBlockTimestamp } from "./utils/time";
  import { getParams, submitGroupTransaction, submitTransaction } from "./utils/transaction";
import fs from 'fs';
  
  jest.setTimeout(1000000);
  
  describe("Algo Consensus V2", () => {
    let algodClient: Algodv2;
    let prevBlockTimestamp: bigint;
    let user1: Account = generateAccount();
    let user2: Account = generateAccount();
    let proposer0: Account = generateAccount();
    let proposer1: Account = generateAccount();
    let admin: Account = generateAccount();
    let registerAdmin: Account = generateAccount();
    let xGovAdmin: Account = generateAccount();
    let proposerAdmin: Account = generateAccount();
    let xAlgoAppId: number, xAlgoId: number;
    let xAlgoConsensusABI: ABIContract;
  
    let xGovRegistryAppId: number;
    let xGovRegistryABI: ABIContract;
    let xGovFee: bigint;
  
    const timeDelay = BigInt(86400);
    const minProposerBalance = BigInt(5e6);
    const maxProposerBalance = BigInt(500e6);
    const premium = BigInt(0.001e16); // 0.1%
    const fee = BigInt(0.1e4); // 10%
  
    const nonce = Uint8Array.from([0, 0]);
    const resizeProposerBoxCost = BigInt(16000);
    const updateSCBoxCost = BigInt(32100);
    const delayMintBoxCost = BigInt(36100);

    beforeAll(async () => {
        console.log("Starting private network...");
        await startPrivateNetwork();
    
        const netConfig = Buffer.from(fs.readFileSync("net1/Primary/algod.net")).toString();
        const [host, port] = netConfig.split(":");
        console.log(`Using host: ${host}, port: ${port}`);
    
        try {
            const response = await fetch(`http://${host}:${port}/health`);
            if (!response.ok) {
                throw new Error(`Algod health check failed with status: ${response.status}`);
            }
            console.log("Algod is healthy.");
        } catch (error) {
            console.error("Failed to connect to Algod:", error);
            throw error;
        }
    
        algodClient = privateAlgodClient();
        algodClient.setIntEncoding(IntDecoding.MIXED);
    
        // initialise accounts with algo
        await fundAccountWithAlgo(algodClient, user1.addr, 1000e6, await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, user2.addr, 1000e6, await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, admin.addr, 1000e6, await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, registerAdmin.addr, 1000e6, await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, xGovAdmin.addr, 1000e6, await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, proposerAdmin.addr, 1000e6, await getParams(algodClient));
    
        // advance time well past current time so we are dealing with deterministic time using offsets
        prevBlockTimestamp = await advancePrevBlockTimestamp(algodClient, 1000);
    
        // deploy xgov registry
        const approval = await compileTeal(compilePyTeal('contracts/testing/xgov_registry'));
        const clear = await compileTeal(compilePyTeal('contracts/common/clear_program', 10));
        const tx = makeApplicationCreateTxn(user1.addr, await getParams(algodClient), OnApplicationComplete.NoOpOC, approval, clear, 0, 0, 1, 0);
        const txId = await submitTransaction(algodClient, tx, user1.sk);
        const txInfo = await algodClient.pendingTransactionInformation(txId).do();
        xGovRegistryAppId = txInfo["application-index"];
        xGovRegistryABI = getABIContract('contracts/testing/xgov_registry');
        xGovFee = BigInt(getParsedValueFromState(await getAppGlobalState(algodClient, xGovRegistryAppId), "xgov_fee") || 0);
    });

    afterAll(() => {
        stopPrivateNetwork();
    });

    describe("Burn", () => {
        beforeEach(async () => {
          await initializeXAlgoConsensusV2();
          await updateAdmins();
          await addProposer();
          await setProposerAdmin();
          await registerOnline();
          await registerOffline();
          await setupImmediateMint();
        });
        test("When burning, the protocol misclassify staked funds as rewards ", async () => {
            // Initial setup - Send rewards to proposers to simulate earnings
            const rewards = BigInt(10e6); 
            await fundAccountWithAlgo(algodClient, proposer0.addr, rewards);
            await fundAccountWithAlgo(algodClient, proposer1.addr, rewards);
            
            const proposerAddrs = [proposer0.addr, proposer1.addr];
            
            // First mint
            const mintAmount = BigInt(200e6);
            const minReceived = BigInt(0);
            let txns = [
                prepareXAlgoConsensusDummyCall(xAlgoConsensusABI, xAlgoAppId, user1.addr, [], await getParams(algodClient)),
                ...prepareImmediateMintFromXAlgoConsensusV2(
                    xAlgoConsensusABI, 
                    xAlgoAppId,
                    xAlgoId,
                    user1.addr,
                    mintAmount,
                    minReceived,
                    proposerAddrs,
                    await getParams(algodClient)
                )
            ];
            await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
        
            // Get state after mint
            let stateBefore = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
            let { algoBalance: balanceBefore, xAlgoCirculatingSupply: supplyBefore } = await getXAlgoRate();
            
            console.log("\nAfter Initial Mint:");
            console.log("Total Active Stake:", stateBefore.totalActiveStake.toString());
            console.log("Total Rewards:", stateBefore.totalRewards.toString());
            console.log("ALGO Balance:", balanceBefore.toString());
            console.log("xALGO Supply:", supplyBefore.toString());
        
            // First burn - 25% of xALGO
            const burnAmount1 = supplyBefore / BigInt(4);
            txns = [
                prepareXAlgoConsensusDummyCall(xAlgoConsensusABI, xAlgoAppId, user1.addr, [], await getParams(algodClient)),
                ...prepareBurnFromXAlgoConsensusV2(
                    xAlgoConsensusABI,
                    xAlgoAppId,
                    xAlgoId,
                    user1.addr,
                    burnAmount1,
                    BigInt(0),
                    proposerAddrs,
                    await getParams(algodClient)
                )
            ];
            await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
        
            // Check state after first burn
            let stateAfterBurn1 = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
            let { algoBalance: balanceAfterBurn1 } = await getXAlgoRate();
            
            console.log("\nAfter First Burn (25%):");
            console.log("Total Active Stake:", stateAfterBurn1.totalActiveStake.toString());
            console.log("Total Rewards:", stateAfterBurn1.totalRewards.toString());
            console.log("ALGO Balance:", balanceAfterBurn1.toString());
        
            // Calculate expected stake reduction for first burn
            const expectedStakeReduction1 = (stateBefore.totalActiveStake * burnAmount1) / supplyBefore;
            const actualStakeReduction1 = stateBefore.totalActiveStake - stateAfterBurn1.totalActiveStake;
        
            console.log("\nFirst Burn Analysis:");
            console.log("Expected Stake Reduction:", expectedStakeReduction1.toString());
            console.log("Actual Stake Reduction:", actualStakeReduction1.toString());
            
            // Verify first burn reduced stake too much
            expect(actualStakeReduction1).toBeGreaterThan(expectedStakeReduction1);
            
            // Second burn to demonstrate compounding error
            const burnAmount2 = supplyBefore / BigInt(4);
            txns = [
                prepareXAlgoConsensusDummyCall(xAlgoConsensusABI, xAlgoAppId, user1.addr, [], await getParams(algodClient)),
                ...prepareBurnFromXAlgoConsensusV2(
                    xAlgoConsensusABI,
                    xAlgoAppId,
                    xAlgoId,
                    user1.addr,
                    burnAmount2,
                    BigInt(0),
                    proposerAddrs,
                    await getParams(algodClient)
                )
            ];
            await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
        
            // Final state check
            const finalState = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
            
            // Calculate total expected stake reduction
            const totalExpectedReduction = (expectedStakeReduction1 * BigInt(2));
            const totalActualReduction = stateBefore.totalActiveStake - finalState.totalActiveStake;
        
            console.log("\nFinal Analysis:"); 
            console.log("Total Expected Reduction:", totalExpectedReduction.toString());
            console.log("Total Actual Reduction:", totalActualReduction.toString());
            console.log("Final Active Stake:", finalState.totalActiveStake.toString());
            console.log("Final Rewards:", finalState.totalRewards.toString());
        
            // Verify compounding error made total reduction too large
            expect(totalActualReduction).toBeGreaterThan(totalExpectedReduction);
        });
    });

    // HELPER FUNCTIONS
    async function getXAlgoRate() {
        const atc = new AtomicTransactionComposer();
        atc.addMethodCall({
          sender: user1.addr,
          signer: makeBasicAccountTransactionSigner(user1),
          appID: xAlgoAppId,
          method: getMethodByName(xAlgoConsensusABI.methods, "get_xalgo_rate"),
          methodArgs: [],
          suggestedParams: await getParams(algodClient),
        });
        const simReq = new modelsv2.SimulateRequest({
          txnGroups: [],
          allowUnnamedResources: true,
        })
        const { methodResults }  = await atc.simulate(algodClient, simReq);
        const { returnValue } = methodResults[0];
        const [algoBalance, xAlgoCirculatingSupply, balances]: [bigint, bigint, Uint8Array] = returnValue as any;
        const proposersBalances = parseUint64s(Buffer.from(balances).toString("base64"));
        return { algoBalance, xAlgoCirculatingSupply, proposersBalances } ;
      }
  
      // async function to initialize the application.
      async function initializeXAlgoConsensusV2() {
          // deploy algo consensus v1
        const { tx: createTx, abi } = await prepareCreateXAlgoConsensusV1(admin.addr, admin.addr, registerAdmin.addr, minProposerBalance, maxProposerBalance, premium, fee, await getParams(algodClient));
        xAlgoConsensusABI = abi;
        let txId = await submitTransaction(algodClient, createTx, admin.sk);
        let txInfo = await algodClient.pendingTransactionInformation(txId).do();
        xAlgoAppId = txInfo["application-index"];
  
        // fund minimum balance
        await fundAccountWithAlgo(algodClient, proposer0.addr, BigInt(0.1e6), await getParams(algodClient));
        await fundAccountWithAlgo(algodClient, getApplicationAddress(xAlgoAppId), 0.6034e6);
  
        // initialise algo consensus v1
        const initTxns = prepareInitialiseXAlgoConsensusV1(xAlgoConsensusABI, xAlgoAppId, admin.addr, proposer0.addr, await getParams(algodClient));
        [, txId] = await submitGroupTransaction(algodClient, initTxns, [proposer0.sk, admin.sk]);
        txInfo = await algodClient.pendingTransactionInformation(txId).do();
        xAlgoId = txInfo['inner-txns'][0]['asset-index'];
  
        // verify xAlgo was created
        const assetInfo = await algodClient.getAssetByID(xAlgoId).do();
        expect(assetInfo.params.creator).toEqual(getApplicationAddress(xAlgoAppId));
        expect(assetInfo.params.reserve).toEqual(getApplicationAddress(xAlgoAppId));
        expect(assetInfo.params.total).toEqual(BigInt(10e15));
        expect(assetInfo.params.decimals).toEqual(6);
        expect(assetInfo.params.name).toEqual('Governance xAlgo');
        expect(assetInfo.params['unit-name']).toEqual('xALGO');
  
        // opt into xALGO
        let optInTx = prepareOptIntoAssetTxn(admin.addr, xAlgoId, await getParams(algodClient));
        await submitTransaction(algodClient, optInTx, admin.sk);
        optInTx = prepareOptIntoAssetTxn(user1.addr, xAlgoId, await getParams(algodClient));
        await submitTransaction(algodClient, optInTx, user1.sk);
        optInTx = prepareOptIntoAssetTxn(user2.addr, xAlgoId, await getParams(algodClient));
        await submitTransaction(algodClient, optInTx, user2.sk);
  
        // mint to get pool started
        const mintAmount = BigInt(100e6);
        const mintTxns = prepareMintFromXAlgoConsensusV1(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user2.addr, mintAmount, proposer0.addr, await getParams(algodClient));
        await submitGroupTransaction(algodClient, mintTxns, mintTxns.map(() => user2.sk));
  
        // verify global state
        const state = await parseXAlgoConsensusV1GlobalState(algodClient, xAlgoAppId);
        expect(state.initialised).toEqual(true);
        expect(state.admin).toEqual(admin.addr);
        expect(state.registerAdmin).toEqual(registerAdmin.addr);
        expect(state.xAlgoId).toEqual(xAlgoId);
        expect(state.numProposers).toEqual(BigInt(1));
        expect(state.minProposerBalance).toEqual(minProposerBalance);
        expect(state.maxProposerBalance).toEqual(maxProposerBalance);
        expect(state.fee).toEqual(fee);
        expect(state.premium).toEqual(premium);
        expect(state.totalPendingStake).toEqual(BigInt(0));
        expect(state.totalActiveStake).toEqual(mintAmount);
        expect(state.totalRewards).toEqual(BigInt(0));
        expect(state.totalUnclaimedFees).toEqual(BigInt(0));
        expect(state.canImmediateMint).toEqual(true);
        expect(state.canDelayMint).toEqual(false);
  
        // verify proposers box
        const proposersBox = await algodClient.getApplicationBoxByName(xAlgoAppId, enc.encode("pr")).do();
        const proposers = new Uint8Array(960);
        proposers.set(decodeAddress(proposer0.addr).publicKey, 0);
        expect(proposersBox.value).toEqual(proposers);
  
        // verify added proposer box
        const boxName = Uint8Array.from([...enc.encode("ap"), ...decodeAddress(proposer0.addr).publicKey]);
        const addedProposerBox = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
        expect(addedProposerBox.value).toEqual(new Uint8Array(0));
  
        // verify balances
        const user2XAlgoBalance = await getAssetBalance(algodClient, user2.addr, xAlgoId);
        expect(user2XAlgoBalance).toEqual(mintAmount);
  
        // update to algo consensus v2
        const approval = await compileTeal(compilePyTeal('contracts/xalgo/consensus_v2'));
        const clear = await compileTeal(compilePyTeal('contracts/common/clear_program', 10));
        const updateTx = makeApplicationUpdateTxn(admin.addr, await getParams(algodClient), xAlgoAppId, approval, clear);
        await submitTransaction(algodClient, updateTx, admin.sk);
        xAlgoConsensusABI = getABIContract('contracts/xalgo/consensus_v2');
  
  
        // initialise
        const oldState = await parseXAlgoConsensusV1GlobalState(algodClient, xAlgoAppId);
  
        // initialise
        const tx = prepareInitialiseXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, admin.addr, await getParams(algodClient));
        await submitTransaction(algodClient, tx, admin.sk);
  
        // verify global state
        const unformattedState = await getAppGlobalState(algodClient, xAlgoAppId);
        expect(getParsedValueFromState(unformattedState, "initialised")).toBeUndefined();
        expect(getParsedValueFromState(unformattedState, "min_proposer_balance")).toBeUndefined();
  
        const newState = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        expect(newState.initialised).toEqual(true);
        expect(newState.admin).toEqual(oldState.admin);
        expect(newState.xGovAdmin).toEqual(oldState.admin);
        expect(newState.registerAdmin).toEqual(oldState.registerAdmin);
        expect(newState.xAlgoId).toEqual(oldState.xAlgoId);
        expect(newState.numProposers).toEqual(oldState.numProposers);
        expect(newState.maxProposerBalance).toEqual(oldState.maxProposerBalance);
        expect(newState.fee).toEqual(oldState.fee);
        expect(newState.premium).toEqual(oldState.premium);
        expect(newState.totalPendingStake).toEqual(oldState.totalPendingStake);
        expect(newState.totalActiveStake).toEqual(oldState.totalActiveStake);
        expect(newState.totalRewards).toEqual(oldState.totalRewards);
        expect(newState.totalUnclaimedFees).toEqual(oldState.totalUnclaimedFees);
        expect(newState.canImmediateMint).toEqual(oldState.canImmediateMint);
        expect(newState.canDelayMint).toEqual(oldState.canDelayMint);
      }
  
      async function setupImmediateMint() { 
        const { canImmediateMint, canDelayMint } = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
  
        // update pause minting
        let tx = preparePauseXAlgoConsensusMinting(xAlgoConsensusABI, xAlgoAppId, admin.addr, "can_immediate_mint", canImmediateMint, await getParams(algodClient));
        await submitTransaction(algodClient, tx, admin.sk);
        tx = preparePauseXAlgoConsensusMinting(xAlgoConsensusABI, xAlgoAppId, admin.addr, "can_delay_mint", canDelayMint, await getParams(algodClient));
        await submitTransaction(algodClient, tx, admin.sk);
        let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        expect(state.canImmediateMint).toEqual(!canImmediateMint);
        expect(state.canDelayMint).toEqual(!canDelayMint);
  
        // restore pause minting
        tx = preparePauseXAlgoConsensusMinting(xAlgoConsensusABI, xAlgoAppId, admin.addr, "can_immediate_mint", !canImmediateMint, await getParams(algodClient));
        await submitTransaction(algodClient, tx, admin.sk);
        tx = preparePauseXAlgoConsensusMinting(xAlgoConsensusABI, xAlgoAppId, admin.addr, "can_delay_mint", !canDelayMint, await getParams(algodClient));
        await submitTransaction(algodClient, tx, admin.sk);
        state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        expect(state.canImmediateMint).toEqual(canImmediateMint);
        expect(state.canDelayMint).toEqual(canDelayMint);
      }
  
      async function addProposer() { 
          const minBalance = BigInt(16100);
          await fundAccountWithAlgo(algodClient, getApplicationAddress(xAlgoAppId), minBalance, await getParams(algodClient));
          await fundAccountWithAlgo(algodClient, proposer1.addr, BigInt(0.1e6), await getParams(algodClient));
    
          // balances before
          const proposerAlgoBalanceB = await getAlgoBalance(algodClient, proposer0.addr);
          const appAlgoBalanceB = await getAlgoBalance(algodClient, getApplicationAddress(xAlgoAppId));
          let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
          const { totalActiveStake: oldTotalActiveStake } = state;
    
          // register
          const txns = prepareAddProposerForXAlgoConsensus(xAlgoConsensusABI, xAlgoAppId, registerAdmin.addr, proposer1.addr, await getParams(algodClient));
          const [, txId] = await submitGroupTransaction(algodClient, txns, [proposer1.sk, registerAdmin.sk]);
          const txInfo = await algodClient.pendingTransactionInformation(txId).do();
          state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
          expect(state.totalActiveStake).toEqual(oldTotalActiveStake);
    
          // balances after
          const { proposersBalances } = await getXAlgoRate();
          const proposerAlgoBalanceA = await getAlgoBalance(algodClient, proposer0.addr);
          const appAlgoBalanceA = await getAlgoBalance(algodClient, getApplicationAddress(xAlgoAppId));
          expect(proposersBalances.length).toEqual(2);
          expect(proposersBalances[1]).toEqual(BigInt(0.1e6));
          expect(proposerAlgoBalanceA).toEqual(proposerAlgoBalanceB);
          expect(appAlgoBalanceA).toEqual(appAlgoBalanceB);
          expect(txInfo['inner-txns']).toBeUndefined();
    
          // verify proposers box
          const proposersBox = await algodClient.getApplicationBoxByName(xAlgoAppId, enc.encode("pr")).do();
          const proposers = new Uint8Array(960);
          proposers.set(decodeAddress(proposer0.addr).publicKey, 0);
          proposers.set(decodeAddress(proposer1.addr).publicKey, 32);
          expect(proposersBox.value).toEqual(proposers);
    
          // verify added proposer box
          const boxName = Uint8Array.from([...enc.encode("ap"), ...decodeAddress(proposer1.addr).publicKey]);
          const addedProposerBox = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
          expect(addedProposerBox.value).toEqual(new Uint8Array(0));
      }

    async function updateAdmins() { 
      // admin updating admin
      let tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "admin", admin.addr, user1.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, admin.sk);
      let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.admin).toEqual(user1.addr);

      // restore old admin
      tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "admin", user1.addr, admin.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, user1.sk);
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.admin).toEqual(admin.addr);

      // admin updating register admin
      tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "register_admin", admin.addr, user1.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, admin.sk);
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.registerAdmin).toEqual(user1.addr);

      // register admin updating register admin
      tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "register_admin", user1.addr, registerAdmin.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, user1.sk);
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.registerAdmin).toEqual(registerAdmin.addr);

      // admin updating xgov admin
      tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "xgov_admin", admin.addr, user1.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, admin.sk);
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.xGovAdmin).toEqual(user1.addr);

      // xgov admin updating xgov admin
      tx = prepareUpdateXAlgoConsensusAdmin(xAlgoConsensusABI, xAlgoAppId, "xgov_admin", user1.addr, xGovAdmin.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, user1.sk);
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.xGovAdmin).toEqual(xGovAdmin.addr);
    }

    async function updatePremium() {
       // update premium
       const tempPremium = BigInt(0.0025e16);
       let tx = prepareUpdateXAlgoConsensusPremium(xAlgoConsensusABI, xAlgoAppId, admin.addr, tempPremium, await getParams(algodClient));
       await submitTransaction(algodClient, tx, admin.sk);
       let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
       expect(state.premium).toEqual(tempPremium);
 
       // restore old premium
       tx = prepareUpdateXAlgoConsensusPremium(xAlgoConsensusABI, xAlgoAppId, admin.addr, premium, await getParams(algodClient));
       await submitTransaction(algodClient, tx, admin.sk);
       state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
       expect(state.premium).toEqual(premium);
    }

    async function setProposerAdmin() {
        const boxName = Uint8Array.from([...enc.encode("ap"), ...decodeAddress(proposer0.addr).publicKey]);
        let addedProposerBox = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
        expect(addedProposerBox.value).toEqual(new Uint8Array(0));

        // fund box
        await fundAccountWithAlgo(algodClient, getApplicationAddress(xAlgoAppId), resizeProposerBoxCost, await getParams(algodClient));

        // immediate if no existing proposer admin
        let tx = prepareSetXAlgoConsensusProposerAdmin(xAlgoConsensusABI, xAlgoAppId, registerAdmin.addr, 0, proposer0.addr, user1.addr, await getParams(algodClient));
        await submitTransaction(algodClient, tx, registerAdmin.sk)
        addedProposerBox = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
        expect(addedProposerBox.value).toEqual(Uint8Array.from([...encodeUint64(prevBlockTimestamp), ...decodeAddress(user1.addr).publicKey]));

        // delay if existing proposer admin
        tx = prepareSetXAlgoConsensusProposerAdmin(xAlgoConsensusABI, xAlgoAppId, registerAdmin.addr, 0, proposer0.addr, proposerAdmin.addr, await getParams(algodClient));
        await submitTransaction(algodClient, tx, registerAdmin.sk)
        addedProposerBox = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
        expect(addedProposerBox.value).toEqual(Uint8Array.from([...encodeUint64(prevBlockTimestamp + timeDelay), ...decodeAddress(proposerAdmin.addr).publicKey]));

        // proceed to timestamp after timeDelay
        const box = await algodClient.getApplicationBoxByName(xAlgoAppId, boxName).do();
        const ts = decodeUint64(box.value.subarray(0, 8), "bigint");
        const offset = Number(ts - prevBlockTimestamp) + 1;
        prevBlockTimestamp = await advancePrevBlockTimestamp(algodClient, offset);
    }

    async function registerOnline() { 
        const registerFeeAmount = BigInt(2e6);
        const voteKey = Buffer.from("G/lqTV6MKspW6J8wH2d8ZliZ5XZVZsruqSBJMwLwlmo=", "base64");
        const selKey = Buffer.from("LrpLhvzr+QpN/bivh6IPpOaKGbGzTTB5lJtVfixmmgk=", "base64");
        const stateProofKey = Buffer.from("Nn0fiJDZH2wyLqxNzrOC3WPF8Vz3AH8JU1IGI2H2xdcnRiqw7YuWkohuKHpC1EJMAe6ZVbUS/S2rPeCRAolfRQ==", "base64");
        const voteFirstRound = 1;
        const voteLastRound = 5000;
        const voteKeyDilution = 1500;

        const txns = prepareRegisterXAlgoConsensusOnline(xAlgoConsensusABI, xAlgoAppId, proposerAdmin.addr, registerFeeAmount, 0, proposer0.addr, voteKey, selKey, stateProofKey, voteFirstRound, voteLastRound, voteKeyDilution, await getParams(algodClient));
        const [, txId] = await submitGroupTransaction(algodClient, txns, txns.map(() => proposerAdmin.sk));
        const txInfo = await algodClient.pendingTransactionInformation(txId).do();

        // check key registration
        const innerRegisterOnlineTx = txInfo['inner-txns'][0]['txn']['txn'];
        expect(innerRegisterOnlineTx.type).toEqual('keyreg');
        expect(innerRegisterOnlineTx.snd).toEqual(Uint8Array.from(decodeAddress(proposer0.addr).publicKey));
        expect(innerRegisterOnlineTx.votekey).toEqual(Uint8Array.from(voteKey));
        expect(innerRegisterOnlineTx.selkey).toEqual(Uint8Array.from(selKey));
        expect(innerRegisterOnlineTx.sprfkey).toEqual(Uint8Array.from(stateProofKey));
        expect(innerRegisterOnlineTx.votefst).toEqual(voteFirstRound);
        expect(innerRegisterOnlineTx.votelst).toEqual(voteLastRound);
        expect(innerRegisterOnlineTx.votekd).toEqual(voteKeyDilution);
        expect(innerRegisterOnlineTx.fee).toEqual(Number(registerFeeAmount));
    }

    async function registerOffline() { 
        for (const sender of [registerAdmin, proposerAdmin]) {
            const tx = prepareRegisterXAlgoConsensusOffline(xAlgoConsensusABI, xAlgoAppId, sender.addr, 0, proposer0.addr, await getParams(algodClient));
            const txId = await submitTransaction(algodClient, tx, sender.sk);
            const txInfo = await algodClient.pendingTransactionInformation(txId).do();
    
            // check key registration
            const innerRegisterOnlineTx = txInfo['inner-txns'][0]['txn']['txn'];
            expect(innerRegisterOnlineTx.type).toEqual('keyreg');
            expect(innerRegisterOnlineTx.snd).toEqual(Uint8Array.from(decodeAddress(proposer0.addr).publicKey));
            expect(innerRegisterOnlineTx.votekey).toBeUndefined();
            expect(innerRegisterOnlineTx.selkey).toBeUndefined();
            expect(innerRegisterOnlineTx.sprfkey).toBeUndefined();
            expect(innerRegisterOnlineTx.votefst).toBeUndefined();
            expect(innerRegisterOnlineTx.votelst).toBeUndefined();
            expect(innerRegisterOnlineTx.votekd).toBeUndefined();
            expect(innerRegisterOnlineTx.fee).toBeUndefined();
          }
    }
  });
```

then run: `PYTHONPATH="./contracts" npx jest test/incorrect-burning.test.ts --runInBand`

Output:

```md
Final Analysis:
Total Expected Reduction: 149999998
Total Actual Reduction: 158999998 
Final Active Stake: 141000002
Final Rewards: 20000000

PASS test/incorrect-burning.test.ts (11.539 s)
Algo Consensus V2 Burn 
✓ When burning, the protocol misclassify staked funds as rewards (4263 ms)
```
