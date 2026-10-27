# 55711 sc insight redundant gas charge in native addvalidation function leads to unnecessary gas costs

**Submitted on Oct 4th 2025 at 10:32:50 UTC by @rionnaldi for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)

* **Report ID:** #55711
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker\\_native.go>

## Description

### Brief / Intro

The `native_addValidation` function within `builtin/staker_native.go` contains a redundant gas charge that causes users to pay higher transaction fees than necessary. When the Proof-of-Stake (PoS) consensus mechanism is not active, the function explicitly charges for a storage read (SloadGas) before calling a subsequent function that already accounts for its own storage access costs. This results in duplicated gas charges and wasted gas for the user.

## Vulnerability Details

{% stepper %}
{% step %}

### What the code does

When PoS is inactive (`if !isPoSActive`), the function:

* Explicitly charges `thor.SloadGas` to the transaction gas meter.
* Calls `Authority.Native(env.State()).Get(...)` to retrieve validator information.
  {% endstep %}

{% step %}

### Why this is redundant

The `Get` function itself reads from contract storage to fetch authority details. Its metered implementation charges the necessary gas for these SLOAD operations. Therefore the explicit `charger.Charge(thor.SloadGas)` performed prior to the `Get` call duplicates charging for the same storage read.
{% endstep %}
{% endstepper %}

## Impact Details

{% hint style="info" %}
Users incur a small but unnecessary extra gas cost (one SLOAD worth) whenever `native_addValidation` is invoked while the chain is not in its PoS phase. Although the waste per transaction is small, it is persistent and accumulates across many transactions.
{% endhint %}

## Proof of Concept

The issue appears in `builtin/staker_native.go`:

{% code title="builtin/staker\_native.go" %}

```go
	isPoSActive, err := Staker.NativeMetered(env.State(), charger).IsPoSActive()
	if err != nil {
		return nil, err
	}

	if !isPoSActive {
		// This gas charge is redundant. The subsequent call to Authority.Native.Get()
		// will account for its own SLOAD operations.
		charger.Charge(thor.SloadGas) // a.getEntry(ValidatorMaster)

		exists, endorser, _, _, err := Authority.Native(env.State()).Get(thor.Address(args.Validator))
		if err != nil {
			return nil, err
		}
		if !exists {
			return nil, staker.NewReverts("authority required in transition period")
		}
		if thor.Address(args.Endorser) != endorser {
			return nil, staker.NewReverts("endorser required")
		}
	}
```

{% endcode %}

The `Authority.Native(env.State()).Get()` call performs its own SLOAD operations and charges for them. The preceding `charger.Charge(thor.SloadGas)` duplicates that charge when `isPoSActive` is false.

## Recommendation

Remove the explicit `charger.Charge(thor.SloadGas)` in the `if !isPoSActive` block and rely on the metered `Authority.Native.Get()` call to account for storage access gas.

## References

* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker\\_native.go>
* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/gen/authority.sol>
* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/gascharger/gas\\_charger.go>
