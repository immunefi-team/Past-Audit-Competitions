# 59296 sc low periodattimestamp uint48 timestamp ignores its parameter and always returns the current period

**Submitted on Nov 10th 2025 at 20:19:08 UTC by @edantes for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59296
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Theft of gas

## Description

## Brief/Intro

The read-only helper periodAtTimestamp(uint48 timestamp) is supposed to return the period number for an arbitrary timestamp. Instead, it uses “now” via \_sinceEpoch(Time.timestamp()), so it returns the current period regardless of the argument. Integrators relying on it to schedule or visualize claim windows will compute the wrong period for historical timestamps.

## Vulnerability Details

**Component:** FirelightVault.sol

**Functions:**

* periodAtTimestamp(uint48 timestamp)
* \_sinceEpoch(uint48 epoch)

**Root cause**

In FirelightVault.sol, periodAtTimestamp(uint48 timestamp) delegates to \_sinceEpoch(periodConfiguration.epoch) when forming the period number, but \_sinceEpoch is defined as Time.timestamp() - epoch. That means the function completely discards the timestamp parameter and computes the period for the present (“now”), not for the caller-provided time.

* periodAtTimestamp(uint48 timestamp) returns startingPeriod + \_sinceEpoch(uint48 epoch) / duration. It uses “now” instead of (timestamp - epoch) / duration.
* \_sinceEpoch(uint48 epoch) returns Time.timestamp() - epoch.

dApps that query a past/future period get the current period number instead. When users act on that misinformation and call claimWithdraw(uint256 period) too early, the contract reverts (if (period >= currentPeriod()) revert InvalidPeriod();). Funds remain safe, but UX is broken and automations fail.

## Impact Details

* **Incorrect Scheduling:** Any integrator using periodAtTimestamp() to map timestamps and period numbers will display/schedule the wrong claim period for historical timestamps. Users attempting to claim based on those schedules will encounter InvalidPeriod / NoWithdrawalAmount until the real period arrives.
* **Automation Churn:** Off-chain bots may repeatedly trigger failed transactions (gas waste & alert noise).

**Risk Breakdown**

**Low:** Incorrect read causing disrupted UX without financial impact. This is a low-impact, high-likelihood issue.

## Recommendation

Use the provided timestamp when computing the period offset from the configuration epoch:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // use the caller-supplied timestamp, not "now"
    return periodConfiguration.startingPeriod + (uint256(timestamp) - uint256(periodConfiguration.epoch)) / periodConfiguration.duration;
}
```

No changes to currentPeriod\*() functions are needed since they correctly use Time.timestamp() for “now.”

## Proof of Concept

## Proof of Concept

A deterministic Hardhat test shows periodAtTimestamp(tsPast) returns the current period rather than the past period because the implementation ignores its timestamp parameter.

**Test Details:**

* Setup: Deploy the vault (initialize with a valid period configuration).
* Choose a historical timestamp: historicalTs = epoch + duration + 10.
* Advance time: Move forward several periods so currentPeriod() > expectedPast.
* Observe:
  * periodAtTimestamp(historicalTs) equals currentPeriod() (bug).
  * The correct past period is starting + (historicalTs - epoch) / duration.

**How to run:**

```bash

# from repo
npm install
npx hardhat test test/PoC_FirelightVault.periodAtTimestamp.spec.js
```

**Hardhat Test File:** test/PoC\_FirelightVault.periodAtTimestamp.spec.js

```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

// Low-level helpers to control time without relying on toolbox helpers
async function setNextTimestamp(ts) {
  await ethers.provider.send("evm_setNextBlockTimestamp", [Number(ts)]);
  await ethers.provider.send("evm_mine", []);
}
async function increaseTime(seconds) {
  await ethers.provider.send("evm_increaseTime", [Number(seconds)]);
  await ethers.provider.send("evm_mine", []);
}

describe("FirelightVault.periodAtTimestamp bug", function () {
  it("returns the current period even when asked about a past timestamp", async function () {
    const [deployer] = await ethers.getSigners();

    // Pick a valid duration and align the epoch
    const DURATION = 86400; // 1 day
    const T0_RAW = 1_900_000_000;
    const T0 = Math.floor(T0_RAW / DURATION) * DURATION; // align epoch to duration boundary

    // Fix the chain time before deployment/initialization so epoch is deterministic
    await setNextTimestamp(T0);

    // Deploy the vault
    const Vault = await ethers.getContractFactory("FirelightVault");
    const vault = await Vault.deploy();
    await vault.waitForDeployment();

    // Init params struct:
    const abi = ethers.AbiCoder.defaultAbiCoder();
    const depositLimit = 10n ** 24n; // any nonzero cap

    const initParams = abi.encode(
      ["address","address","address","address","address","uint256","uint48"],
      [deployer.address, deployer.address, deployer.address, deployer.address, deployer.address, depositLimit, DURATION]
    );

    // Info: A dummy non-zero address is used as the asset (not used in this test)
    await vault.initialize(
      deployer.address,         // dummy IERC20 asset address (non-zero)
      "Firelight Vault",
      "FLV",
      initParams
    );

    // Grab the initial period configuration so we know epoch/duration/startingPeriod
    const pc0 = await vault.currentPeriodConfiguration();
    const epoch = Number(pc0.epoch);
    const duration = Number(pc0.duration);
    const starting = Number(pc0.startingPeriod); // expected 0 on first config

    // Choose a historical timestamp well in the past (1 full period after epoch)
    const historicalTs = epoch + duration + 10;

    // Advance chain time to several periods in the future so currentPeriod > pastPeriod
    await increaseTime(5 * duration + 7);

    const reportedForPast = await vault.periodAtTimestamp(historicalTs);
    const current = await vault.currentPeriod();

    // What the correct (expected) past period *should* be:
    const expectedPast =
      starting + Math.floor((historicalTs - epoch) / duration);

    // BUG: periodAtTimestamp(timestamp) equals currentPeriod()
    // because _sinceEpoch() uses Time.timestamp() (now), ignoring the parameter.
    expect(reportedForPast).to.equal(current);

    // And it should *not* equal the true past period number
    expect(Number(reportedForPast)).to.not.equal(expectedPast);

    // For extra clarity, assert that current is indeed greater than the expected past period
    expect(Number(current)).to.be.greaterThan(expectedPast);
  });
});
```
