# 56362 bc insight during addvalidation if pos not active authority native env state get should consume double the gas

**Submitted on Oct 15th 2025 at 03:26:07 UTC by @emarai for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56362
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>

## Description

### Brief / Intro

A commit on the `release/hayabusa` branch removed the gas charge `charger.Charge(thor.SloadGas)` before calling `Authority.Native(..).Get()`:

```diff
			if !isPoSActive {
-				charger.Charge(thor.SloadGas) // a.getEntry(ValidatorMaster)
+

				exists, endorser, _, _, err := Authority.Native(env.State()).Get(thor.Address(args.Validator))
```

However, gas still needs to be charged since `Authority.Native(..).Get()` is not charging it internally. A similar call in `authority_native.go` explicitly charges twice the amount of gas:

```go
		{"native_isEndorsed", func(env *xenv.Environment) []any {
			var nodeMaster common.Address
			env.ParseArgs(&nodeMaster)

			env.UseGas(thor.SloadGas * 2)
			listed, endorsor, _, _, err := Authority.Native(env.State()).Get(thor.Address(nodeMaster))
```

Relevant PR: <https://github.com/vechain/thor/commit/f08442278f1eb45bae74f1fe922955c8e2f1d367>

## Impact Details

The gas that was supposed to be charged (200 or 400 if following `authority_native.go`) is not charged in the code path shown, resulting in lower gas consumption than intended.

## References

* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/authority\\_native.go#L72>
* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/staker\\_native.go#L189>

{% stepper %}
{% step %}

### Reproduce (modify code)

Remove or comment out the call to `Authority.Native(env.State()).Get` in `builtin/staker_native.go` to demonstrate that skipping the call does not change gas consumption:

```diff
--- a/builtin/staker_native.go
+++ b/builtin/staker_native.go
@@ -186,7 +186,10 @@ func init() {
                        }

                        if !isPoSActive {
-                               exists, endorser, _, _, err := Authority.Native(env.State()).Get(thor.Address(args.Validator))
+                               // exists, endorser, _, _, err := Authority.Native(env.State()).Get(thor.Address(args.Validator))
+                               // note: skipping the call to show that no gas is charged
+                               exists := true
+                               endorser := thor.Address(args.Endorser)
                                if err != nil {
                                        return nil, err
                                }
```

{% endstep %}

{% step %}

### Run test

Execute the specific test to observe gas usage remains unchanged:

```bash
go test -timeout 30s -run ^TestStakerNativeGasCosts$ github.com/vechain/thor/v2/builtin
```

Expected outcome: No difference in gas consumption when the `Get` call is removed, indicating the `Authority.Native(env.State()).Get` did not consume gas in that code path.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
The fix should ensure that a gas charge is applied where appropriate (e.g., reintroduce the `charger.Charge(thor.SloadGas)` before calling `Authority.Native(...).Get()` or make the `Get` call charge gas internally) to match the behavior in `authority_native.go`.
{% endhint %}
