# 55957 sc medium checkstake does not check for uint64 overflow

**Submitted on Oct 8th 2025 at 10:15:13 UTC by @Haxatron for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #55957
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/vechain/thor/tree/c090c1abb1387d057bc25f26ac83f96c49f4ef24/builtin/gen>
* **Impacts:** Security hardening and incorrect event emission

## Description

### Brief/Intro

`checkStake` does not check for uint64 overflow. This is especially relevant for functions such as `decreaseStake` which allow specifying any amount and don't require sending VET to the contract.

### Vulnerability Details

On the smart contract layer, the `decreaseStake` function allows a user to specify any `amount` without needing to send VET to the contract:

```solidity
    function decreaseStake(
        address validator,
        uint256 amount
    ) public checkStake(amount) stakerNotPaused {
        StakerNative(address(this)).native_decreaseStake(validator, msg.sender, amount);
        emit StakeDecreased(validator, amount);
    }
```

A user could provide an `amount` equal to `type(uint64).max * 1e18 + 1e18`, for example. Later, in the native `thor` client code it calls `ToVET` on the amount:

```go
{"native_decreaseStake", func(env *xenv.Environment) ([]any, error) {
    var args struct {
        Validator common.Address
        Endorser  common.Address
        Amount    *big.Int
    }
    env.ParseArgs(&args)
    charger := gascharger.New(env)

    err := Staker.NativeMetered(env.State(), charger).
        DecreaseStake(
            thor.Address(args.Validator),
            thor.Address(args.Endorser),
            staker.ToVET(args.Amount), // convert from wei to VET,
        )
    if err != nil {
        return nil, err
    }
    return []any{}, nil
}},
```

`ToVET` converts the `amount` to `uint64`. For `type(uint64).max * 1e18 + 1e18`, the conversion will wrap/truncate and result in `1` when `ToVET` is called. The function proceeds normally, as if the user specified an `amount` of 1 VET (1e18 wei) to `decreaseStake`:

```go
func ToVET(wei *big.Int) uint64 {
    return new(big.Int).Div(wei, bigE18).Uint64()
}
```

This can lead to incorrect event emission for the `StakeDecreased` event since the Solidity contract emits the original (large) `amount`:

```solidity
        emit StakeDecreased(validator, amount);
```

For most functions the total VET supply prevents reaching uint64 max, so sending msg.value that exceeds uint64 VET is infeasible. However, since `decreaseStake` can be called without sending VET to the contract, it is possible to pass an arbitrary large `amount`. It's recommended to add a check in `checkStake` ensuring that `amount / 1e18 <= type(uint64).max`.

Suggested modifier change:

```solidity
    modifier checkStake(uint256 amount) {
        require(amount > 0, "staker: stake is empty");
        require(amount % 1e18 == 0, "staker: stake is not multiple of 1VET");
+      require(amount / 1e18 <= uint256(type(uint64).max), "staker: stake VET cannot exceed uint64.max");
        _;
    }
```

## Impact Details

* Security hardening: prevent unexpected uint64 overflow/truncation in native code.
* Incorrect event emission: the Solidity event may show a vastly different wei amount than what native code uses after conversion to uint64 VET.

## Proof of Concept

POC not required for Insight in this Attackathon.
