# 56187 bc insight brittle hardcoded gas metering model

**Submitted on Oct 12th 2025 at 12:42:33 UTC by @rionnaldi for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)

* **Report ID:** #56187
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>

## Description

### Brief / Intro

All native functions in this file use a static, hardcoded gas metering model. The Go code makes an explicit `env.UseGas()` call with a manually pre-calculated gas cost. This pattern is brittle and a significant source of long-term maintenance risk.

### Vulnerability Details

This design tightly couples the gas accounting logic (in the `_native.go` wrapper) to the business logic (in the underlying `authority.go` implementation). If a developer refactors `Authority.Native(...).Get()` to use one or three `SLOAD`s instead of two, they must remember to find and update this `UseGas` call. Forgetting to do so will cause the operation to be mis-priced, leading to a potential gas-based DoS vector.

This is an inferior pattern compared to the dynamic `gascharger` model used in the `Staker` contract, where the `gascharger` object is passed down and gas is consumed dynamically as storage operations occur. The `gascharger` model correctly decouples accounting from logic.

### Impact Details

This pattern makes refactoring and maintaining code more error-prone because developers need to remember and update the `UseGas` call whenever underlying storage or computation changes.

## Recommendation

Create a long-term plan to refactor the `Authority`, `Energy`, and other older built-in contracts to use the `gascharger` model. Suggested migration steps:

{% stepper %}
{% step %}

### Create a metered constructor

Create a `NativeMetered` constructor for the `Authority` contract that accepts a `*gascharger.Charger`.
{% endstep %}

{% step %}

### Pass a charger into core logic

Pass the charger into the core logic functions (`Add`, `Revoke`, `Get`, etc.) so the core can consume gas as operations (e.g., SLOAD, SSTORE) are performed.
{% endstep %}

{% step %}

### Remove hardcoded UseGas calls

Remove all hardcoded `env.UseGas()` calls and have the underlying storage abstractions charge gas dynamically via the charger.
{% endstep %}
{% endstepper %}

## Benefit

* Reduces Risk of Bugs: Decouples gas logic from business logic, making it impossible to forget to update gas costs during a refactor.
* Improves Maintainability: Simplifies the code and makes it easier and safer to modify.
* Standardizes Architecture: Creates a single, robust gas accounting model across all built-in contracts.

## References

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/authority\\_native.go>

<https://github.com/vechain/thor/blob/release/hayabusa/builtin/staker\\_native.go>

## Proof of Concept

`builtin/authority_native.go`:

{% code title="authority\_native.go" %}

```go
		{"native_get", func(env *xenv.Environment) []any {
			var nodeMaster common.Address
			env.ParseArgs(&nodeMaster)

			env.UseGas(thor.SloadGas * 2) // Manually calculated gas cost
			listed, endorsor, identity, active, err := Authority.Native(env.State()).Get(thor.Address(nodeMaster))
			}
		}
```

{% endcode %}

List of files using this pattern: `authority_native.go`, `energy_native.go`, `extension_native.go`, `params_native.go`, `prototype_native.go`
