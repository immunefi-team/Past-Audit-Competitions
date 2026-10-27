# #37889 \[SC-High] Underflow in \`burn()\` function will cause user funds to partially frozen

**Submitted on Dec 18th 2024 at 10:46:52 UTC by @ruhum for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37889
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In `burn()`, the contract subtracts the amount of ALGO to send from the `total_active_stake_key`. The amount of ALGO can be bigger than `total_active_stake_key` causing an underflow.

## Vulnerability Details

In `burn()` it subtracts the amount of ALGO that's sent to the user from `total_active_stake_key`:

```py
App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
```

`total_active_stake_key` is the sum of all the ALGO deposits made by users. When the user redeems their xALGO they get a little more ALGO back than they initially deposited because of the rewards earned by the proposers. That surplus amount of ALGO is not included in `total_active_stake_key` causing the total xALGO converted to ALGO to be bigger than `total_active_stake_key` which in turn will cause an underflow in certain situations.

`algo_to_send` is calculated as:

```sol
        algo_to_send.store(
            mul_scale(
                burn_amount,
                algo_balance.load(),
                get_x_algo_circulating_supply() + burn_amount
            )
        ),
```

Here, `algo_balance` is the total ALGO balance of the proposers (deposits + rewards).

This only applies to very large depositors or the last users to redeem their xALGO.

## Impact Details

A small subset of user funds will be frozen and not recoverable.

## References

<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L824>

## Proof of Concept

## Proof of Concept

Following test can be copied into `xAlgoConsensusV2.test.ts` under `describe("burn")`:

```ts
      test("issue", async () => {
        // airdrop rewards
        const additionalRewards = BigInt(10e6);
        await fundAccountWithAlgo(algodClient, proposer1.addr, additionalRewards, await getParams(algodClient));
        const additionalRewardsFee = mulScale(additionalRewards, fee, ONE_4_DP);

        const proposerAddrs = [proposer0.addr, proposer1.addr];

        // we burn all the XAlgo from user1 and user2. That should cause `total_active_stake_key` to underflow.
        const user1XAlgoBalance = await getAssetBalance(algodClient, user1.addr, xAlgoId);
        let txns = prepareBurnFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user1.addr, user1XAlgoBalance, 0, proposerAddrs, await getParams(algodClient));
        let [, txId] = await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));

        const user2XAlgoBalance = await getAssetBalance(algodClient, user2.addr, xAlgoId);
        txns = prepareBurnFromXAlgoConsensusV2(xAlgoConsensusABI, xAlgoAppId, xAlgoId, user2.addr, user2XAlgoBalance, 0, proposerAddrs, await getParams(algodClient));
        [, txId] = await submitGroupTransaction(algodClient, txns, txns.map(() => user2.sk));

        /*
          error:
          URLTokenBaseHTTPError: Network request error. Received status 400 (Bad Request): TransactionPool.Remember: transact
          ion YZRHNW5M2NOI6MNZQX5YKVG7K7OITD73SSK6U5HOG5PX4MVAKADQ: logic eval error: - would result negative. Details: app=1010,
          pc=2969, opcodes=app_global_get; load 42; -
        */
      });
```
