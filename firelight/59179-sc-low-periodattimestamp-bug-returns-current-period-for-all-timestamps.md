# 59179 sc low periodattimestamp bug returns current period for all timestamps

**Submitted on Nov 9th 2025 at 16:14:05 UTC by @coinsspor for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59179
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

I found a bug in periodAtTimestamp() - it ignores the timestamp parameter and always returns the current period. This breaks external contracts and services trying to query historical or future periods.

## Vulnerability Details

**Location:** FirelightVault.sol, lines 246-250 and 795-797

**What I Found:** periodAtTimestamp() should calculate which period a given timestamp falls into. But I noticed there's a bug in the \_sinceEpoch() helper - it always uses Time.timestamp() (current block time) instead of the timestamp parameter.

Here's the buggy code:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory pc = periodConfigurationAtTimestamp(timestamp);
    return pc.startingPeriod + _sinceEpoch(pc.epoch) / pc.duration;
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;  // Bug: always uses current time
}
```

The timestamp parameter never gets used. I tested this and every query returns the current period regardless of what timestamp I pass in.

**What I Tested:** I deployed the contract and moved time forward 10 periods (current period = 10):

* When I queried timestamp for period 2 → Got 10 back (wrong)
* When I queried timestamp for period 15 → Got 10 back (wrong)
* When I queried timestamps for periods 1-5 → All returned 10 (all wrong)

The only time it works is when I query current block.timestamp (and that's just by accident).

## Impact Details

**External Contracts:** Any contract checking elapsed time will get wrong period data. I can see this causing problems with withdrawal eligibility checks - periodAtTimestamp(requestTime) always returns current period so the checks become useless.

**Off-chain Systems:** From what I can tell:

* Indexers will show flat historical data
* Analytics dashboards won't display period metrics correctly
* Monitoring alerts won't work properly

**Integration Partners:** Projects building on Firelight need accurate period data. This bug makes the public API unreliable.

**Why I'm Categorizing as Griefing:** There's no direct fund loss, but I think this causes systematic data corruption that damages protocol usability. No profit motive for attacker, but clear damage to users and integrations.

## References

**My Suggested Fix:** Line 249 should be:

```solidity
return periodConfiguration.startingPeriod + 
       (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
```

## Proof of Concept

## Proof of Concept / Steps to Reproduce

**Test Environment:**

* OS: Ubuntu 22.04
* Node.js: v20.x
* Hardhat: 2.22.x
* Solidity: 0.8.28

**Setup Steps:**

1. I cloned the Firelight repository:

```bash
git clone https://github.com/firelight-protocol/firelight-core.git
cd firelight-core
npm install
```

2. I created a simple MockERC20 contract for testing:

```solidity
// contracts/MockERC20.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}
```

3. I configured the environment:

```bash

# Created .env file with test keys
cat > .env <<EOF
DEPLOYMENT_ACCOUNT_KEY=ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
HARDHAT_CHAIN_ID=31337
EOF

# Disabled forking in hardhat.config.js (line 29)
sed -i '29s/forking,/\/\/ forking,/' hardhat.config.js
```

4. I wrote this test to reproduce the bug:

```javascript
// test/PeriodAtTimestamp_PROOF.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("FIRELIGHT VAULT BUG - periodAtTimestamp() PROOF", function () {
  let vault, asset, deployer, attacker;
  const PERIOD_DURATION = 86400;
  const DEPOSIT_LIMIT = ethers.parseEther("100000");

  before(async function () {
    console.log("\n" + "=".repeat(80));
    console.log("FIRELIGHT VAULT - periodAtTimestamp() BUG PROOF OF CONCEPT");
    console.log("=".repeat(80));
    
    [deployer, attacker] = await ethers.getSigners();

    const MockERC20 = await ethers.getContractFactory("contracts/MockERC20.sol:MockERC20");
    asset = await MockERC20.deploy("Mock Token", "MOCK");
    await asset.waitForDeployment();

    const Vault = await ethers.getContractFactory("FirelightVault");
    vault = await Vault.deploy();
    await vault.waitForDeployment();

    const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
      ["tuple(address,address,address,address,address,uint256,uint48)"],
      [[deployer.address, deployer.address, deployer.address,
        deployer.address, deployer.address, DEPOSIT_LIMIT, PERIOD_DURATION]]
    );

    await vault.initialize(await asset.getAddress(), "Firelight Vault", "fVault", initParams);
    console.log("Vault initialized\n");
  });

  it("PROOF: Historical query returns WRONG period", async function () {
    console.log("=".repeat(80));
    console.log("TEST: HISTORICAL QUERY");
    console.log("=".repeat(80));

    const config = await vault.periodConfigurations(0);
    const startEpoch = config.epoch;
    const duration = config.duration;
    
    console.log(`\nStart Epoch: ${startEpoch}, Duration: ${duration}s`);

    await time.increase(Number(duration) * 10);
    const currentPeriod = await vault.currentPeriod();
    console.log(`Current Period: ${currentPeriod}\n`);

    const queryTimestamp = Number(startEpoch) + (Number(duration) * 2);
    const expected = 2;
    const actual = await vault.periodAtTimestamp(queryTimestamp);
    
    console.log("Query for Period 2:");
    console.log(`   Expected: ${expected}`);
    console.log(`   Got: ${actual}`);
    console.log(`   ERROR: ${Number(actual) - expected} periods off!\n`);

    expect(actual).to.equal(currentPeriod);
    expect(Number(actual)).to.not.equal(expected);
  });

  it("PROOF: Multiple queries ALL WRONG", async function () {
    console.log("=".repeat(80));
    console.log("TEST: MULTIPLE HISTORICAL QUERIES");
    console.log("=".repeat(80));

    const config = await vault.periodConfigurations(0);
    const startEpoch = config.epoch;
    const duration = config.duration;
    const currentPeriod = await vault.currentPeriod();

    console.log("\nTesting Periods 1-5:");
    console.log("Expected | Got | Status");
    console.log("-".repeat(40));

    for (let p = 1; p <= 5; p++) {
      const ts = Number(startEpoch) + (Number(duration) * p);
      const result = await vault.periodAtTimestamp(ts);
      const status = result == currentPeriod ? "FAIL" : "OK";
      console.log(`   ${p}     |  ${result}  | ${status}`);
      expect(result).to.equal(currentPeriod);
    }
    console.log("\n");
  });
});
```

**Running the Test:**

```bash
npx hardhat test test/PeriodAtTimestamp_PROOF.test.js
```

**Results:**

```
FIRELIGHT VAULT - periodAtTimestamp() BUG PROOF OF CONCEPT
================================================================================
Vault initialized

TEST: HISTORICAL QUERY
================================================================================
Start Epoch: 1762614530, Duration: 86400s
Current Period: 10

Query for Period 2:
   Expected: 2
   Got: 10
   ERROR: 8 periods off!

PROOF: Historical query returns WRONG period (passed)

TEST: MULTIPLE HISTORICAL QUERIES
================================================================================
Testing Periods 1-5:
Expected | Got | Status
----------------------------------------
   1     |  10  | FAIL
   2     |  10  | FAIL
   3     |  10  | FAIL
   4     |  10  | FAIL
   5     |  10  | FAIL

PROOF: Multiple queries ALL WRONG (passed)

2 passing (144ms)
```

**What This Proves:**

I tested the periodAtTimestamp() function with different timestamps:

* Queried for period 2 (2 days after deployment) → Got period 10 (current)
* Queried for periods 1-5 → All returned period 10 (current)

The function completely ignores the timestamp parameter. This happens because \_sinceEpoch() uses Time.timestamp() instead of the parameter.

**Verification:** Anyone can reproduce this by following the setup steps above. The bug is deterministic.
