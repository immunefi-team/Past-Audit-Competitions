# #39507 \[BC-Critical] Insufficient validation on ClaimReward transaction allows attacker to claim an inflated reward OR prevent all nodes from being rewarded

**Submitted on Jan 31st 2025 at 13:28:55 UTC by @throwing5tone7 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39507
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

The processing of an internal transaction called ClaimReward does not sufficiently validate it's input, which allows\
at least two outcomes (described below) that lose funds from the network. The\
rewards should be calculated from the time the node joins the network to the time it leaves. Additionally,\
once a node has been rewarded, no subsequent reward will be processed for the same period. Due to missing\
validation on the parameters of the message, an attacker can cause one of two outcomes:

1. A malicious validator can send a transaction that claims a massively inflated reward on behalf of their node - which must be an active validator - e.g. a reward of \~ 500 million SHM is obtainable
2. A non-staked unknown attacker can send a transaction that claims a zero reward on behalf of any victim node (and can repeat this across every node account they know of) - this blocks the node from claiming the reward they are due for the time they are actively validating

Both outcomes lead to loss of funds - either rewards created inappropriately or rewards inappropriately blocked\
from being earned.

## Vulnerability Details

The root cause vulnerability is a lack of sanity checking on the `nodeDeactivatedTime` of the transaction,\
which will be discussed first, but the exploit is made much easier and more widely applicable by\
a failure to correctly validate other fields of the message, and a lack of authorization, described further on.

Before we get to the vulnerabilities a quick refresher on how rewards are calculated, to help understand the vulnerabilities.

### Context: How rewards are calculated

Here is a code snippet from the `applyClaimRewardTx` function, showing the most important points of how rewards are\
calculated in order to frame the discussion. Starting from <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/claimReward.ts#L230>

```
  /* eslint-enable security/detect-object-injection */
  const currentRate = _base16BNParser(network.current.nodeRewardAmountUsd) //BigInt(Number('0x' +
  const rate = nodeAccount.rewardRate > currentRate ? nodeAccount.rewardRate : currentRate
  const nodeRewardAmount = scaleByStabilityFactor(rate, network)
  const nodeRewardInterval = BigInt(network.current.nodeRewardInterval)

  if (nodeAccount.rewardStartTime < 0) {
    nestedCountersInstance.countEvent('shardeum-staking', `applyClaimRewardTx fail rewardStartTime < 0`)
    shardus.applyResponseSetFailed(
      applyResponse,
      `applyClaimReward failed because rewardStartTime is less than 0`
    )
    return
  }

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
  //update total reward var so it can be logged
  rewardedAmount = rewardedAmount / nodeRewardInterval
```

The most important points are:

* The `rewardedAmount` is based on the amount of time spent in the network (`durationInNetwork`)
* The fields `nodeAccount.nodeRewardStart` (previously captured as the node activation time when processing the `InitRewardTimes` transaction) and `tx.nodeDeactivatedTime` are used to determine the amount of time spent in the network, so they are critical
* `durationInNetwork` cannot be negative, i.e. `tx.nodeDeactivatedTime` must be >= `nodeAccount.nodeRewardStart`
* `nodeAccount.nodeRewardStart` cannot be zero or reward will be zero - which means an `InitRewardsTime` transaction must previously have set it for that node account (normally achieved by that node becoming an active validator)
* `nodeAccount.rewarded` cannot be true - and this function in fact sets it later, blocking repeated rewards for the same period of validation activity

### Missing checks on nodeDeactivatedTime

The root cause vulnerability is a lack of validation on the `nodeDeactivatedTime` - critical for calculating the reward\
amount due. The only hard constraint on `nodeDeactivatedTime` appears in `claimReward.validateClaimRewardTx` in <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/claimReward.ts#L110>

```
  if (tx.nodeDeactivatedTime <= 0) {
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateClaimRewardTx fail tx.duration <= 0`)
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateClaimRewardTx fail tx.duration <= 0', tx)
    return { isValid: false, reason: 'duration must be > 0' }
  }
