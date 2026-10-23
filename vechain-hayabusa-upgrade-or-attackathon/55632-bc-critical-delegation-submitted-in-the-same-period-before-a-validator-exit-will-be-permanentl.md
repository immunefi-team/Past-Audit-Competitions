# 55632 bc critical delegation submitted in the same period before a validator exit will be permanently frozen

**Submitted on Oct 3rd 2025 at 06:29:50 UTC by @Haxatron for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #55632
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/vechain/thor/tree/release/hayabusa>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

Any delegation that is submitted in the same period before a validator exit will be permanently frozen.

## Vulnerability Details

Any delegation that is submitted in the same period of a validator exit will be permanently frozen due to the following edge case. Consider the following scenario:

{% stepper %}
{% step %}

### Delegation added in Period 5

`AddDelegation` is called by a delegator in Period 5; its `FirstIteration` will be `CurrentIteration + 1` so, Period 6.

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker/staker.go#L534-L543>

```go
	current, err := val.CurrentIteration(currentBlock)
	if err != nil {
		return nil, err
	}
	delegationID, err := s.delegationService.Add(validator, current+1, stake, multiplier)
	if err != nil {
		logger.Info("failed to add delegation", "validator", validator, "error", err)
		return nil, err
	}
```

{% endstep %}

{% step %}

### Validator signals exit in Period 5

`SignalExit` is called by the validator in Period 5; its `CompletedPeriods` will be set to 5 periods.

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker/validation/service.go#L165-L185>

```go
func (s *Service) SignalExit(validator thor.Address, currentBlock uint32, minBlock uint32, maxTry int) error {
	...

	current, err := validation.CurrentIteration(currentBlock)
	if err != nil {
		return err
	}
	validation.ExitBlock = &exitBlock
	// validator is going to exit after current iteration
	validation.CompletedPeriods = current

	return s.repo.updateValidation(validator, validation)
}
```

{% endstep %}

{% step %}

### Exit processed at start of Period 6

On Period 6, the validator's exit is processed. This causes the `aggregation.Pending` to be set to 0 during the start of Period 6 when housekeeping is performed.

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker/aggregation/aggregation.go#L82-L95>

```go
func (a *Aggregation) exit() *globalstats.Exit {
	// Return these values to modify contract totals
	exit := globalstats.Exit{
		ExitedTVL:      a.Locked.Clone(),
		QueuedDecrease: a.Pending.VET,
	}

	// Reset the aggregation
	a.Exiting = &stakes.WeightedStake{}
	a.Locked = &stakes.WeightedStake{}
	a.Pending = &stakes.WeightedStake{}

	return &exit
}
```

{% endstep %}

{% step %}

### Delegator withdraws after exit — stuck due to underflow

When the delegator tries to withdraw via `WithdrawDelegation` after the exit, since the `CompletedPeriods` is set to 5 then `CurrentIteration` will return 5. Because the delegation's `FirstIteration` is set to 6, `started` will be false.

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker/delegation/delegation.go#L28-L38>

```go
func (d *Delegation) Started(val *validation.Validation, currentBlock uint32) (bool, error) {
	if val.Status == validation.StatusQueued || val.Status == validation.StatusUnknown {
		return false, nil // Delegation cannot start if the validation is not active
	}
	currentStakingPeriod, err := val.CurrentIteration(currentBlock)
	if err != nil {
		return false, err
	}
	return currentStakingPeriod >= d.FirstIteration, nil
}
```

Because `started` is false, the withdraw path attempts to decrement `aggregation.Pending` via `aggregationService.SubPendingVet`, but `aggregation.Pending` was already reset to 0 during exit. This causes an underflow and the transaction reverts.

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker/staker.go#L679-L688>

```go
func (s *Staker) WithdrawDelegation(
	delegationID *big.Int,
	currentBlock uint32,
) (uint64, error) {
	...
	if !started {
		weightedStake := stakes.NewWeightedStakeWithMultiplier(withdrawableStake, del.Multiplier)
		if err = s.aggregationService.SubPendingVet(del.Validation, weightedStake); err != nil {
			return 0, err
		}

		if err = s.globalStatsService.RemoveQueued(withdrawableStake); err != nil {
			return 0, err
		}
	} else {
	...
}
```

{% endstep %}
{% endstepper %}

## Impact Details

