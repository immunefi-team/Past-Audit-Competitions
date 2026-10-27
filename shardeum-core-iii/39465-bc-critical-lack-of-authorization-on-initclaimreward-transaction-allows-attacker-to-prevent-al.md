# #39465 \[BC-Critical] Lack of authorization on InitClaimReward transaction allows attacker to prevent all nodes from being rewarded

**Submitted on Jan 30th 2025 at 17:32:20 UTC by @throwing5tone7 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39465
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

The processing of an internal transaction called InitRewardTimes does not authorize the sender, so an attacker can send this message.\
By sending this message, an attacker can increase the `rewardStartTime` to a later time than it should be. The\
rewards should be calculated from the time the node joins the network to the time it leaves, but because the attacker\
can increase the recorded start time, they can maliciously reduce the window that the victim will be rewarded for, reducing\
the reward. By itself this would cause a loss of the funds that the node should have earned as rewards.\
However, because of a second bug, a unit confusion issue (seconds vs milliseconds), the attacker can in\
fact set this start time to a future time that will never be reached. This has the effect that the node will never\
earn a reward for its work (because the current time in seconds will never reach the end time specified in milliseconds). It\
is trivial for the attacker to repeat this process for all nodes in the network, leading to a loss of funds (the rewards\
that the nodes deserve for their processing work but will never receive) for all node accounts the attacker targets.

## Vulnerability Details

Please bear in mind that there are two bugs here - the lack of authorization **AND** the unit confusion. Even without\
the unit confusion, the lack of authorization would still be exploitable, but as I haven't seen a way to make the unit\
confusion exploitable in a profitable way by itself, I am combining these into a single report. When\
used together, the unit confusion makes the other bug easier to exploit in a way that guarantees maximum impact.\
However, in isolation the unit confusion is probably only an insight at best. I will describe\
each of the bugs and how they lead to loss separately.

### Lack of authorization

Most of the Shardeum internal transaction handling appropriately combines authentication and authorization, i.e.\
they verify the transaction signature (which checks that verifying `sign.sig` with the hash of the transaction yields\
the specified `sign.owner` public key of the transaction) and they also check that `sign.owner` is some expected public key (e.g. a\
public key of an active node). However, none of the various validation functions in `https://github.com/shardeum/shardeum/blob/bugbounty/src/tx/initRewardTimes.ts`\
checks the `sign.owner` field, meaning it will accept a transaction signed by anybody.

As an example, here is the `validateFields` function code - <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/initRewardTimes.ts#L80C1-L106C2>

