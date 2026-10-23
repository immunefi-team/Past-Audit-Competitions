# #37660 \[SC-High] incorrect tracking of \`TOTAL\_ACTIVE\_STAKE\` leads to permanent freezing of funds

**Submitted on Dec 11th 2024 at 21:56:47 UTC by @A2Security for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37660
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

There is a flow in the internal accounting for tracking the total active amount of staked Algo in the state variable `TOTAL_ACTIVE_STAKE`, this state variable is only incremented by the amount of algo deposited (increased in `claim_delayed_mint()` and `immediate_mint()`) when minting xAlgo.

The bug however arises from the fact that in the `burn()` function we reduce the `TOTAL_ACTIVE_STAKE` by the amount of the algo recieved from burning the xAlgo.

To simplify the bug:

* User A deposits 100 ALGO recieves 100 xALGO, TOTAL\_ACTIVE\_STAKE = 100 ALGO
* After a year the user, accrues some rewards 10 ALGO so the XALGO he owns is now worth 110 ALGO
* when the user will try to burn hi 100 xAlgo, he will fail because in burn() we will try to reduce TOTAL\_ACTIVE\_STAKE (100 ALGO) by 110 ALGO which will underflow. (10 ALGO are locked in the contract)

## Vulnerability Details

The vulnerability arises from the fact that Total Active Stake is not tracked correctly leading to an undersestimation which wil result in an underflow in the `burn()` function.

The TOTAL\_ACTIVE\_STAKE is only updated in the `immediate_mint()` and `claim_delayed_mint()` functions.

```python3
def claim_delayed_mint(receiver: abi.Address, nonce: abi.StaticBytes[L[2]]) -> Expr:
    box_name = Concat(DelayMintBox.NAME_PREFIX, receiver.get(), nonce.get())
    box = BoxGet(box_name)

    delay_mint_receiver = Extract(box.value(), DelayMintBox.RECEIVER, Int(32))
    delay_mint_stake = ExtractUint64(box.value(), DelayMintBox.STAKE)
    delay_mint_round = ExtractUint64(box.value(), DelayMintBox.ROUND)

    algo_balance = ScratchVar(TealType.uint64)
    mint_amount = ScratchVar(TealType.uint64)

    return Seq(
        # callable by anyone
        rekey_and_close_to_check(),
        # ensure initialised
        Assert(App.globalGet(initialised_key)),
        # check nonce is 2 bytes
        Assert(Len(nonce.get()) == Int(2)),
        # check box
        box,
        Assert(box.hasValue()),
        Assert(receiver.get() == delay_mint_receiver),
        Assert(Global.round() >= delay_mint_round),
        # update total stake and total rewards
        App.globalPut(total_pending_stake_key, App.globalGet(total_pending_stake_key) - delay_mint_stake),
@>>        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) + delay_mint_stake),
```

The problem however arises because we don't increase the `TOTAL_ACTIVE_STAKE` when we sync rewards and unclaimed fees for the protocol. This leads to underestimating the TOTAL\_ACTIVE\_STAKE by the accumalted amounts

This will lead to the actual withdrawable assets of users to be significantly higher than the value stored in the state variable `TOTAL_ACTIVE_STAKE`. Which will lead to an underflow in the `burn()` function

```python3
def burn(send_xalgo: abi.AssetTransferTransaction, min_received: abi.Uint64) -> Expr:
    burn_amount = send_xalgo.get().asset_amount()
    algo_balance = ScratchVar(TealType.uint64)
    algo_to_send = ScratchVar(TealType.uint64)

    return Seq(
        rekey_and_close_to_check(),
        # ensure initialised
        Assert(App.globalGet(initialised_key)),
        # check xALGO sent
        check_x_algo_sent(send_xalgo),
        # update total rewards
        update_total_rewards_and_unclaimed_fees(),
        # calculate algo amount to send
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
        # check amount and send ALGO to user
        Assert(algo_to_send.load()),
        Assert(algo_to_send.load() >= min_received.get()),
       send_algo_from_proposers(Txn.sender(), algo_to_send.load()),
        # @audit underflow here
@>>        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
```

N.B algo to

## Impact Details

* Permanent locked funds for the user who withdraws last.

A simplified example to showcase this vulnerability. 2 users A and B.