```

(NOTE: although the validateClaimRewardState function fails if `nodeAccount.rewardEndTime >= nodeDeactivatedTime` this only affects\
repeated rewards, since the rewardEndTime is only set by claiming a reward)

The check above allows two forms of abuse, leading the two outcomes I discussed:

1. Claim that the node was deactivated at some point far in the future (e.g. in thousands of years' time) - this benefits the nominator account of the reward
2. Claim that the node was deactivated at the exact same time it was activated - this penalises the node account by setting their reward to zero and disallowing any update to the reward for that time period (rewarded is set to true)

### Enabling bug 1: missing check that deactivatedNodeId relates to nominee

On top of the root cause bug there are also issues that enable the bug to exploited more easily.\
The `ClaimReward` transaction has several inter-related fields:

* `deactivatedNodeId` - the node ID of the node that is due a reward
* `nominee` - the public key of the account for the node that is due a reward
* `nominator` - the public key of the account that staked the node

These should all relate to each other, in so far as a staker's EOA account is a nominator to a node account that has\
both a public key and an ID in the network. However, the validation functions do not enforce that the node's public key\
and ID are referring to the same entity, i.e. `nominee` can specify a public key related to a node account and `deactivatedNodeId` can specify an ID of another node.

This enables exploits because `deactivatedNodeId` is used to check that the node is currently not active in the network - the system is supposed\
to block rewards until the node leaves the network. This is verified by the following check - <https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/tx/claimReward.ts#L120>

```
  if (shardus.getNode(tx.deactivatedNodeId)) {
    /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `validateClaimRewardTx fail node still active`)
    /* prettier-ignore */ if (ShardeumFlags.VerboseLogs) console.log('validateClaimRewardTx fail node still active', tx)
    return { isValid: false, reason: 'Node is still active' }
  }