Any delegation that is submitted in the same period before a validator exit will be permanently frozen (any fix will require a hardfork).

## Proof of Concept

<details>

<summary>Show PoC test (add to hayabusa-e2e tests)</summary>

```go
package poc

import (
        "context"
        "testing"
        "time"

        "github.com/stretchr/testify/assert"
        "github.com/stretchr/testify/require"
        "github.com/vechain/hayabusa-e2e/hayabusa"
        "github.com/vechain/hayabusa-e2e/testutil"
        "github.com/vechain/hayabusa-e2e/utils"
        "github.com/vechain/thor/v2/thor"
        "github.com/vechain/thor/v2/thorclient/builtin"
)

func Test_Delegations(t *testing.T) {
        staker, config, validationIDs := newDelegationSetup(t)
        ticker := utils.NewTicker(staker.Raw().Client())

        t.Run("[POC] Delegations are added in the same period as validator exit", func(t *testing.T) {
                t.Parallel()

                validator, err := staker.GetValidation(validationIDs[0])
                require.NoError(t, err)
                validatorAccount := hayabusa.ValidatorAccounts[2]

                for _, acc := range hayabusa.ValidatorAccounts {
                        if acc.Node.Address().String() == validator.Address.String() {
                                validatorAccount = acc
                                break
                        }
                }

                // add the delegation
                receipt := testutil.Send(t, hayabusa.Stargate,
                        staker.AddDelegation(validationIDs[0], builtin.MinStake(), 100))
                require.NoError(t, err)
                delegationID1 := testutil.ReceiptToID(receipt)
                delegation1, err := staker.GetDelegation(delegationID1)
                require.NoError(t, err)
                assert.Equal(t, builtin.MinStake(), delegation1.Stake)
                assert.Equal(t, uint8(100), delegation1.Multiplier)
                assert.False(t, delegation1.Locked)

                // This line is commented out: This waits 1 period before the validator signals exit. Uncommenting will pass the test.
                // require.NoError(t, ticker.WaitForBlock(receipt.Meta.BlockNumber+config.MinStakingPeriod*1))

                receipt = testutil.Send(t, validatorAccount.Endorser, staker.SignalExit(validatorAccount.Node.Address()))

                // wait for validators last period to end
                require.NoError(t, ticker.WaitForBlock(receipt.Meta.BlockNumber+config.MinStakingPeriod*1))

                // withdraw - should revert
                testutil.Send(t, hayabusa.Stargate, staker.WithdrawDelegation(delegationID1))
        })
}

func newDelegationSetup(t *testing.T) (*builtin.Staker, *hayabusa.Config, [3]thor.Address) {
        t.Helper()
        config := &hayabusa.Config{
                Nodes:             3,
                MaxBlockProposers: 3,
                ForkBlock:         0,
                TransitionPeriod:  4,
                EpochLength:       4,
                CooldownPeriod:    4,
                MinStakingPeriod:  4,
                MidStakingPeriod:  12,
                HighStakingPeriod: 259200,
                Name:              t.Name(),
                BlockInterval:     uint64(3),
        }
        network, err := hayabusa.NewNetwork(config, t.Context())
        require.NoError(t, err)
        t.Cleanup(network.Stop)
        require.NoError(t, network.Start())

        staker, err := builtin.NewStaker(network.ThorClient())
        if err != nil {
                t.Fatalf("failed to create staker: %v", err)
        }
        if err := utils.WaitForFork(t.Context(), staker, config.ForkBlock); err != nil {
                t.Fatalf("failed to wait for fork: %v", err)
        }

        validationIDs := [3]thor.Address{}
        senders := &utils.Senders{}

        for i := range validationIDs {
                account := hayabusa.ValidatorAccounts[i]
                sender := staker.AddValidation(account.Node.Address(), builtin.MinStake(), config.MinStakingPeriod).
                        Send().
                        WithSigner(account.Endorser).
                        WithOptions(testutil.TxOptions())
                senders.Add(sender)
                validationIDs[i] = account.Node.Address()
        }

        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        t.Cleanup(cancel)

        if _, _, err := senders.Send(ctx); err != nil {
                t.Fatal(err)
        }
        if err := utils.WaitForPOS(t.Context(), staker, config.ForkBlock+config.TransitionPeriod); err != nil {
                t.Fatalf("failed to wait for PoS: %v", err)
        }

        return staker, config, validationIDs
}
```

</details>
