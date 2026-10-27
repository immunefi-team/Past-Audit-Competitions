# #37791 \[SC - Insight] consensus contract distributes algo for proposers that are offline that cause

## #37791 \[SC-Insight] Consensus contract distributes Algo for proposers that are offline that cause losing of reward

**Submitted on Dec 16th 2024 at 07:44:30 UTC by @perseverance for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37791
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Description

### Brief/Intro

When users mint XALGO, they can use "immediate\_mint" or "delayed\_mint" by sending ALGO to the consensus contract. Then the consensus will distribute the ALGO to all Proposers almost evenly.

The register\_admin or proposer\_admin can register the proposer to be offline by calling "register\_offline" function.

### The vulnerability

#### Vulnerability Details

When distributing the ALGO, the consensus contract does not check if the proposer is offline, but still distributes the ALGO evenly. So if in case, that the register\_admin or proposer\_admin register the proposer to be offline, then the proposer status is offline, then the proposer will not earn any reward. By still distributing the ALGO to offline proposers, this will cause losing of the reward, because the user's capital is not used for active proposers.

So if the offline period and number of offline proposers is high enough, then this will significantly impact users' interest.

## Impacts

## About the severity assessment

Bug Severity: Low

Impact category:

Contract fails to deliver promised returns, but doesn't lose value

Likelyhood: Very Likely

Recommendation: Implement the status tracking of proposer status and prioritize ALGO distribution for online proposers to maximize the reward earning.

### Proof of Concept

## Proof of concept

Steps to reproduce the bug:

Step 1: The register\_admin or proposer\_admin can register the proposer to be offline by calling "register\_offline" function.

Step 2: User call "immediate\_mint" to mint the XALGO. The ALGO received is distributed for offline proposers.

Test code:

I modify the test case "succeeds and splits between proposers" in algo-liquid-staking-contracts\test\xAlgoConsensusV2.test.ts and add the code to register\_offline

```typescript
for (const sender of [registerAdmin, proposerAdmin]) {
        const tx_1 = prepareRegisterXAlgoConsensusOffline(xAlgoConsensusABI, xAlgoAppId, sender.addr, 0, proposer0.addr, await getParams(algodClient));
        const txId_1 = await submitTransaction(algodClient, tx_1, sender.sk);
        const txInfo_1 = await algodClient.pendingTransactionInformation(txId_1).do();

        // check key registration
        const innerRegisterOnlineTx = txInfo_1['inner-txns'][0]['txn']['txn'];
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
```

So the full test case. Just copy the test case to file: algo-liquid-staking-contracts\test\xAlgoConsensusV2.test.ts