```

Apart from this, the validation function just checks that the field is a string of 64 characters. Given that this ID is\
not constrained to relate to a real account in any way, the attacker can just supply garbage for this ID and pass the\
validation check. The impact is that the nominator account being rewarded can be the account for an active node in the network.\
This means that the attacker does not have any time sensitivity to send the claim reward. If it wasn't for this they need\
to time their `ClaimReward` transaction to a time just after the node leaves the network but before the network\
automatically processes the `ClaimReward` for that node. My experiments show this is still feasible, at least for the\
first attack outcome (inflating the reward for an attacker-controlled node). But in the presence of this enabling bug,\
the attacker can just send a garbage\
node ID, so they can send the attacking transactions at any time, and the exploit becomes much easier and more reliable.

### Enabling feature 2: no authorization

Due to the automated nature of the current `ClaimReward` processing system, a lack of authorization on who sends the\
transaction is a feature rather than the bug (the transaction must be sent by another node that notices the deactivated node\
left the network). However, I would like to point out that the fact that anyone that can sign a message can create\
this transaction enables completely unknown attackers to send them, making my second attack outcome easier to apply\
(and making the PoC of the first outcome easier to explain & script).

### Mitigation recommendation

I would recommend the following:

1. Constrain `nodeDeactivatedTime` to not be in the future, e.g. check `nodeDeactivatedTime < shardeumGetTime() / 1000` (accounting for milliseconds to seconds conversion)
   * This completely prevents attack outcome 1
2. I assume it is non-trivial to directly check whether a node ID has ever been active, just based on the ID, but from what I have seen you can check whether a public key relates to any active node - if you do this you can remove the redundant `deactivatedNodeId` field and truly enforce that claims are not processed for active node using the public key field
   * This will make the timing for attack outcome 2 very difficult to achieve
3. Consider restricting the accounts that can send the `ClaimReward` to be a currently active node (which seems to be the intention)
   * This at least means you know the identity of the attacker, potentially motivating them to not run the reward-blocking attack
4. Consider any other sanity checks you can make on `nodeDeactivatedTime` - e.g. reach a quorum on when other nodes think that node deactivated, you could do this in a similar way to how you generate penalty transactions (where a node won't apply the penalty unless it already calculated the exact same TX data itself)

## Impact Details

### Attack outcome 1 - malicious staker / validator inflates their own reward

Clearly by sending a massive `nodeDeactivatedTime` far in the future the attacker can generate a large reward for\
themselves. In my PoC I send `2000000000000` as an example, which is more than 1.9 trillion seconds in the future at\
the time of writing. So when the reward is 1 SHM per hour, i.e. 1 SHM per 3600 seconds, as per the defaults in the repo,\
this leads to a reward of \~ 500 million SHM. This would be a direct loss to the network. The attacker only needs their\
node to be active and validating for a small amount of time (e.g. 5 minutes while they run the attack).

### Attack outcome 2 - unknown attacker blocks rewards for all nodes

The alternative outcome is to send a small `nodeDeactivatedTime` to force reward to zero for a range of victim\
nodes. Because the nodes are not correctly checked to be active, the attacker can send this at any time between the node\
activation and when they claim the reward. Because subsequent rewards for the same period will be blocked, (due to `rewarded` being set to true)\
the attacker has stopped any victim nodes from earning their just rewards, causing a loss of funds.

By sending a message for every node account the attacker\
ever sees, they can block all nodes from receiving rewards, shutting down the reward processing of the network.

Although the node can potentially send a new `InitRewardTimes` message to recover they will have to start a new reward\
period (a later start time) to be accepted, so they have lost funds due to that point, and the attacker can just keep\
repeating their attack.

## Link to Proof of Concept

<https://gist.github.com/throwin5tone7/f9fc8b4a469a11a423907826a33ac9d1>

## Proof of Concept

## Proof of concept

### PoC for outcome 1 - attacker massively inflates own node reward

Video - <https://youtu.be/V2P-h9I6lDI>

In this proof of concept I demonstrate that the attacker can affect the node account of a node they staked to\
claim a massively inflated reward (e.g. \~ 500 million SHM). I focus mainly on this PoC here, as I feel it the more\
compelling loss out of the two outcomes described in the report,\
but I also include instructions below (without a video) for a PoC of the second outcome.

In this attack setting, the attacker would typically be a validator, who knows the private key of the node account they\
staked, and they could make use of that to sign the transaction as the node,\
and could maliciously modify their own validator code to launch the attack. However, here I have kept the attack as a separate\
PoC script that doesn't make use of the validator private key, just to make the script easier to use and understand as a PoC.

#### PoC setup

This PoC requires a network of validators, where the attacker has staked on behalf of an active node that is due to earn rewards. You can\
see from my video that the only changes I need to make to a clean copy of shardeum is to adjust the node counts and\
rotation settings and fund a few accounts with known keys in genesis.json (I only need one to act as\
staker/attacker).

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
11. Stake on behalf of the yet-to-be-started attacker node (from the validator-cli folder), i.e. `operator-cli stake 10` and enter the attacker/staker account private key e.g. `139d047987e84d19691d4f8049e051d0dc6c87f45bb35ba50c89cd503c5b4b52`
12. Launch an attacker node that uses this stake `operator-cli start`
13. Take down a normal node in the network, i.e. in shardeum repo folder run `shardus stop-net 1` - this gives us a slot for the attacker node to join
14. Wait for attacker node (controlled by validator-cli) to be active in the network, e.g. watch `operator-cli status` until state says `active`

### PoC execution

Once this state of the setup is achieved, the PoC is demonstrated by the following steps:

1. In a fresh folder, download my `package.json` and `poc-claim-reward-inflating.js` script
2. Set this to use node 18.16.1 or equivalent, e.g. `fnm use 18.16.1`
3. `npm install` to fetch dependencies
4. Choose a node that will be used to inject the transaction (any running one will do, I normally choose one that isn't the attacker's node), e.g. `localhost:9002`
5. Double-check the account has been staked and the `rewardStartTime` has been set legitimately, e.g. `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewardStartTime` is set to a sensible time in seconds, e.g. similar order of magnitude to 1738227733
6. Run the PoC, e.g. assuming `localhost:9050` is the attacker node, run `node poc-claim-reward-inflating.js localhost:9050 localhost:9002`
   * This script is very simple, it just creates an `ClaimReward` transaction and injects it to the processing node (last argument), the transaction fields are set as:
     * `nominee` is set to the attacker node's public key
     * `nominator` is set to the public key of the known attacker/staker EOA account who staked the node (e.g. the public key corresponding to secret key `139d047987e84d19691d4f8049e051d0dc6c87f45bb35ba50c89cd503c5b4b52` if following from above)
     * `deactivatedNodeId` is set to a garbage ID of the correct length - `25d1cf0d9b06d24e81e33683688d65f003d0da9c06182b1ab35978b1051cf454` - it is important that this is not an active node ID or the transaction is rejected
     * `nodeDeactivatedTime` is set to an arbitrarily large value, far in the future in seconds, `2000000000000`
       * This leads to a massive duration in the network, leading to a reward of \~ 500 million SHM
     * `timestamp` is set to `Date.now() - 1000`, i.e. 1 second before the current time in milliseconds
       * A time that is slightly earlier than now is used in case of clock drift against the shardeum nodes, shardeum will accept earlier times than current but not later
     * `cycle` can be set to any positive integer, I use `25`
   * If the script is successful you should see the message "Injection successful" with a txId
7. Wait for tx to be processed (e.g. 1m) and rerun the account check `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewarded` is set to true (meaning that a reward has been calculated and an attempt to claim another reward will be blocked) and `reward` is set to a very large number (e.g. > 500M SHM, depending on exact reward settings of the network account)
   * In the rare case that after waiting a few minutes this value has not been updated, just repeat the attack

### PoC for outcome 2 - nodes blocked from receiving awards

In this proof of concept I demonstrate that the attacker can affect the node account of an active staked node to\
claim a reward of zero, blocking them from taking further rewards. As discussed in the report this\
has the effect that the targetted node does not receive the reward they are entitled to for their validation efforts, losing funds they are entitled to.

My PoC does not demonstrate the attack against multiple nodes, but I don't see any reason why it would be\
more difficult - the attack is not timing sensitive - any time after the stake is created and before the node exits the\
network will work.

#### PoC setup

The PoC requires a network of validators, where one active node has a stake and is due to earn rewards. Additionally, the penalty settings should be left\
disabled in config (as it is in the `bugbounty` tag) because penalties can interfere with rewards,\
making it harder to conclusively show that forcing the reward to be claimed early has blocked further rewards while the node stays active.\
Similarly to the outcome 1 PoC, the only changes I need to make to a clean copy of shardeum is to adjust the node counts and\
rotation settings and fund a few accounts with known keys in genesis.json (I only need one to act as\
staker).

I'm sure the developers will have a way to achieve a similar set up, but the exact steps I followed to end up in the desired state are:

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
11. Stake on behalf of the yet-to-be-started victim node (from the validator-cli folder), i.e. `operator-cli stake 10` and enter one of the known private keys e.g. `139d047987e84d19691d4f8049e051d0dc6c87f45bb35ba50c89cd503c5b4b52`
12. Launch a victim node that uses this stake `operator-cli start`
13. Take down a normal node in the network, i.e. in shardeum repo folder run `shardus stop-net 1` - this gives us a slot for the victim node to join
14. Wait for victim node (controlled by validator-cli) to be active in the network, e.g. watch `operator-cli status` until state says `active`

#### PoC execution

Once this state of the setup is achieved, the PoC is demonstrated by the following steps:

1. In a fresh folder, download my `package.json` and `poc-claim-reward-blocking.js` script
2. Set this to use node 18.16.1 or equivalent, e.g. `fnm use 18.16.1`
3. `npm install` to fetch dependencies
4. Choose a node that will be used to inject the transaction (any running will do, I normally choose one that isn't the victim node), e.g. `localhost:9002`
5. Double-check the account has been staked and the `rewardStartTime` has been set legitmately, e.g. `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewardStartTime` is set to a sensible time in seconds, e.g. 1738227733
6. Run the PoC, e.g. assuming `localhost:9050` is the victim node, run `node poc-claim-reward-blocking.js localhost:9050 localhost:9002`
   * This script is very simple, it just creates an `ClaimReward` transaction and injects it to the processing node (last argument), the transaction fields are set as:
     * `nominee` is set to the victim node's public key
     * `nominator` is set to the public key of the staker who staked the victim (also a victim)
     * `deactivatedNodeId` is set to a garbage ID of the correct length - `25d1cf0d9b06d24e81e33683688d65f003d0da9c06182b1ab35978b1051cf454` - it is important that this is not an active node ID or the transaction is rejected
     * `nodeDeactivatedTime` is set to the same value as the `rewardStartTime` (the script calculates this by calling the `/account` endpoint for the victim node account)
       * This has the effect when calculating the reward that `rewardInterval` and hence the reward itself become zero
     * `timestamp` is set to `Date.now() - 1000`, i.e. 1 second before the current time in milliseconds
       * A time that is slightly earlier than now is used in case of clock drift against the shardeum nodes, shardeum will accept earlier times than current but not later
     * `cycle` can be set to any positive integer, I use `25`
7. Wait for tx to be processed (e.g. 1m) and rerun the account check `curl localhost:9002/account/7f5a8ce4f4b0aab3c6ad5356a9e97ef6036420cefe24ede81d0953165b5719cb?type=9`
   * You should see `rewarded` is set to true (meaning that a reward has been calculated and an attempt to claim another reward will be blocked) and `reward` is set to `0x0`
   * In the rare case that after waiting a few minutes this value has not been updated, just repeat the attack
8. Stop the victim node - automated processing should kick in after a few minutes to process the reward
9. After say 10m, check the account again, you should still see that rewarded and reward are set the same, indicating that it was not possible to generate any legitimate reward for the node account