* User A deposits 100 ALGO recieves 100 xALGO, TOTAL\_ACTIVE\_STAKE = 100 ALGO w8 a year and his 100 xAlgo will be worth 110 ALGO
* User B deposits 10 ALGO recieves \~8 xALGO, TOTAL\_ACTIVE\_STAKE = 110 ALGO (100 from User A and 10 from User B)
* User A burns his xAlgo (worth 110 ALGO), TOTAL\_ACTIVE\_STAKE = 110 - 110 = 0 ALGO
* User B can't withdraw his ALGO because when he tries to withdraw reducing 10 from 0 will underflow.

## Proof of Concept

## Proof of Concept

xpected Result:

```log
> algo-liquid-staking-contracts@0.0.1 test
> PYTHONPATH='./contracts' jest --runInBand

  console.log
    totoal pending stake 0n

      at Object.log (test/xAlgoConsensusV2.test.ts:1590:15)

  console.log
    total active stake 182352954n

      at Object.log (test/xAlgoConsensusV2.test.ts:1591:15)

  console.log
    total unclaimed fees 0n

      at Object.log (test/xAlgoConsensusV2.test.ts:1592:15)

  console.log
    0n 182352954n 63000000n 0n

      at Object.log (test/xAlgoConsensusV2.test.ts:1593:15)

  console.log
    proposers algo balance 245352954n

      at Object.log (test/xAlgoConsensusV2.test.ts:1597:15)

  console.log
    unclaimable balance 63000000n

      at Object.log (test/xAlgoConsensusV2.test.ts:1599:15)
```

Please add the following modifications:

