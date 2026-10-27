# #37807 \[SC-Insight] Truncation of mint\_amount to zero leading to potential stake loss

**Submitted on Dec 16th 2024 at 13:53:22 UTC by @danvinci\_20 for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37807
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The calculation of the mint\_amount in the current implementation of the claim\_delay\_mint can be truncated to zero if the numerator= (delay\_mint\_stake× get\_x\_algo\_circulating\_supply) is less than the denominator the current algo\_balance, this leading to total loss of value especially by small stakes this is possible since the algo\_balance can increase over time due to the rewards while the x\_algo\_citculating\_supply remains constant,

## Vulnerability Details

This is the current implented functionality it doesn't check for zero-value mint amount and this can lead to loss of value, also the mint\_amount is integer type variables and cannot store floating values

```
@router.method(no_op=CallConfig.CALL)
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
        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) + delay_mint_stake),
        update_total_rewards_and_unclaimed_fees(),
        # calculate mint amount
        algo_balance.store(
            App.globalGet(total_active_stake_key)
            + App.globalGet(total_rewards_key)
            - delay_mint_stake
            - App.globalGet(total_unclaimed_fees_key)
        ),
        mint_amount.store(
            If(
                algo_balance.load(),
                mul_scale(delay_mint_stake, get_x_algo_circulating_supply(), algo_balance.load()),
                delay_mint_stake
            )
        ),
        # send xALGO to user
        mint_x_algo(mint_amount.load(), receiver.get()),
        # delete box so cannot claim multiple times
        Assert(BoxDelete(box_name)),
        # give box min balance to sender as incentive
        InnerTxnBuilder.Begin(),
        get_transfer_inner_txn(Global.current_application_address(), Txn.sender(), get_app_algo_balance(), Int(0)),
        InnerTxnBuilder.Submit(),
        # log so can retrieve info for claiming
        Log(Concat(
            MethodSignature("ClaimDelayedMint(byte[36],address,uint64,uint64)"),
            box_name,
            receiver.get(),
            Itob(delay_mint_stake),
            Itob(mint_amount.load()),
        )),
    )
```

The code can be resolved by adding the following line :

```
Assert(mint_amount.load())
```

This ensure that the execution is reverted for zero amount mint\_value to ensure that value are not loss by users most especially small stakers, since the box that contains the minting details is deleted afterwards

## Impact details

Likelihood: Moderate (it occurs most times when staked values are small). Impact: High it results in complete loss of stake for affected users

## Resolution

Another implementation is to set a minimum value on the amount that can be staked in the implementation of the delay\_mint functionality to prevent users from staking very small values.

## Proof of Concept

## Proof of Concept

```
# Example values
delay_mint_stake = 100
x_algo_circulating_supply = 10000000000
algo_balance = 1000000000001  

# Calculate the mint_amount
numerator = delay_mint_stake * x_algo_circulating_supply
denominator = algo_balance

# Perform the integer division (mimicking the truncation behavior in PyTeal)
mint_amount = numerator // denominator

# Print the results
print(f"Numerator: {numerator}")
print(f"Denominator: {denominator}")
print(f"Calculated mint_amount (before truncation): {numerator / denominator}")
print(f"mint_amount (after truncation): {mint_amount}")

# Check the result
if mint_amount == 0:
    print("The mint_amount was truncated to 0, causing a loss of the stake.")
else:
    print("mint_amount is valid and greater than zero.")
```