```
export function validateFields(tx: InitRewardTimes, shardus: Shardus): { success: boolean; reason: string } {
  /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('Validating InitRewardTimesTX fields', tx)
  if (!tx.nominee || tx.nominee === '' || tx.nominee.length !== 64) {
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes fail invalid nominee field', tx)
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateFields InitRewardTimes fail invalid nominee field`)
    return { success: false, reason: 'invalid nominee field in setRewardTimes Tx' }
  }
  if (!tx.nodeActivatedTime) {
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes fail nodeActivatedTime missing', tx)
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateFields InitRewardTimes fail nodeActivatedTime missing`)
    return { success: false, reason: 'nodeActivatedTime field is not found in setRewardTimes Tx' }
  }
  if (tx.nodeActivatedTime < 0 || tx.nodeActivatedTime > shardeumGetTime()) {
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes fail nodeActivatedTime is not correct ', tx)
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateFields InitRewardTimes fail nodeActivatedTime is not correct `)
    return { success: false, reason: 'nodeActivatedTime is not correct in setRewardTimes Tx' }
  }
  const isValid = crypto.verifyObj(tx)
  if (!isValid) {
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes fail Invalid signature', tx)
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateFields InitRewardTimes fail Invalid signature`)
    return { success: false, reason: 'Invalid signature' }
  }

  /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes success', tx)
  return { success: true, reason: 'valid' }
}
```

What you can see here is that `crypto.verifyObj(tx)` is called, which gives authentication, but not in a way that\
provides authorization, i.e. the call is not of a form `crypto.verifyObj(tx, expectedPublicKey)`. And following on\
through the code, there are no other constraints on `tx.sign.owner` - which could check the transaction was sent by\
somebody that is allowed to take this action. The\
code is very similar in the `validate` and `validateInitRewardState` - both essentially make the same verify call without\
providing an expected public key and lack any\
other authorization logic on `tx.sign.owner`.

### Importance of reward start time

The reward start time is intended to record the time that the node was active from. This is saved to the node account\
and used to calculate how much SHM reward the node is entitled to when a `ClaimReward` transaction is processed for\
that node's account (normally this happens automatically when the node leaves).\
In the `apply` function for the InitRewardTimes transaction we see the update of the reward start\
time - <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/initRewardTimes.ts#L187>

`nodeAccount.rewardStartTime = tx.nodeActivatedTime`

And in the `apply` function of the `ClaimReward` transaction we see how this is used to calculate the reward - <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/claimReward.ts#L244>

```
  let durationInNetwork = tx.nodeDeactivatedTime - nodeAccount.rewardStartTime
  if (durationInNetwork < 0) {
    nestedCountersInstance.countEvent('shardeum-staking', `applyClaimRewardTx fail durationInNetwork < 0`)
    //throw new Error(`applyClaimReward failed because durationInNetwork is less than or equal 0`)
    shardus.applyResponseSetFailed(
      applyResponse,
      `applyClaimReward failed because durationInNetwork is less than 0`
    )
    return
  }

  // special case for seed nodes:
  // they have 0 rewardStartTime and will not be rewarded but the claim tx should still be applied
  if (nodeAccount.rewardStartTime === 0) {
    nestedCountersInstance.countEvent('shardeum-staking', `seed node claim reward ${nodeAccount.id}`)
    durationInNetwork = 0
  }

  if (nodeAccount.rewarded === true) {
    nestedCountersInstance.countEvent('shardeum-staking', `applyClaimRewardTx fail already rewarded`)
    //throw new Error(`applyClaimReward failed already rewarded`)
    shardus.applyResponseSetFailed(applyResponse, `applyClaimReward failed already rewarded`)
    return
  }

  nodeAccount.rewardEndTime = tx.nodeDeactivatedTime

  //we multiply fist then devide to preserve precision
  let rewardedAmount = nodeRewardAmount * BigInt(durationInNetwork * 1000) // Convert from seconds to milliseconds
```

Based on this we can see that both of the following are true:

* An attacker that can increase the `rewardStartTime` to a time after the node was activated reduces the value of `rewardedAmount` meaning they cause the node to lose some funds they should receive
* An attacker that can set the `rewardStartTime` to some far future time will block the node from ever being able to claim a reward (because in normal processing the `tx.nodeDeactivatedTime` will never reach the recorded start time) and so the `durationInNetwork < 0` condition will be met and the reward claim transaction will fail to be processed.

### Impact of the unit confusion

There are checks in the `validateFields` function that intend to prevent the second case from happening, i.e. the attacker\
should never be able to set `rewardStartTime` to a future time. Here is the relevant code from <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/initRewardTimes.ts#L92>

```
  if (tx.nodeActivatedTime < 0 || tx.nodeActivatedTime > shardeumGetTime()) {
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateFields InitRewardTimes fail nodeActivatedTime is not correct ', tx)
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateFields InitRewardTimes fail nodeActivatedTime is not correct `)
    return { success: false, reason: 'nodeActivatedTime is not correct in setRewardTimes Tx' }
  }
```

The unit confusion here is that `tx.nodeActivatedTime` and the `rewardStartTime` fields are intended to be in seconds\
(this is clear from the code line above that says `let rewardedAmount = nodeRewardAmount * BigInt(durationInNetwork * 1000) // Convert from seconds to milliseconds`).\
However, the `shardeumGetTime()` function returns a value in milliseconds. This gives the attacker the possibility\
to set a value that is lower than the `shardeumGetTime` result, but still far ahead of the current time in seconds.

For example, in my testing, I set a `rewardStartTime` of `1738228173681` which is an epoch time of "Thursday, 30 January 2025 09:09:33.681 GMT+00:00"\
whereas the epoch time in seconds is `1738228174`. When measured in seconds, we won't reach the epoch time of `1738228173681`\
until at least 1,730,000,000,000 seconds after this time, which is roughly 54,000 years in the future!\
Hence we can confidently say that an attacker who exploits this failing can set to the `rewardStartTime` to a value\
that can never be reached (and other sensible checks in the code prevent anybody from setting this time to an\
earlier value than already recorded, they can only increase the recorded value).

## Impact Details

From the above vulnerability description it is clear that - a malicious actor can affect the critical data of a node account in a way\
that means that node loses some of the reward it was entitled to. Furthermore, any user who can send a transaction can\
cause this effect. Without the unit confusion issue, the attacker could reduce the reward for any given node by sending\
a message with rewardStartTime set to the current time in seconds, and then the node will lose any reward amount they\
should have received for the processing since they became active. By repeating this attack until the claim reward transaction\
is processed the attacker could ensure the reward amount is very close to zero. However, they would have to tradeoff between\
the amount of messages sent, the number of nodes targetted, the reduction in rewards achieved per node, and the risk\
of detection.