```typescript
test("succeeds and splits between proposers", async () => {
      // airdrop rewards
      const additionalRewards = BigInt(10e6);
      await fundAccountWithAlgo(algodClient, proposer1.addr, additionalRewards, await getParams(algodClient));
      const additionalRewardsFee = mulScale(additionalRewards, fee, ONE_4_DP);

      // ensure allocation will go to both proposers
      const { algoBalance: oldAlgoBalance, xAlgoCirculatingSupply: oldXAlgoCirculatingSupply, proposersBalances: oldProposersBalance } = await getXAlgoRate();
      expect(oldProposersBalance[0]).toBeGreaterThan(oldProposersBalance[1]);
      const excessMintAmount = BigInt(5e6);
      const diffMintAmount = oldProposersBalance[0] - oldProposersBalance[1];
      const mintAmount = diffMintAmount + excessMintAmount;

      // calculate rate
      const minReceived = BigInt(0);
      const expectedReceived = mulScale(
        mulScale(mintAmount, oldXAlgoCirculatingSupply, oldAlgoBalance),
        ONE_16_DP - premium,
        ONE_16_DP
      );

    // [perseverance] Add test code to register_offline here
      for (const sender of [registerAdmin, proposerAdmin]) {
        const tx_1 = prepareRegisterXAlgoConsensusOffline(xAlgoConsensusABI, xAlgoAppId, sender.addr, 0, proposer0.addr, await getParams(algodClient));
        const txId_1 = await submitTransaction(algodClient, tx_1, sender.sk);
        const txInfo_1 = await algodClient.pendingTransactionInformation(txId_1).do();

        // check key registration
        const innerRegisterOnlineTx = txInfo_1['inner-txns'][0]['txn']['txn'];
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

    // end of modification

      // state before
      let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      const {
        totalPendingStake: oldTotalPendingStake,
        totalActiveStake: oldTotalActiveStake,
        totalRewards: oldTotalRewards ,
        totalUnclaimedFees: oldTotalUnclaimedFees ,
      } = state;

      // immediate mint
      const proposerAddrs = [proposer0.addr, proposer1.addr];
      const txns = [
        prepareXAlgoConsensusDummyCall(xAlgoConsensusABI, xAlgoAppId, user1.addr, [], await getParams(algodClient)),
        ...prepareImmediateMintFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user1.addr, mintAmount, minReceived, proposerAddrs, await getParams(algodClient))
      ];
      const [, , txId] = await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
      const txInfo = await algodClient.pendingTransactionInformation(txId).do();
      const { txn: algoTransfer0 } = txInfo['inner-txns'][0].txn;
      const { txn: algoTransfer1 } = txInfo['inner-txns'][1].txn;
      const { txn: xAlgoTransfer } = txInfo['inner-txns'][2].txn;

      // state after
      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
      const {
        totalPendingStake,
        totalActiveStake,
        totalRewards,
        totalUnclaimedFees,
      } = state;
      expect(totalPendingStake).toEqual(oldTotalPendingStake);
      expect(totalActiveStake).toEqual(oldTotalActiveStake + mintAmount);
      expect(totalRewards).toEqual(oldTotalRewards + additionalRewards);
      expect(totalUnclaimedFees).toEqual(oldTotalUnclaimedFees + additionalRewardsFee);

      // balances after
      const { algoBalance, xAlgoCirculatingSupply, proposersBalances } = await getXAlgoRate();
      expect(algoBalance).toEqual(oldAlgoBalance + mintAmount);
      expect(xAlgoCirculatingSupply).toEqual(oldXAlgoCirculatingSupply + expectedReceived);
      expect(proposersBalances[0]).toEqual(oldProposersBalance[0] + excessMintAmount / BigInt(2) + BigInt(1));
      expect(proposersBalances[1]).toEqual(oldProposersBalance[1] + diffMintAmount + excessMintAmount / BigInt(2) - BigInt(1));
      expect(txInfo['inner-txns'].length).toEqual(3);
      expect(algoTransfer0.type).toEqual("pay");
      expect(algoTransfer0.amt).toEqual(Number(excessMintAmount / BigInt(2) + BigInt(1)));
      expect(algoTransfer0.snd).toEqual(decodeAddress(getApplicationAddress(xAlgoAppId)).publicKey);
      expect(algoTransfer0.rcv).toEqual(decodeAddress(proposer0.addr).publicKey);
      expect(algoTransfer1.type).toEqual("pay");
      expect(algoTransfer1.amt).toEqual(Number(diffMintAmount + excessMintAmount / BigInt(2) - BigInt(1)));
      expect(algoTransfer1.snd).toEqual(decodeAddress(getApplicationAddress(xAlgoAppId)).publicKey);
      expect(algoTransfer1.rcv).toEqual(decodeAddress(proposer1.addr).publicKey);
      expect(xAlgoTransfer.type).toEqual("axfer");
      expect(xAlgoTransfer.xaid).toEqual(Number(xAlgoId));
      expect(xAlgoTransfer.aamt).toEqual(Number(expectedReceived));
      expect(xAlgoTransfer.snd).toEqual(decodeAddress(getApplicationAddress(xAlgoAppId)).publicKey);
      expect(xAlgoTransfer.arcv).toEqual(decodeAddress(user1.addr).publicKey);   
           
    });
```

Run test case:

```
npm run test

```

Test results:

```log
immediate mint
      
      ✓ succeeds and splits between proposers (1320 ms)
```

Explanation:

The test code first register proposer\_0 offline.

But when user call "immediate\_mint" the contract still distributes the ALGO to node proposer\_0.

Full log for reference:

