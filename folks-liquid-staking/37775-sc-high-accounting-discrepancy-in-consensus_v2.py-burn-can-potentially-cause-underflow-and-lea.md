# #37775 \[SC-High] Accounting Discrepancy in \`consensus\_v2.py::burn()\`can potentially cause underflow

**Submitted on Dec 15th 2024 at 14:13:38 UTC by @Oxbakeng for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37775
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `burn()` function in `consensus_v2.py` contains a flaw in how it updates `the total_active_stake` which can accumulate discrepancies over time. This leads to situations where users cannot burn their entire xAlgo holdings, potentially locking funds indefinitely and opening a door for an adversary to orchestrate a DOS attack.

## Vulnerability Details

To see how the vulnerability is introduced we first need to understand the relationship between minting xAlgo and burning xAlgo and how the value of xAlgo grows due to the rewards distributed in the proposer's accounts.

When users `immediate_mint()` for the first time, the circulating supply of xAlgo and amount of Algo being held in the contract are considered to calculate the value of how much xAlgo to allocate to user based on how much Algo the user is sending into the contract->proposer's accounts, then the proportional amount of xAlgo in value is sent into the caller's address while the Algo is sent to the proposer's accounts, during this process the amount of Algo is incremented to the total\_active\_stake\_key at that base value, the problem is introduced as more rewards are distributed to the proposer's accounts with only a small number of stakers and a relatively low circulating supply, due to the increase in Algo the value will increase anyway as "rewards" for stakers, so if a large number of stakers burn their xAlgo in order to get the rewards/more Algo per xAlgo, during the `burn()` function call, `algo_to_send` is always calculated by taking the total\_rewards\_key in to account thus leading to more Algo subtracted from total\_active\_stake\_key than was put in.

```
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
```

Now when `total_active_stake_key` is decremented, more than what was initially put in is decremented, which WILL eventually lead to a deficit and lead to an underflow when algo\_to\_send is higher than what is stored in `total_active_stake_key`, the function will revert leading to temporary frozen funds up to an hour or more.

The vulnerability is primarily located at: `App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load())`

In the `burn()` function:

```
@router.method(no_op=CallConfig.CALL)
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
        # update total active stake
        App.globalPut(total_active_stake_key, App.globalGet(total_active_stake_key) - algo_to_send.load()),
        # log burn
        Log(Concat(
            MethodSignature("Burn(address,uint64,uint64)"),
            Txn.sender(),
            Itob(burn_amount),
            Itob(algo_to_send.load()),
        )),
    )
```

This update method:

1. Temporary DoS: If a user attempts to burn a significant amount of xAlgo, particularly when there are few stakers or when one staker holds a disproportionately large amount, the total\_active\_stake might fall below the algo\_to\_send, leading to transaction reversion due to underflow.
2. Scalability Issue: There's no maximum burn amount check, which means large burns can exacerbate the discrepancy between total\_active\_stake and the actual balance of proposers.
3. Exploitation Risk: An adversary who noticed the flaw with enough ALGO could mint a large amount of xAlgo, wait for the reward cycle to increase the xAlgo's value, then burn, further widening the gap between total\_active\_stake and proposer balances. This is because:

* When tokens are minted, the total\_active\_stake is updated based on the current ALGO value.
* Over time, as rewards (which increase the value of xAlgo) are distributed into proposer balances, burning xAlgo results in more ALGO being returned than what was added to total\_active\_stake at the time of minting.
* Each burn subtracts the full amount of ALGO returned, including the increased value from rewards, from total\_active\_stake, thus widening the discrepancy.

## Impact Details

* Temporary Denial of Service: Users might temporarily be unable to burn their entire holding of xAlgo if the total\_active\_stake\_key is insufficient due to this accounting error which will result in the new total\_active\_stake\_key entry being a negative number which would cause an underflow and revert the burn function call.
* Potential Exploitation: An attacker could exploit this by manipulating the mint and burn cycles to increase the discrepancy, potentially causing more frequent DoS scenarios or manipulating the token economics to their benefit.
* The Adversary can always ensure that whales that have staked large sums are always in a situation where they cannot burn their entire holding if need be.

As funds can be frozen/unable to burn entire amount up to an hour or more depending on external factors, as per the scope, this qualifies as a High vulnerabilty.

