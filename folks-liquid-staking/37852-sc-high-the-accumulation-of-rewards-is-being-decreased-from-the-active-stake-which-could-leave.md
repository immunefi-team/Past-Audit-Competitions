# #37852 \[SC-High] The accumulation of rewards is being decreased from the active stake which could le

**Submitted on Dec 17th 2024 at 13:12:10 UTC by @dustykid for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37852
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The reward mechanism of the Folks finance protocol is of "share" type in which the liquid assets value is being increased from the accumulation of underlying asset in the vault/protocol. The way that users claim their portion of the rewards is by burning their xAlgo and receiving Algo in larger amount than their original deposit. The delta is their portion of the rewards. The problem is that `total_active_stake_key` is increasing only in size only for the initially deposited amount during minting while during burning is being decreased for the initially deposited amount + portion of rewards. This is leading to two problems with the contract implementation:

1. Rewards "eat up" the initial deposited amount of the users which could leave some of them unable to redeem the initially deposited Algo amount.
2. This also means that the rewards will be claimed. on 'first come - first serve' basis and could leave out some users without their portion fo the rewards due to `total_active_stake_key` not accounting for the rewards.

## Vulnerability Details

When users mint xAlgo either via `delayed_mint` or `immediate_mint` the `total_active_stake_key` is being increased for the Algo amount that they are depositing:

```
# update total active stake before total rewards as to not mistake new algo received for rewards
        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) + algo_sent),
```

Since the value of xAlgo increases with the accumulation of the rewards within the protocol whenever users would like to burn their xAlgo to receive Algo, The amount that they will receive is being calculated based on `total_active_stake_key` + `total_rewards_key`.

```
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
        )
```

while the `total_rewards_key` is being calculated from the difference between the `total_active_stake_key` and the Algo balance of the proposers.

```
new_total_rewards.store(get_proposers_algo_balance(Int(0)) - App.globalGet(total_pending_stake_key) - App.globalGet(total_active_stake_key)),
        App.globalPut(total_rewards_key, new_total_rewards.load()),
```

Users can redeem more Algo than they have initially deposited due to rewards accumulation. The problem arises during the burning of xAlgo where the `total_active_stake_key` is being decreased for the initially deposited amount + users portion of the rewards.

```
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
        # update total active stake
        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
```

## Impact Details

The mechanics described above will result in unjustly decreasing virtual balance of users because of the accumulation of rewards which could lead to some users being unable to redeem a portion or the entirety of their Algo deposit due to `total_active_stake_key` accounting for rewards claimed by other users while the `total_active_stake_key` is never being increased for the reward amount.

## Link to Proof of Concept

<https://gist.github.com/kaloyan-boyanov/6ae073753b682a31b4cc289d493fae5b>

## Proof of Concept

## Proof of Concept

Please get the code from the provided gist.

Paste it in a file with extension `.test.ts` in the repo `/test` directory. It can be run with the instructions for running tests from the protocol repo README.md
