# #45514 \[SC-Medium] malicious agents can trap stakers by raising the exit collateral ratio

## #45514 \[SC-Medium] Malicious agents can trap stakers by raising the exit collateral ratio

**Submitted on May 15th 2025 at 23:58:30 UTC by @escrow for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45514
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Agents.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### Brief/Intro

The Flare team implements the function `Agents.sol::setPoolExitCollateralRatioBIPS()` in order to change the exit collateral ratio of a pool.

Unfortunately, the current logic is flawed and allows a malicious agent to raise the pool exit collateral ratio higher to than what is allowed, and what is meant to be within range. This is due to a faulty require statement implementation. As a result, they have the ability to trap stakers or constantly rise the BIPS above normal limits compared to other users.

### Vulnerability Details

The logic is safe and sound when we are firstly creating an agent vault and initialising it via the same function.

On an already initialised agent vault we execute the following flow:

`AgentSettingsFacet.sol::announceAgentSettingUpdate()` -> `AgentSettingsUpdater.sol::announceUpdate()` -> `AgentSettingsUpdater.sol::executeUpdate()` -> `AgentSettingsUpdater.sol::_executeUpdate()` -> `Agents.sol::setPoolExitCollateralRatioBPS()`

The updates are done by using a hash and then based on the hash value we execute the correct operation.

In the event that we are tweaking the `exitCollateralRatioBIPS` variable, we will execute the function `setPoolExitCollateralRatioBPS()`.

Inside of this function, we have the main check which is responsible for ensuring that we are not too low and not too high.

As such, Flare invoke the following require statement:

`require(_poolExitCollateralRatioBIPS <= currentExitCR * 3 / 2 || _poolExitCollateralRatioBIPS <= minCR * 12 / 10, "increase too big");`

This require statement is problematic because we are using the `||` operators here, and thus the `require` statement works in a manner of an `OR` operation.

These two statements are critical as they serve two different purposes, but they are used together which is where the problem arises. The current statement means that only one has to be true, in order for the entire `require` statement to rule out true.

A malicious agent can take advantage of this because the `currentExitCR` is utilised and compared to the variable we are changing it to. We have room for leverage here in respect of the `currentExitCR` versus `currentExitCR * 3 / 2`. A delta example is given in the PoC based on the scenario.

Ultimately, we can keep forcing changes in respect of the delta zone here which means we never have to depend on succeeding the second statement of the overall require statement. So, we bypass the `maxCR` (so to speak) which is calculated by `minCR * 12 / 10`.

As a result, agents have unwarranted control over the natural mechanism of a collateral pool and have significant leverage over stakers where they can disallow them from exiting.

Such scenario is relevant to the functionality of `CollateralPool.sol::exitTo` or `exit` where within `_exitTo` we meet the following statement:

`require(_staysAboveCR(assetData, natShare, assetData.exitCR), "...");` where `assetData.exitCR` is retrieved via `_getAssetData()` (which retrieves the exit collateral ratio via `_safeExitCollateralRatioBIPS()`). The staker has to meet the conditions of the system in respect of the `exitCR` value.

## Impact Details

Malicious agents would be able to trap stakers within certain collateral pools unfairly and the stakers would not be able to correctly exit due to the inflated requirements.

Additionally, the malicious agents have the ability to simply bypass the margins of what BIPS they can work with, having an advantage over other agents with other collateral pools who abide by the rules.

If stakers are being griefed/trapped, then there is no way to guarantee that they will be aware of the grief/trap. Also, you can still force stakers to deal with higher amounts than usual just from 1 change due to the bypass of the second statement. Then, we have another scenario of an agent for instance purposefully bypassing the thresholds they should be abiding to even in normal situations with no malicious intent.

### References

Already made within the vulnerability details.

### Proof of Concept

### Proof of Concept

I am aware only a step by step PoC is required, but I thought I'd attach the following anyway as you can see the logic in Chisel (Foundry).

```
➜ uint minCR = 4000;
➜ uint currentExitCR = 5500
➜ uint $ = 7000; /* _poolExitCollateralRatioBIPS */
➜ uint maxCR = minCR * 12 / 10;
➜ bool($ <= currentExitCR * 3 / 2)
Type: bool
└ Value: true
➜ bool ($ <= minCR * 12 / 10)
Type: bool
└ Value: false
➜ bool($ <= currentExitCR * 3 / 2 || $ <= minCR * 12 / 10)
Type: bool
└ Value: true
➜ uint delta = (currentExitCR * 3 / 2) - currentExitCR
➜ delta
Type: uint256
├ Hex: 0xabe
├ Hex (full word): 0x0000000000000000000000000000000000000000000000000000000000000abe
└ Decimal: 2750
```

Due to the above logic, all we have to do is just stay within the bounds of `$ <-> currentExitCR * 3 / 2` and we can continue raising the CR required to leave, and as a result make it impossible to leave.

You cannot do a significant BIPS change straight away, but you can gradually climb there based on the delta of the statement of what you are allowed to change to. This increases every time due to how we increase upon every change.

Excluding the griefing stakers context, we still have a problem here regarding bypassing the `minCR * 12 / 10` threshold which should not be allowed.
