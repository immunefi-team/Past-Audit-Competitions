# 56513 bc insight during the call to native issuance there s a missing gas charge before call to calculaterewards&#x20;

**Submitted on Oct 17th 2025 at 06:55:44 UTC by @emarai for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56513
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:** Modification of transaction fees outside of design parameters

## Description

### Brief/Intro

The call to `staker_native::native_issuance` is followed by a call to `Energy.Native(env.State(), env.BlockContext().Time).CalculateRewards(staker)`, which reads from storage. Any read from storage should be preceded by `charger.Charge(thor.SloadGas)`; otherwise the call to `native_issuance` is cheaper than it actually needs to be.

## Vulnerability Details

In `builtin/staker_native.go` line 445 the `native_issuance` implementation calls `CalculateRewards` without a prior gas charge:

```go
{"native_issuance", func(env *xenv.Environment) ([]any, error) {
    charger := gascharger.New(env)

    staker := Staker.NativeMetered(env.State(), charger)
@>    issuance, err := Energy.Native(env.State(), env.BlockContext().Time).CalculateRewards(staker)
    if err != nil {
        return nil, err
    }
    return []any{issuance}, nil
}}
```

The `CalculateRewards` function in `builtin/energy/energy.go` (around line 341) performs storage reads such as `e.params.Get(thor.KeyCurveFactor)` and `staker.LockedStake()`. Each of these storage reads should incur `thor.SloadGas`. Because these reads are not preceded by a gas charge in the native call, the native call is undercharged. The gas charged should be 400 instead of 200 (see PoC).

Relevant snippet from `CalculateRewards`:

```go
func (e *Energy) CalculateRewards(staker staker) (*big.Int, error) {
@>    totalStaked, _, err := staker.LockedStake()
    if err != nil {
        return nil, err
    }
    // sqrt(totalStaked in VET) * 1e18, we are calculating sqrt on VET and then converting to wei
    sqrtStake := new(big.Int).Sqrt(new(big.Int).SetUint64(totalStaked))
    sqrtStake.Mul(sqrtStake, bigE18)

@>    curveFactor, err := e.params.Get(thor.KeyCurveFactor)
    if err != nil {
        return nil, err
    }
    if curveFactor.Uint64() == 0 {
        curveFactor = thor.InitialCurveFactor
    }

    // reward = 1 * curveFactor * sqrt(totalStaked / 1e18) / blocksPerYear
    reward := big.NewInt(1)
    reward.Mul(reward, curveFactor)
    reward.Mul(reward, sqrtStake)
    reward.Div(reward, thor.NumberOfBlocksPerYear)
    return reward, nil
}
```

## Impact Details

Gas for native calls is cheaper than it should be; the native call is exposed via the generated `staker.sol` contract's `issuance()` function:

```solidity
function issuance() external view returns (uint256 issued) {
    return StakerNative(address(this)).native_issuance();
}
```

This means external callers (or contracts) invoking `issuance()` may pay less gas than the protocol's intended accounting for storage reads, altering transaction-fee behavior relative to design.

## References

* <https://github.com/vechain/thor/blob/706bee9e6693244a6ddac17f883c7b09c6c63852/builtin/staker\\_native.go#L445>
* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/energy/energy.go#L341>
* <https://github.com/vechain/thor/blob/release/hayabusa/builtin/gen/staker.sol#L283>

## Proof of Concept

<details>

<summary>PoC: modify test to demonstrate missing gas charge</summary>

Change the following line in `builtin/staker_native_gas_test.go`, then run the test:

Command: go test -timeout 30s -run ^TestStakerNativeGasCosts$ github.com/vechain/thor/v2/builtin

Patch:

```diff
--- a/builtin/staker_native_gas_test.go
+++ b/builtin/staker_native_gas_test.go
@@ -236,7 +236,7 @@ func TestStakerNativeGasCosts(t *testing.T) {
                },
                {
                        function:    "native_issuance",
-                       expectedGas: 200,
+                       expectedGas: 400,
                        description: "Get issuance for the current block",
                },
        }
```

The test will fail because the code did not charge for the additional storage read (`e.params.Get(thor.KeyCurveFactor)`), demonstrating the undercharge.

</details>
