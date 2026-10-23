# 56946 bc insight the code comparing two big in pointers for equality not their numeric values

**Submitted on Oct 22nd 2025 at 02:11:03 UTC by @jesse03 for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56946
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

The code compares two *big.Int* pointers for equality, not their numeric values:

```go
currentBGP, err := builtin.Params.Native(newState).Get(thor.KeyLegacyTxBaseGasPrice)
if err != nil {
    return errors.WithMessage(err, "failed to get the current base gas price")
}
if currentBGP == baseGasPrice {  // <-- BUG: pointer comparison, almost never true
    return nil
}
```

In Go, `==` on `*big.Int` checks whether the addresses are the same, not whether the integers hold the same value. Since `currentBGP` and `baseGasPrice` are different heap objects, this condition will essentially always be false — even when the on‑chain base gas price already equals `1e13`.

Impact: the node will always craft and try to pack a privileged `Params.set` transaction on startup, needlessly mutating (or attempting to mutate) chain parameters and consuming resources. Given this is signed with a dev key and built with an extremely long expiration, it’s a governance‑level logic error.

## Proof of Concept

<details>

<summary>Test PoC (expand to view)</summary>

{% code title="poc\_test.go" %}

```go
func TestInitSolo_PoC_NoTxWhenBasePriceAlreadySet(t *testing.T) {
	solo := newSolo()

	// First init should set base gas price to 1e13 and mine 1 block
	startNum := solo.repo.BestBlockSummary().Header.Number()
	require.NoError(t, solo.init(context.Background()))
	afterFirst := solo.repo.BestBlockSummary().Header.Number()
	require.Equal(t, startNum+1, afterFirst)

	// Verify base gas price is 1e13
	best := solo.repo.BestBlockSummary()
	st := solo.stater.NewState(best.Root())
	v, err := builtin.Params.Native(st).Get(thor.KeyLegacyTxBaseGasPrice)
	require.NoError(t, err)
	require.Equal(t, baseGasPrice, v)

	// Second init SHOULD be a no-op if values are equal (correct behavior)
	require.NoError(t, solo.init(context.Background()))
	afterSecond := solo.repo.BestBlockSummary().Header.Number()

	// Expected: no new block; Actual (bug): afterSecond == afterFirst+1
	require.Equal(t, afterFirst, afterSecond)
}
```

{% endcode %}

</details>
