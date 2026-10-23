# 57300 sc insight initialization bypasses the max 2 weeks guard for min upgrade delay&#x20;

**Submitted on Oct 25th 2025 at 04:39:57 UTC by @Afriauditor for Audit Comp | Folks Finance: Wormhole NTT on Algorand**

* **Report ID:** #57300
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/ntt\\_manager/NttManager.py>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

`NttManager.py` inherits the `Upgradeable` contract. In `Upgradeable`, admins are constrained to set a maximum minimum-upgrade delay of two weeks enforced in the setter via:

assert min\_upgrade\_delay <= self.max\_for\_min\_upgrade\_delay(), "Delay exceeds maximum allowed"

The code comments explicitly state this cap “prevents setting `get_active_min_upgrade_delay()` so large that upgrades become effectively impossible.” However, this constraint is **not enforced at initialization**: `NttManager.create(...)` forwards the provided `min_upgrade_delay` to `Upgradeable.create(...)`, which **does not validate** the value. As a result, an arbitrarily large delay can be locked in at genesis.

### Vulnerability Details

* `NttManager.create(..., min_upgrade_delay)` → calls `Upgradeable.create(self, min_upgrade_delay)`.
* `Upgradeable.create(min_upgrade_delay)` stores `delay_1 = min_upgrade_delay` **without** the two-week cap check.
* All future scheduling operations must satisfy:

timestamp >= Global.latest\_timestamp + get\_active\_min\_upgrade\_delay()

* If the initial `min_upgrade_delay` is set to an extremely large value at creation, the system effectively **prevents** timely scheduling of:
  * contract upgrades, and
  * future reductions to the delay itself,

contradicting the safety intent documented in the code.

## Impact Details

A large `min_upgrade_delay` at deployment effectively locks the contract’s upgrade path.

## References

* Target source: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/ntt\\_manager/NttManager.py>

## Proof of Concept

Add the following test to `NttManger.test.ts` to demonstrate that create accepts a delay greater than two weeks and stores it unbounded:

{% code title="NttManger.test.ts" %}

```typescript
test("create uses provided min_upgrade_delay even if > 2 weeks", async () => {
  const HUGE_DELAY = 20n * SECONDS_IN_DAY; // set to 20 days which is over over 2 weeks
  const { appClient } = await factory.deploy({
    createParams: {
      sender: creator,
      method: "create",
      args: [nttTokenAppId, SOURCE_CHAIN_ID, THRESHOLD, HUGE_DELAY],
      extraFee: (1000).microAlgos(),
    },
  });

  // read this instance's state
  expect(await appClient.state.global.minUpgradeDelay()).toEqual({
    delay_0: 0n,
    delay_1: HUGE_DELAY,   // proves no clamp at create-time
    timestamp: 0n,
  });
  expect(await appClient.getActiveMinUpgradeDelay()).toEqual(HUGE_DELAY);
});
```

{% endcode %}

## Notes

No fixes or mitigation steps are included in this report.