Note: While the contract can be upgraded/updated to mitigate this issue or by the protocol team minting xAlgo tokens to recalibrate total\_active\_stake, without a fix, the problem will reoccur mutiple times down the line as the accounting debt will keep catching up, especially under certain conditions or with specific user behaviors.

## References

<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/main/contracts/xalgo/consensus\\_v2.py>

## Proof of Concept

Please add the describe block in the project test file

```
  describe("Accounting debt in total_active_stakes_key", () => {
    test("The burn call fails if total combined proposer balance can satisfy burn but total active stake is lower than algo to be sent to user", async () => {
      let canProposersSatisfyTogether = false;
      let canTotalActiveStakeSatisfy = false;
      let totalAvailableFromProposers = BigInt(0);
      let expectedReceived = BigInt(0);
  
      try {
        const additionalRewards = BigInt(10e6);
        console.log('\nInitial Setup:');
        console.log('Additional Rewards:', additionalRewards.toString(), 'microAlgos');
        await fundAccountWithAlgo(algodClient, proposer1.addr, additionalRewards, await getParams(algodClient));
        const additionalRewardsFee = mulScale(additionalRewards, fee, ONE_4_DP);
        console.log('Additional Rewards Fee:', additionalRewardsFee.toString(), 'microAlgos');
        
        let initialState = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        console.log('\nInitial Global State:');
        console.log('Total Active Stake:', initialState.totalActiveStake.toString(), 'microAlgos');
        console.log('Total Pending Stake:', initialState.totalPendingStake.toString(), 'microAlgos');
        console.log('Total Rewards:', initialState.totalRewards.toString(), 'microAlgos');
        console.log('Total Unclaimed Fees:', initialState.totalUnclaimedFees.toString(), 'microAlgos');
    
        const smallerBurnAmount = BigInt(70e6);
        console.log('\nFirst Burn Parameters (user2):');
        console.log('Burn Amount:', smallerBurnAmount.toString(), 'microAlgos');
    
        const { algoBalance: preFirstBurnAlgoBalance, 
                xAlgoCirculatingSupply: preFirstBurnXAlgoSupply, 
                proposersBalances: preFirstBurnProposersBalances } = await getXAlgoRate();
        
        console.log('\nPre-First-Burn State:');
        console.log('Algo Balance:', preFirstBurnAlgoBalance.toString(), 'microAlgos');
        console.log('xAlgo Circulating Supply:', preFirstBurnXAlgoSupply.toString());
        console.log('Proposer0 Balance:', preFirstBurnProposersBalances[0].toString(), 'microAlgos');
        console.log('Proposer1 Balance:', preFirstBurnProposersBalances[1].toString(), 'microAlgos');
    
        const firstBurnTxns = prepareBurnFromXAlgoConsensusV2(
          xAlgoConsensusABI, 
          xAlgoAppId, 
          xAlgoId, 
          user2.addr, 
          smallerBurnAmount, 
          BigInt(0), 
          [proposer0.addr, proposer1.addr], 
          await getParams(algodClient)
        );
        
        await submitGroupTransaction(algodClient, firstBurnTxns, firstBurnTxns.map(() => user2.sk));
        console.log('\nFirst burn successful');
    
        let postFirstBurnState = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        console.log('\nPost-First-Burn Global State:');
        console.log('Total Active Stake:', postFirstBurnState.totalActiveStake.toString(), 'microAlgos');
        console.log('Total Pending Stake:', postFirstBurnState.totalPendingStake.toString(), 'microAlgos');
        console.log('Total Rewards:', postFirstBurnState.totalRewards.toString(), 'microAlgos');
        console.log('Total Unclaimed Fees:', postFirstBurnState.totalUnclaimedFees.toString(), 'microAlgos');
    
        const { algoBalance: oldAlgoBalance, 
                xAlgoCirculatingSupply: oldXAlgoCirculatingSupply, 
                proposersBalances: oldProposersBalance } = await getXAlgoRate();
        
        console.log('\nPre-Second-Burn Rate Calculation Values:');
        console.log('Algo Balance:', oldAlgoBalance.toString(), 'microAlgos');
        console.log('xAlgo Circulating Supply:', oldXAlgoCirculatingSupply.toString());
        console.log('Proposer0 Balance:', oldProposersBalance[0].toString(), 'microAlgos');
        console.log('Proposer1 Balance:', oldProposersBalance[1].toString(), 'microAlgos');
        
        const burnAmount = BigInt(100e6);
        console.log('\nSecond Burn Parameters (user1):');
        console.log('Burn Amount:', burnAmount.toString(), 'microAlgos');
        
        const minReceived = BigInt(0);
        expectedReceived = mulScale(burnAmount, oldAlgoBalance, oldXAlgoCirculatingSupply);
        console.log('Expected Received:', expectedReceived.toString(), 'microAlgos');
        
        const totalProposersBalance = oldProposersBalance[0] + oldProposersBalance[1];
        console.log('\nProposer Calculations:');
        console.log('Total Proposers Balance:', totalProposersBalance.toString(), 'microAlgos');
        
        const numProposers = BigInt(2);
        const targetBalance = (totalProposersBalance - expectedReceived) / numProposers;
        console.log('Target Balance per Proposer:', targetBalance.toString(), 'microAlgos');
        
        console.log('\nDistribution Analysis:');
        const proposer0Available = oldProposersBalance[0] > targetBalance ? 
                                 oldProposersBalance[0] - targetBalance : BigInt(0);
        const proposer1Available = oldProposersBalance[1] > targetBalance ? 
                                 oldProposersBalance[1] - targetBalance : BigInt(0);
        
        totalAvailableFromProposers = proposer0Available + proposer1Available;
        
        console.log('Proposer0 Can Provide:', proposer0Available.toString(), 'microAlgos');
        console.log('Proposer1 Can Provide:', proposer1Available.toString(), 'microAlgos');
        console.log('Total Available from Both:', totalAvailableFromProposers.toString(), 'microAlgos');
        console.log('Required Amount:', expectedReceived.toString(), 'microAlgos');
        
        canProposersSatisfyTogether = totalAvailableFromProposers >= expectedReceived;
        
        console.log('\nCombined Proposer Analysis:');
        console.log('Can Proposers Together Satisfy Request?', canProposersSatisfyTogether);
        
        const initialStateForStakeCheck = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
        canTotalActiveStakeSatisfy = initialStateForStakeCheck.totalActiveStake >= expectedReceived;
        
        console.log('\nTotal Active Stake Analysis:');
        console.log('Total Active Stake:', initialStateForStakeCheck.totalActiveStake.toString(), 'microAlgos');
        console.log('Required Amount:', expectedReceived.toString(), 'microAlgos');
        console.log('Can Total Active Stake Satisfy Request?', canTotalActiveStakeSatisfy);
  
        if (!canProposersSatisfyTogether) {
          console.log('\nInsufficient combined balance across all proposers');
          console.log('Required:', expectedReceived.toString(), 'microAlgos');
          console.log('Available:', totalAvailableFromProposers.toString(), 'microAlgos');
        }
    
        const proposerAddrs = [proposer0.addr, proposer1.addr];
        const txns = prepareBurnFromXAlgoConsensusV2(
          xAlgoConsensusABI, 
          xAlgoAppId, 
          xAlgoId, 
          user1.addr, 
          burnAmount, 
          minReceived, 
          proposerAddrs, 
          await getParams(algodClient)
        );
        
        if (!canProposersSatisfyTogether || !canTotalActiveStakeSatisfy) {
          console.log("Expecting transaction to fail due to total_active_stakes_key being lower than algo_to_send which causes underflow and reverts");
          
          await expect(submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk)))
            .rejects.toThrow();
          
          return;
        }
        
        await submitGroupTransaction(algodClient, txns, txns.map(() => user1.sk));
        console.log("Transaction succeeded");
        
      } catch (error) {
        if (error instanceof Error) {
          console.log('\nError Details:', error.message);
          
          const isInsufficientFundsError = 
            error.message.includes('underflow on subtracting') || 
            error.message.includes('insufficient balance');
          
          if ((!canProposersSatisfyTogether || !canTotalActiveStakeSatisfy) && isInsufficientFundsError) {
            console.log('Transaction correctly rejected due to insufficient funds');
            console.log('Total Available from Proposers:', totalAvailableFromProposers.toString(), 'microAlgos');
            
            const finalStateForStakeCheck = await parseXAlgoConsensusV2GlobalState(algodClient, xAlgoAppId);
            console.log('Total Active Stake:', finalStateForStakeCheck.totalActiveStake.toString(), 'microAlgos');
            
            console.log('Required Amount:', expectedReceived.toString(), 'microAlgos');
            return;
          }
        }
        throw error;
      }
    });
  });
```
