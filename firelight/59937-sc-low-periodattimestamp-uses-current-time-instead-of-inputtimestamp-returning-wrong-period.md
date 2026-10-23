# #59937 \[SC-Low] periodattimestamp uses current time instead of inputtimestamp returning wrong period

**Submitted on Nov 17th 2025 at 03:42:51 UTC by @IronsideSec for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59937
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The vault’s `periodAtTimestamp(timestamp)` ignores its input `timestamp` argument and computes using the current block time. This yields incorrect period numbers for historicalqueries, can mis-schedule accounting tied to periods, and enables unfairness between users whose actions are recorded in different periods than intended.

### Vulnerability Details

* **Concrete math with the PoC’s numbers**:
  * **Initial config**: epoch = 100, duration = 10
  * **Later config**: at epoch = 160, duration becomes 20 (unrelated to the bug but present in the PoC)
  * **Now = current time**: 150
  * **Query**: `periodAtTimestamp( t = 115)`
  * **Returned (buggy)**: starting + (now − epoch)/duration = 0 + (150 − 100)/10 = 5 period
  * **Correct**: starting + (t − epoch)/duration = 0 + (115 − 100)/10 = 1 period
* **Buggy code (uses now timestamp, not input):**

```194:210:contracts/firelightvault.sol
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // solhint-disable-next-line max-line-length
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

/**
 * ...
 */
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

* **PoC in tests (exact lines):**

```87:134:test/poc.t.sol
function test_PeriodAtTimestamp_DependsOnNow_NotInput() public {
    // Force epoch = 100 (days) by warping before initialize of a fresh vault
    vm.warp(100 days);

    FirelightVault v = new FirelightVault();
    FirelightVault.InitParams memory params = FirelightVault.InitParams({
        defaultAdmin: address(this),
        limitUpdater: address(this),
        blocklister: address(0),
        pauser: address(0),
        periodConfigurationUpdater: address(this),
        depositLimit: LIMIT,
        periodConfigurationDuration: 10 days // duration = 10
    });

    // Initialize fresh vault with 10-day periods starting at epoch=100 days
    bytes memory initCalldata = abi.encodeWithSelector(
        v.initialize.selector,
        address(token),
        "Vault",
        "VLT",
        abi.encode(params)
    );
    (bool ok, ) = address(v).call(initCalldata);
    require(ok, "init failed");

    // "after a while it is 20": change duration to 20 starting at epoch = 160 days
    v.addPeriodConfiguration(uint48(160 days), uint48(20 days));

    // Now = 150
    vm.warp(150 days);

    // Query period at t = 115
    uint48 t = uint48(115 days);
    uint256 actual = v.periodAtTimestamp(t);

    // Buggy function uses now: starting(0) + (150-100)/10 = 5
    assertEq(actual, 5, "bug: returns period 5 at now=150");

    // Correct: starting(0) + (115-100)/10 = 1
    uint256 expected = (uint256(115 days) - uint256(100 days)) / uint256(10 days);
    assertEq(expected, 1, "sanity: expected period 1");

    // Prove mismatch
    assertTrue(actual != expected, "function should have used input timestamp");
}
```

### Impact Details

* **Incorrect period classification** for historical timestamps used by other logic (e.g., scheduling, accounting, limits) can:
  * Allocate or settle amounts in wrong periods.
  * Create unfairness between users whose actions are bucketed into unintended periods.
  * Break assumptions around period-based configuration changes.

### References

* Affected function: `periodAtTimestamp` in `contracts/FirelightVault.sol` lines 194–210 (see snippet above).
* Reproducer: `test/X.t.sol` lines 87–134 (see snippet above).

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/7a44ccd02d3a87d17a19cc9b3e1edf2f>

## Proof of Concept

## Proof of Concept

1. run forge init --force
2. Delete `test/Counter.t.sol` and `script/Counter.s.sol`
3. Create `test/POC.t.sol` and paste the code copied from <https://gist.github.com/IronsideSec/7a44ccd02d3a87d17a19cc9b3e1edf2f>
4. then run `forge t --mt test_PeriodAtTimestamp_DependsOnNow_NotInput -vvvv`