```log

npm run test  > test_all_241216_1400.log
(node:15045) ExperimentalWarning: The Fetch API is an experimental feature. This feature could change at any time
(Use `node --trace-warnings ...` to show where the warning was created)
PASS test/xAlgoConsensusV2.test.ts (84.908 s)
  Algo Consensus V2
    creation
      ✓ succeeds in updating from x algo consensus v1 to x algo consensus v2 (9331 ms)
    initialise
      ✓ succeeds for admin (67 ms)
      ✓ fails when already setup (39 ms)
    Update admin
      ✓ fails for invalid admin type (37 ms)
      ✓ admins can update admins (250 ms)
      ✓ non-admin cannot update admins (236 ms)
    add proposer
      ✓ fails for non register admin (177 ms)
      ✓ fails if proposer is not rekeyed (1064 ms)
      ✓ fails when proposer has already been added (37 ms)
      ✓ succeeds for register admin for second proposer (283 ms)
    update max proposer
      ✓ fails for non-admin (41 ms)
      ✓ succeeds for admin (81 ms)
    update premium
      ✓ fails for non-admin (37 ms)
      ✓ fails when premium is greater than 1% (38 ms)
      ✓ succeeds for admin (77 ms)
    pause minting
      ✓ fails for non-admin (37 ms)
      ✓ fails for invalid minting type (34 ms)
      ✓ succeeds for admin (196 ms)
    set proposer admin
      ✓ succeeds for register admin (1112 ms)
      ✓ fails for pending proposer admin (38 ms)
      ✓ succeeds for proposer admin (175 ms)
      ✓ fails for non register or proposer admin (157 ms)
      ✓ fails when proposer does not exist (38 ms)
    register online
      ✓ fails for non proposer admin (213 ms)
      ✓ fails when you don't send algo (138 ms)
      ✓ fails when proposer does not exist (1124 ms)
      ✓ succeeds for proposer admin (73 ms)
    register offline
      ✓ fails for non register admin (76 ms)
      ✓ fails when proposer does not exist (47 ms)
      ✓ succeeds for register and proposer admin (104 ms)
    subscribe to xgov
      ✓ fails for non xgov admin (145 ms)
      ✓ fails when proposer does not exist (67 ms)
      ✓ fails when don't send xgov fee (289 ms)
      ✓ succeeds for xgov admin (84 ms)
    unsubscribe from xgov
      ✓ fails for non xgov admin (1120 ms)
      ✓ fails when proposer does not exist (37 ms)
      ✓ succeeds for xgov admin (40 ms)
    immediate mint
      ✓ fails when immediate mint is paused (161 ms)
      ✓ fails when you don't send algo (158 ms)
      ✓ fails when proposer max balance is exceeded (111 ms)
      ✓ fails when you receive less x algo than min received specified (111 ms)
      ✓ succeeds and allocates to lowest balance proposer (198 ms)
      ✓ succeeds and splits between proposers (1320 ms)
    delayed mint
      ✓ fails when delay mint is paused (147 ms)
      ✓ fails when you don't send algo (131 ms)
      ✓ fails when proposer max balance is exceeded (101 ms)
      ✓ fails when nonce is not 2 bytes (65 ms)
      ✓ succeeds (1182 ms)
      ✓ fails when box is already used (125 ms)
    claim delayed mint
      ✓ fails when nonce is not 2 bytes (41 ms)
      ✓ fails when box does not exist (67 ms)
      ✓ fails when 320 rounds hasn't passed (69 ms)
      ✓ succeeds (26546 ms)
    burn
      ✓ fails when you don't send x algo (110 ms)
      ✓ fails when you receive less algo than min received specified (108 ms)
      ✓ succeeds and allocates from highest balance proposer (186 ms)
      ✓ succeeds and splits between proposers (1295 ms)
    update fee
      ✓ fails for non-admin (38 ms)
      ✓ fails when fee is greater than 100% (38 ms)
      ✓ succeeds for admin (280 ms)
    claim fee
      ✓ succeeds (211 ms)
    update smart contract
      ✓ fails in smart contract update when nothing scheduled (1338 ms)
      ✓ succeeds in scheduling update (645 ms)
      ✓ succeeds in overriding and scheduling update (1620 ms)
      ✓ fails in scheduling update when not admin (634 ms)
      ✓ fails in smart contract update when not past scheduled timestamp (1496 ms)
      ✓ fails in smart contract update when not admin (617 ms)
      ✓ succeeds in smart contract update (1351 ms)

Test Suites: 1 passed, 1 total
Tests:       68 passed, 68 total
Snapshots:   0 total
Time:        85.494 s
Ran all test suites.
```