```sh
diff --git a/test/xAlgoConsensusV2.test.ts b/test/xAlgoConsensusV2.test.ts
index 662887b..563887e 100644
--- a/test/xAlgoConsensusV2.test.ts
+++ b/test/xAlgoConsensusV2.test.ts
@@ -1528,6 +1528,45 @@ describe("Algo Consensus V2", () => {
   });
 
   describe("claim fee", () => {
+    // test("succeeds", async () => {
+    //   // airdrop 10 ALGO rewards (%fee of which will be claimable by admin)
+    //   const additionalRewards = BigInt(10e6);
+    //   await fundAccountWithAlgo(algodClient, proposer0.addr, additionalRewards / BigInt(2), await getParams(algodClient));
+    //   await fundAccountWithAlgo(algodClient, proposer1.addr, additionalRewards / BigInt(2), await getParams(algodClient));
+    //   const additionalRewardsFee = mulScale(additionalRewards, fee, ONE_4_DP);
+
+    //   // balances before
+    //   const adminAlgoBalanceB = await getAlgoBalance(algodClient, admin.addr);
+    //   const { algoBalance: oldAlgoBalance, xAlgoCirculatingSupply: oldXAlgoCirculatingSupply, proposersBalances: oldProposersBalance } = await getXAlgoRate();
+
+    //   // state before
+    //   let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
+    //   const { totalRewards: oldTotalRewards, totalUnclaimedFees: oldTotalUnclaimedFees } = state;
+
+    //   // claim fee
+    //   const proposerAddrs = [proposer0.addr, proposer1.addr];
+    //   const tx = prepareClaimXAlgoConsensusV2Fee(xAlgoConsensusABI, xAlgoAppId, user1.addr, admin.addr, proposerAddrs, await getParams(algodClient));
+    //   const txId = await submitTransaction(algodClient, tx, user1.sk);
+    //   const txInfo = await algodClient.pendingTransactionInformation(txId).do();
+    //   const { txn: transfer } = txInfo['inner-txns'][txInfo['inner-txns'].length - 1].txn;
+    //   state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
+    //   expect(state.totalRewards).toEqual(oldTotalRewards + additionalRewards - (oldTotalUnclaimedFees + additionalRewardsFee));
+    //   expect(state.totalUnclaimedFees).toEqual(BigInt(0));
+
+    //   // balances after
+    //   const adminAlgoBalanceA = await getAlgoBalance(algodClient, admin.addr);
+    //   const { algoBalance, xAlgoCirculatingSupply, proposersBalances } = await getXAlgoRate();
+    //   expect(adminAlgoBalanceA).toEqual(adminAlgoBalanceB + oldTotalUnclaimedFees + additionalRewardsFee);
+    //   expect(algoBalance).toEqual(oldAlgoBalance);
+    //   expect(xAlgoCirculatingSupply).toEqual(oldXAlgoCirculatingSupply);
+    //   expect(proposersBalances[0] + proposersBalances[1]).toEqual(oldProposersBalance[0] + oldProposersBalance[1] - (oldTotalUnclaimedFees + additionalRewardsFee));
+    //   expect(proposersBalances[0] - BigInt(1)).toBeLessThanOrEqual(proposersBalances[1]);
+    //   expect(proposersBalances[0] + BigInt(1)).toBeGreaterThanOrEqual(proposersBalances[1]);
+    //   expect(transfer.type).toEqual("pay");
+    //   expect(transfer.amt).toEqual(Number(oldTotalUnclaimedFees + additionalRewardsFee));
+    //   expect(transfer.snd).toEqual(decodeAddress(getApplicationAddress(xAlgoAppId)).publicKey);
+    //   expect(transfer.rcv).toEqual(decodeAddress(admin.addr).publicKey);
+    // });    
     test("succeeds", async () => {
       // airdrop 10 ALGO rewards (%fee of which will be claimable by admin)
       const additionalRewards = BigInt(10e6);
@@ -1537,11 +1576,7 @@ describe("Algo Consensus V2", () => {
 
       // balances before
       const adminAlgoBalanceB = await getAlgoBalance(algodClient, admin.addr);
-      const { algoBalance: oldAlgoBalance, xAlgoCirculatingSupply: oldXAlgoCirculatingSupply, proposersBalances: oldProposersBalance } = await getXAlgoRate();
 
-      // state before
-      let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
-      const { totalRewards: oldTotalRewards, totalUnclaimedFees: oldTotalUnclaimedFees } = state;
 
       // claim fee
       const proposerAddrs = [proposer0.addr, proposer1.addr];
@@ -1549,24 +1584,21 @@ describe("Algo Consensus V2", () => {
       const txId = await submitTransaction(algodClient, tx, user1.sk);
       const txInfo = await algodClient.pendingTransactionInformation(txId).do();
       const { txn: transfer } = txInfo['inner-txns'][txInfo['inner-txns'].length - 1].txn;
-      state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
-      expect(state.totalRewards).toEqual(oldTotalRewards + additionalRewards - (oldTotalUnclaimedFees + additionalRewardsFee));
-      expect(state.totalUnclaimedFees).toEqual(BigInt(0));
 
+      let state = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
+      const { totalRewards, totalUnclaimedFees,totalActiveStake, totalPendingStake } = state;
+      console.log("totoal pending stake", totalPendingStake);
+      console.log("total active stake", totalActiveStake);
+      console.log("total unclaimed fees", totalUnclaimedFees);
+      console.log(totalPendingStake, totalActiveStake, totalRewards, totalUnclaimedFees);
       // balances after
-      const adminAlgoBalanceA = await getAlgoBalance(algodClient, admin.addr);
-      const { algoBalance, xAlgoCirculatingSupply, proposersBalances } = await getXAlgoRate();
-      expect(adminAlgoBalanceA).toEqual(adminAlgoBalanceB + oldTotalUnclaimedFees + additionalRewardsFee);
-      expect(algoBalance).toEqual(oldAlgoBalance);
-      expect(xAlgoCirculatingSupply).toEqual(oldXAlgoCirculatingSupply);
-      expect(proposersBalances[0] + proposersBalances[1]).toEqual(oldProposersBalance[0] + oldProposersBalance[1] - (oldTotalUnclaimedFees + additionalRewardsFee));
-      expect(proposersBalances[0] - BigInt(1)).toBeLessThanOrEqual(proposersBalances[1]);
-      expect(proposersBalances[0] + BigInt(1)).toBeGreaterThanOrEqual(proposersBalances[1]);
-      expect(transfer.type).toEqual("pay");
-      expect(transfer.amt).toEqual(Number(oldTotalUnclaimedFees + additionalRewardsFee));
-      expect(transfer.snd).toEqual(decodeAddress(getApplicationAddress(xAlgoAppId)).publicKey);
-      expect(transfer.rcv).toEqual(decodeAddress(admin.addr).publicKey);
+      
+      const { algoBalance } = await getXAlgoRate();
+      console.log("proposers algo balance", algoBalance );
+
+      console.log("unclaimable balance", algoBalance - totalActiveStake)
     });
+    
   });
 
   describe("update smart contract", () => {
```
