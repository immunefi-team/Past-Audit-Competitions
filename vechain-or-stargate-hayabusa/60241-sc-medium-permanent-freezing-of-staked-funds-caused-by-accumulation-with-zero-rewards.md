# 60241 sc medium permanent freezing of staked funds caused by accumulation with zero rewards

Submitted on Nov 20th 2025 at 10:17:47 UTC by @Paludo0x for [Audit Comp | Vechain | Stargate Hayabusa](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #60241
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Impacts:
  * Permanent freezing of funds

## Description

### Brief / Intro

A delegator's NFT in the Stargate protocol can become permanently locked under these conditions:

* It accumulates more claimable periods than `maxClaimablePeriods`.
* It receives zero rewards across those periods.

Because `lastClaimedPeriod` only advances when the claimable reward is greater than zero, users with consistently zero rewards are unable to reduce the claimable period window. Once the period span exceeds the maximum allowed, all calls to `unstake()` and `delegate()` revert with `MaxClaimablePeriodsExceeded`, resulting in a permanent freeze of the user's staked VET.

### Vulnerability details (summary)

The root cause is the interaction between:

* computation of claimable period window (`_claimableDelegationPeriods`),
* the `maxClaimablePeriods` guard (`_exceedsMaxClaimablePeriods`),
* and how `lastClaimedPeriod` is updated in `_claimRewards`.

*key points:*

* `_claimableDelegationPeriods` computes a window from `lastClaimedPeriod + 1` up to either the delegation `endPeriod` or the validator's `completedPeriods`.
* As the validator completes more periods, `lastClaimablePeriod` (the window upper bound) can grow indefinitely for active delegations.
* The guard `_exceedsMaxClaimablePeriods` checks the window length and blocks operations when the window length >= `maxClaimablePeriods`.
* `_claimRewards` updates `lastClaimedPeriod` only if the computed `claimableAmount` is > 0. If `claimableAmount == 0`, the function returns early without updating `lastClaimedPeriod`.
* Integer division when computing per-period rewards can make a delegator's reward round to 0:

  ```
  rewardPerPeriod = (effectiveStake * delegationPeriodRewards) / delegatorsEffectiveStake;
  ```
* If rewards round to zero for many consecutive periods, `lastClaimedPeriod` stays unchanged, the window grows, then becomes and remains larger than `maxClaimablePeriods`. Because `unstake()` and redelegation check this guard before calling `_claimRewards`, the delegator cannot advance `lastClaimedPeriod` and becomes permanently locked.

### Example of relevant code snippets

* Window computation (excerpt):

```solidity
// next claimable period is the last claimed period + 1
uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
if (nextClaimablePeriod < startPeriod) {
    nextClaimablePeriod = startPeriod;
}
// ...
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

* Max periods guard:

```solidity
if (lastClaimablePeriod - firstClaimablePeriod >= $.maxClaimablePeriods) {
    return true;
}
```

* Claim rewards (excerpt):

```solidity
uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
if (claimableAmount == 0) {
    // BUG: lastClaimedPeriod is NOT updated when claimableAmount == 0
    return;
}
$.lastClaimedPeriod[_tokenId] = lastClaimablePeriod;
VTHO_TOKEN.safeTransfer(tokenOwner, claimableAmount);
```

### Impact

* Affected delegators:
  * Cannot claim non-zero rewards (there are none).
  * Cannot shrink the claimable period window.
  * Cannot unstake or redelegate because those operations revert before allowing `_claimRewards` to advance the cursor.
* This results in permanently frozen VET for affected NFTs.

Severity stated: Critical in text (but reported severity: Medium).

## Proof of Concept (PoC)

<details>

<summary>PoC scenario and logs (expand)</summary>

The PoC demonstrates the vulnerability with this scenario:

* Total stake: 498,750 VET (95 whales × 1,500 VET each)
* Dust user stake: 150 VET (0.03% share)
* Rewards per period: 1000 wei (from mock)
* Dust user reward calculation: (150 × 1000) / 498,750 = 0 (integer division)

PoC log excerpt:

```
  Security Reproduction: Dust Delegator Deadlock

 Configured MaxClaimablePeriods to 5

 Added 95 mega whale delegations at level 3 to maximize dilution

 Setting up dust delegator...

Staked NFT of level 1

Fast-forwarded 10 blocks to mature the NFT

Delegated NFT to validator 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
 Dust delegation effective stake: 150000000000000000000
 Delegation is ACTIVE

 Requesting delegation exit for dust user...

 Fast forwarding 7 periods to exceed the limit...
 After FF period 1: claimableFirst=2, claimableLast=2, totalStake=498900000000000000000000
 After FF period 2: claimableFirst=2, claimableLast=3, totalStake=498750000000000000000000
 After FF period 3: claimableFirst=2, claimableLast=4, totalStake=498750000000000000000000
 After FF period 4: claimableFirst=2, claimableLast=5, totalStake=498750000000000000000000
 After FF period 5: claimableFirst=2, claimableLast=6, totalStake=498750000000000000000000
 After FF period 6: claimableFirst=2, claimableLast=7, totalStake=498750000000000000000000
 After FF period 7: claimableFirst=2, claimableLast=8, totalStake=498750000000000000000000

 Attempting unstake (should fail)...
 Unstake reverted as expected (MaxClaimablePeriodsExceeded)

 Attempting to claim rewards...
 Pending Rewards calculated by contract: 0 wei
 Claimable delegation periods before claim: first=2 last=8
 Validator total effective stake at last claimable period: 498750000000000000000000
 Dust user effective stake: 150000000000000000000
 Dust user share (approx) = 300 ppm
 Rewards are exactly 0 wei due to integer division
 This reproduces the exact bug condition.
 Claim transaction executed successfully

 Attempting unstake again after claiming...
DEADLOCK CONFIRMED: claimRewards(0) completed but did not update lastClaimedPeriod
User is permanently locked and cannot unstake
    ✔ Should permanently block a dust delegator when rewards round to zero and max periods are exceeded (11640ms)
```

</details>

### PoC code

The full PoC test provided in the report (TypeScript / Hardhat test) is included below:

```ts
// (PoC code block starts)
import { expect } from "chai";
import { ethers } from "hardhat";
import {
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
    MyERC20__factory,
    StargateNFT,
    Stargate,
} from "../../typechain-types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { setCode } from "@nomicfoundation/hardhat-network-helpers";
import {
    getOrDeployContracts,
    log,
    mineBlocks,
    fastForwardValidatorPeriods,
    stakeAndDelegateNFT,
} from "../helpers";

describe("Security Reproduction: Dust Delegator Deadlock", () => {
    let protocolStakerMock: ProtocolStakerMock;
    let stargateNFTContract: StargateNFT;
    let stargateContract: Stargate;

    let deployer: HardhatEthersSigner; // Admin & Whale Validator
    let dustUser: HardhatEthersSigner; // Victim
    let whaleAccounts: HardhatEthersSigner[];

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        // Deploy protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy mock VTHO token at the hardcoded VeChain VTHO address
        const VTHO_ADDRESS = "0x0000000000000000000000000000456E65726779";
        const vthoMockFactory = new MyERC20__factory(deployer);
        const vthoMockDeployed = await vthoMockFactory.deploy(deployer.address, deployer.address);
        await vthoMockDeployed.waitForDeployment();

        // Copy the deployed bytecode to the hardcoded VTHO address
        const vthoCode = await ethers.provider.getCode(await vthoMockDeployed.getAddress());
        await setCode(VTHO_ADDRESS, vthoCode);

        // Get VTHO contract at the hardcoded address
        const vthoContract = MyERC20__factory.connect(VTHO_ADDRESS, deployer);

        // Deploy contracts
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });

        // Mint VTHO to the Stargate contract
        await vthoContract.mint(
            await contracts.stargateContract.getAddress(),
            ethers.parseEther("1")
        );

        stargateNFTContract = contracts.stargateNFTContract;
        stargateContract = contracts.stargateContract;
        dustUser = contracts.otherAccounts[95]; // Use account after the 95 mega whales
        whaleAccounts = contracts.otherAccounts.slice(0, 95); // First 95 accounts for whales

        // Add validator with 120 blocks period
        const tx = await protocolStakerMock.addValidation(deployer.address, 120);
        await tx.wait();

        // Set validator to active
        await protocolStakerMock.helper__setValidatorStatus(deployer.address, 2); // ACTIVE = 2
    });

    it("Should permanently block a dust delegator when rewards round to zero and max periods are exceeded", async () => {
        // 1. CONFIGURATION
        const TEST_MAX_PERIODS = 5;
        await stargateContract.connect(deployer).setMaxClaimablePeriods(TEST_MAX_PERIODS);
        log(`\n Configured MaxClaimablePeriods to ${TEST_MAX_PERIODS}`);

        // 2. SETUP MEGA WHALE TO EXTREME DILUTION
        const megaWhales = [deployer, ...whaleAccounts.slice(0, 94)]; // 95 mega whales total
        const megaWhaleLevelId = 3; // Mjolnir level (highest stake)

        for (const whale of megaWhales) {
            await stakeAndDelegateNFT(
                whale,
                megaWhaleLevelId,
                deployer.address,
                stargateNFTContract,
                stargateContract,
                false
            );
            const [whalePeriodDuration, whaleStartBlock] =
                await protocolStakerMock.getValidationPeriodDetails(deployer.address);
            await fastForwardValidatorPeriods(
                Number(whalePeriodDuration),
                Number(whaleStartBlock),
                0
            );
        }
        log(
            `\n Added ${megaWhales.length} mega whale delegations at level ${megaWhaleLevelId} to maximize dilution`
        );

        // 3. SETUP DUST USER
        const levelId = 1;
        log("\n Setting up dust delegator...");
        const { tokenId } = await stakeAndDelegateNFT(
            dustUser,
            levelId,
            deployer.address,
            stargateNFTContract,
            stargateContract
        );

        const [periodDurationInitial, startBlockInitial] =
            await protocolStakerMock.getValidationPeriodDetails(deployer.address);
        await fastForwardValidatorPeriods(
            Number(periodDurationInitial),
            Number(startBlockInitial),
            0
        );
        await protocolStakerMock.helper__setValidationCompletedPeriods(deployer.address, 1);

        log(
            ` Dust delegation effective stake: ${
                await stargateContract.getEffectiveStake(tokenId)
            }`
        );

        const status = await stargateContract.getDelegationStatus(tokenId);
        expect(status).to.equal(2n); // ACTIVE
        log(" Delegation is ACTIVE");

        log("\n Requesting delegation exit for dust user...");
        await stargateContract.connect(dustUser).requestDelegationExit(tokenId);

        // 3. FAST FORWARD TIME
        const [periodDurationFastForward, startBlockFastForward] =
            await protocolStakerMock.getValidationPeriodDetails(deployer.address);
        
        log(`\n Fast forwarding ${TEST_MAX_PERIODS + 2} periods to exceed the limit...`);

        for (let i = 0; i < TEST_MAX_PERIODS + 2; i++) {
            await fastForwardValidatorPeriods(
                Number(periodDurationFastForward),
                Number(startBlockFastForward),
                0
            );
            await protocolStakerMock.helper__setValidationCompletedPeriods(
                deployer.address,
                2 + i
            );

            const [claimableFirst, claimableLast] =
                await stargateContract.claimableDelegationPeriods(tokenId);
            if (claimableFirst > 0) {
                const totalStake = await stargateContract.getDelegatorsEffectiveStake(
                    deployer.address,
                    claimableLast
                );
                log(
                    ` After FF period ${i + 1}: claimableFirst=${claimableFirst}, claimableLast=${claimableLast}, totalStake=${totalStake}`
                );
            }
        }

        // 4. VERIFY THE TRAP (Unstake attempt #1)
        log("\n Attempting unstake (should fail)...");
        await expect(
            stargateContract.connect(dustUser).unstake(tokenId)
        ).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");
        log(" Unstake reverted as expected (MaxClaimablePeriodsExceeded)");

        // 5. THE ATTEMPTED FIX (Claim Rewards)
        log("\n Attempting to claim rewards...");
        
        const pendingRewards = await stargateContract["claimableRewards(uint256)"](tokenId);
        log(` Pending Rewards calculated by contract: ${pendingRewards.toString()} wei`);
        const [claimFirst, claimLast] = await stargateContract.claimableDelegationPeriods(tokenId);
        log(` Claimable delegation periods before claim: first=${claimFirst} last=${claimLast}`);
        const validatorStakeBefore = await stargateContract.getDelegatorsEffectiveStake(
            deployer.address,
            claimLast === 0 ? 0 : claimLast
        );
        const dustEffectiveStake = await stargateContract.getEffectiveStake(tokenId);
        log(` Validator total effective stake at last claimable period: ${validatorStakeBefore.toString()}`);
        log(` Dust user effective stake: ${dustEffectiveStake.toString()}`);
        log(
            ` Dust user share (approx) = ${
                validatorStakeBefore > 0n
                    ? (dustEffectiveStake * 1_000_000n) / validatorStakeBefore
                    : 0n
            } ppm`
        );
        
        if (pendingRewards === 0n) {
            log(" Rewards are exactly 0 wei due to integer division");
            log(" This reproduces the exact bug condition.");
        } else {
            console.warn(` WARNING: Rewards are ${pendingRewards} wei (not 0)`);
        }

        const claimTx = await stargateContract.connect(dustUser).claimRewards(tokenId);
        await claimTx.wait();
        log(" Claim transaction executed successfully");

        // 6. VERIFY THE DEADLOCK (Unstake attempt #2)
        log("\n Attempting unstake again after claiming...");

        if (pendingRewards === 0n) {
            await expect(
                stargateContract.connect(dustUser).unstake(tokenId)
            ).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");

            log("DEADLOCK CONFIRMED: claimRewards(0) completed but did not update lastClaimedPeriod");
            log("User is permanently locked and cannot unstake");
        } else {
            log(" System working correctly: rewards > 0, so lastClaimedPeriod was updated");
        }
    });
});
// (PoC code block ends)
```

## Required changes (provided in report)

The report includes three required edits used to run the PoC. These are reproduced here verbatim.

{% stepper %}
{% step %}

### 1. Mock Configuration

File: contracts/mocks/ProtocolStakerMock.sol\
Location: Line 292 in function getDelegatorsRewards()

Change:

```diff
// OLD (line 292):
- return 0.1 ether;

