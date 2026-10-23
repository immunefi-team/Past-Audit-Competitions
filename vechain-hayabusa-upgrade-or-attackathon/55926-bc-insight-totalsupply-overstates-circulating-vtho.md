# 55926 bc insight totalsupply overstates circulating vtho

**Submitted on Oct 7th 2025 at 20:42:53 UTC by @spongebob for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #55926
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

`energy.TotalSupply()` returns the lazily grown base supply plus cumulative issuance and never subtracts burned amounts.

Relevant code:

<https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy/energy.go#L108-L134>

```go
func (e *Energy) TotalSupply() (*big.Int, error) {
	initialSupply, err := e.getInitialSupply()
	if err != nil {
		return nil, err
	}

	// calc grown energy for total token supply
	acc := state.Account{
		Balance:   initialSupply.Token,
		Energy:    initialSupply.Energy,
		BlockTime: initialSupply.BlockTime,
	}

	// this is a virtual account, use account.CalcEnergy directly
	stopTime, err := e.GetEnergyGrowthStopTime()
	if err != nil {
		return nil, err
	}
	grown := acc.CalcEnergy(e.blockTime, stopTime)

	issued, err := e.getIssued()
	if err != nil {
		return nil, err
	}

	return grown.Add(grown, issued), nil
}
```

Burns are only tracked by incrementing `total.TotalSub` inside `Energy.Sub` (used by gas spending and fees):

<https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy/energy.go#L180-L205>

<https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/runtime/resolved\\_tx.go#L146-L212>

The net burned amount is exposed via `TotalBurned()`:

<https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy/energy.go#L136-L144>

```go
func (e *Energy) TotalBurned() (*big.Int, error) {
	total, err := e.getTotalAddSub()
	if err != nil {
		return nil, err
	}
	burned := new(big.Int).Sub(total.TotalSub, total.TotalAdd)
	return burned, nil
}
```

Because `issued` is only increased (for example, block rewards call `e.addIssued(reward)`) and never decreased, `TotalSupply()` reports cumulative minted tokens rather than minted minus burned. After any net burn, `sum(balanceOf)` < `totalSupply()`.

The public ERC‑20 wrapper simply forwards `totalSupply()` to the native value, so on-chain consumers expecting standard semantics receive the inflated figure unless they manually subtract `TotalBurned()`:

<https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/gen/energy.sol#L25-L27>

```solidity
function totalSupply() public view returns(uint256) {
    return EnergyNative(this).native_totalSupply();
}
```

The native energy module misreports supply, so the network layer exposes incorrect contract data. While nothing is stolen immediately, downstream contracts can misbehave because of the incorrect figure; this is classified as MEDIUM severity.

## Proof of Concept (reproduction)

Add the following test to `energy_test.go` and run.

{% stepper %}
{% step %}

### Seed an account with energy

Create a state and seed an account with 1 000 VTHO:

```go
st := state.New(muxdb.NewMem(), trie.Root{})

acc := thor.BytesToAddress([]byte("burn"))
if err := st.SetEnergy(acc, big.NewInt(1000), 0); err != nil {
    t.Fatalf("failed to seed energy: %v", err)
}

p := params.New(thor.BytesToAddress([]byte("par")), st)
eng := New(thor.BytesToAddress([]byte("eng")), st, 0, p)

assert.NoError(t, eng.SetInitialSupply(big.NewInt(1), big.NewInt(1000)))
```

{% endstep %}

{% step %}

### Check total supply before burn

```go
totalBefore, err := eng.TotalSupply()
assert.NoError(t, err)
assert.Equal(t, big.NewInt(1000), totalBefore)
```

{% endstep %}

{% step %}

### Burn energy

```go
ok, err := eng.Sub(acc, big.NewInt(600))
assert.NoError(t, err)
assert.True(t, ok)

balance, err := eng.Get(acc)
assert.NoError(t, err)
assert.Equal(t, big.NewInt(400), balance)
```

{% endstep %}

{% step %}

### Observe totalSupply() unchanged while TotalBurned() tracks burns

```go
totalAfter, err := eng.TotalSupply()
assert.NoError(t, err)
assert.Equal(t, totalBefore, totalAfter)
assert.True(t, totalAfter.Cmp(balance) > 0)

burned, err := eng.TotalBurned()
assert.NoError(t, err)
assert.Equal(t, big.NewInt(600), burned)
```

The test demonstrates that after burning 600 VTHO, `TotalSupply()` still reports 1 000 while `TotalBurned()` reports 600 and the account balance is 400, proving the supply overstatement.
{% endstep %}
{% endstepper %}
