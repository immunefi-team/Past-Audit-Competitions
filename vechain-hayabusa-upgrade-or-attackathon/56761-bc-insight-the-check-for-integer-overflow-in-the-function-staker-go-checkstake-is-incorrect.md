# 56761 bc insight the check for integer overflow in the function staker go checkstake is incorrect

**Submitted on Oct 20th 2025 at 13:10:36 UTC by @LeoFlint for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56761
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>

## Description

The function `builtin/staker/staker.go#checkStake()` adds three uint64 values and attempts to check for integer overflow. The current implementation misses verifying the carry after the first addition, which can let an overflow go undetected and cause the function to return an incorrect result.

## Details

Current code:

```go
func checkStake(valNextPeriodTVL, aggNextPeriodTVL, amount uint64) (uint64, error) {
	total1, carry := bits.Add64(valNextPeriodTVL, aggNextPeriodTVL, 0)
	//@audit no check carry
	total2, carry := bits.Add64(total1, amount, carry)
	if carry != 0 {
		return 0, NewReverts("stake is out of range")
	}
	return total2, nil
}
```

Because the code does not check `carry` immediately after the first addition, an overflow occurring there can be missed and the function can proceed incorrectly.

## Impact

All values passed to `checkStake()` were bounded before and overflow should never happen. This makes the issue informational in the current code paths, but the check itself is incorrect and should be fixed to be robust.

## Recommendation

Check the carry after the first addition as well. For example:

```go
func checkStake(valNextPeriodTVL, aggNextPeriodTVL, amount uint64) (uint64, error) {
	total1, carry := bits.Add64(valNextPeriodTVL, aggNextPeriodTVL, 0)
	if carry != 0 {
		return 0, NewReverts("stake is out of range")
	}

	total2, carry := bits.Add64(total1, amount, carry)
	if carry != 0 {
		return 0, NewReverts("stake is out of range")
	}
	return total2, nil
}
```

## References

<https://github.com/vechain/thor/blob/55dfff0c869801d9c02f50780b15f9b153953970/builtin/staker/staker.go#L739>

## Proof of Concept

<details>

<summary>Test demonstrating missed overflow</summary>

Test to add to `builtin/staker/staker_test.go`:

```go
func Test_CheckStakeCarry(t *testing.T) {
	var valNextPeriodTVL uint64 = 10
	var aggNextPeriodTVL uint64 = math.MaxUint64
	var amount uint64 = 5
	_, err := checkStake(valNextPeriodTVL, aggNextPeriodTVL, amount)
	assert.NoError(t, err)
}
```

With the original (incorrect) implementation the test passes even though an integer overflow occurs:

```bash
PASS
ok      github.com/vechain/thor/v2/builtin/staker       0.024s
```

</details>