But in combination with the unit confusion exploit, the attacker can send a single transaction per node-account to set the start time\
to a far future time that will never be reached. They can send this at any time after the node account has been created,\
which in my testing appears to be any time after the node is staked. By sending a message for every node account the attacker\
ever sees, they can block all nodes from receiving rewards, shutting down the reward processing of the network.

This leads to a loss of all funds that should be earned as rewards, totally negating the financial\
incentives to be a legitimate validator.

Because the code prevents setting the `rewardStartTime` to an earlier time than previously set, there isn't any recovery\
path other than manual intervention and deploying code fixes.

## Link to Proof of Concept

<https://gist.github.com/throwin5tone7/955fd9d5fa71b4f18351e4d59c04dece>

## Proof of Concept

## Proof of concept

Video - <https://youtu.be/yTTcdl6dEKY>

In this proof of concept I demonstrate that the attacker can affect the node account of a staked node to\
set a reward start time very far in the future that will not be reached. As discussed in the report this\
has the effect that the targetted node will never earn their reward, losing funds they are entitled to.

My PoC does not demonstrate the attack against multiple nodes, but I don't see any reason why it would be\
more difficult - the attack is not timing sensitive - any time after the stake is created and before the claim reward message\
is processed for a node will work.

### PoC setup

The PoC requires a network of validators, where one active node has a stake and is due to earn rewards. Additionally, the penalty settings should be left\
disabled in config (as it is in the `bugbounty` tag) because penalties can interfere with rewards,\
making it harder to conclusively show that changing the rewardStartTime has blocked rewards.\
You can\
see from my video that the only changes I need to make to a clean copy of shardeum is to adjust the node counts and\
rotation settings and fund a few accounts with known keys in genesis.json (I only need one to act as\
staker).

I'm sure the developers will have a way to achieve a similar set up, but the exact steps I followed to end up in the state shown in the video are:

1. Download shardeum repo and checkout bugbounty tag
2. Apply my config changes `git apply shardeum.patch`
3. `npm ci` and `npm run prepare` to get everything up to date
4. Launch a network of 10 nodes `shardus start 10` (I need a network up and running to process the staking TX)
5. Download validator-cli repo and link to shardeum folder e.g. `ln -s "$(cd ../LEGIT-shardeum && pwd)" ../validator`
6. Apply config changes for server to run against local net `git apply validator-cli.patch`
7. `npm ci && npm link` to get things hooked up
8. `npm run compile`
9. Wait for network to reach `processing` mode
10. Launch a JSON RPC server
11. Stake on behalf of the yet-to-be-started validator CLI node, i.e. `operator-cli stake 10` and enter one of the known private keys e.g. `139d047987e84d19691d4f8049e051d0dc6c87f45bb35ba50c89cd503c5b4b52`
12. Launch a node that uses this stake `operator-cli start`
13. Take down a node in the network, i.e. in shardeum repo folder run `shardus stop-net 1` - this gives us a slot for the victim node to join
14. Wait for node ran by validator-cli to be active in the network, e.g. watch `operator-cli status` until state says `active`

### PoC execution

Once this state of the setup is achieved, the PoC is demonstrated by the following steps:

1. In a fresh folder, download my `package.json` and `poc-init-reward.js` script
2. Set this to use node 18.16.1 or equivalent, e.g. `fnm use 18.16.1`
3. `npm install` to fetch dependencies
4. Choose a node that will be used to inject the transaction (any running will do, I normally choose one that isn't the validator-cli controlled node), e.g. `localhost:9002`
5. Double-check the account has been staked and the `rewardStartTime` has been set legitmately, e.g. `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewardStartTime` is set to a sensible time in seconds, e.g. 1738227733
6. Run the PoC, e.g. assuming `localhost:9050` is the victim node, run `node poc-init-reward.js localhost:9050 localhost:9002`
   * This script is very simple, it just creates an `InitRewardTimes` transaction and injects it to the processing node (last argument)
   * `nominee` on the transaction is set to the victim node's public key
   * `timestamp` and `nodeActivatedTime` on the transaction are both set to `Date.now() - 1000`, i.e. 1 second before the current time in milliseconds
     * A time that is slightly earlier than now is used in case of clock drift against the shardeum nodes, shardeum will accept earlier times than current but not later
7. Wait for tx to be processed (e.g. 1m) and rerun the account check `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewardStartTime` is set to a non-sensical time in seconds, around 1000x it's old value
   * In the rare case that after waiting a few minutes this value has not been updated, just repeat the attack
8. Stop the victim node - automated processing should kick in after a few minutes to process the reward
9. After say 10m, check the account again, you should see `rewarded: false` indicating that it was not possible to reward the node account