// NEW:
// Low value to simulate scenarios where integer division causes rounding to zero
+ return 1000;
```

Reason: Simulate low VTHO generation conditions where dust delegators' rewards round to zero.
{% endstep %}

{% step %}

### 2. Hardhat Configuration

File: hardhat.config.ts\
Location: Lines 76-81 (hardhat network) and 82-90 (vechain\_solo network).

Changes:

```diff
// OLD:
hardhat: {
    chainId: 1337,
},

// NEW:
hardhat: {
    chainId: 1337,
    accounts: {
        count: 100,
    },
},
```

And:

```diff
vechain_solo: {
    url: getSoloUrl(),
-   accounts: {
-       mnemonic: getEnvMnemonic(),
-       count: 20,  // (OLD)
-       path: VECHAIN_DERIVATION_PATH,
-   },
+   accounts: {
+       mnemonic: getEnvMnemonic(),
+       count: 100,  // Changed from 20 to 100
+       path: VECHAIN_DERIVATION_PATH,
+   },
    gas: 10000000,
},
```

Reason: The test requires 96+ accounts (95 mega whales + 1 dust user + deployer).
{% endstep %}

{% step %}

### 3. Deploy Helper

File: test/helpers/deploy.ts\
Location: Lines 57-64

Change:

```diff
// OLD (lines 58-62):
if (network.name === "hardhat") {
    const newBalance = ethers.parseEther("50000000");
    for (let i = 0; i < 5; i++) {
        await setBalance(otherAccounts[i].address, newBalance);
    }
}

// NEW:
if (network.name === "hardhat") {
    // Seed otherAccounts with 50M VET - highest node value is currently worth 15.6M VET
    const newBalance = ethers.parseEther("50000000");
    const accountsToFund = Math.min(100, otherAccounts.length);
    for (let i = 0; i < accountsToFund; i++) {
        await setBalance(otherAccounts[i].address, newBalance);
    }
}
```

Reason: Prevents array out-of-bounds errors when funding test accounts.
{% endstep %}
{% endstepper %}

## References

* Target repository: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

(Report includes PoC test and mock/hardhat changes above.)

***

If you want, I can:

* produce a minimal patch suggestion (contract-side) to fix the issue (e.g., ensure `lastClaimedPeriod` is advanced even when rewards are zero, but only up to a safe clamped bound), or
* convert the PoC test into a smaller reproducer or a transaction sequence for on-chain verification.

Which would you prefer?
