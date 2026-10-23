# 60334 sc high unstake permanently reverts when validator exits after delegator exit double decrease of effective stake&#x20;

**Submitted on Nov 21st 2025 at 13:41:46 UTC by @Diavol0 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60334
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### High‑level overview

In the Hayabusa Stargate staking protocol, a delegator’s staking position is represented by an NFT. When the user wants to fully exit, the expected flow is:

1. While the validator is still active, the user calls `requestDelegationExit(tokenId)` to signal exit.
2. After the delegation has effectively ended and the validator’s current period has advanced, the user calls `unstake(tokenId)` to burn the NFT and withdraw their VET.

Due to the way effective stake accounting is implemented, there is a timing pattern where this flow **permanently breaks**:

* If the user calls `requestDelegationExit` while the validator is ACTIVE (which already decreases their effective stake for future periods), and
* Later the validator itself moves to `EXITED` before the user calls `unstake`,

then calling `unstake(tokenId)` triggers a **second decrease** of the same token’s effective stake. In the common case where this token was the only delegator (or the main one), this second decrease underflows and the entire `unstake` call reverts every time.

As a result, the user’s NFT can never be successfully unstaked through the normal interface, effectively **freezing their VET principal** in the protocol. There is no way for the user to recover the funds without an upgrade or administrative intervention.

This matches Immunefi’s **“Permanent freezing of funds”** category: user funds are not directly stolen or misallocated to another party, but the contract logic prevents the user from ever withdrawing their own stake under some timing conditions.

### Root cause

The bug is in the interaction between:

* How effective stake is tracked per validator and period via `_updatePeriodEffectiveStake`, and
* When and how that function is called in `requestDelegationExit` and `unstake`.

Relevant code locations:

* File: `packages/contracts/contracts/Stargate.sol`
  * `_updatePeriodEffectiveStake` (around lines 993–1015)
  * `requestDelegationExit` (around lines 523–571)
  * `unstake` (around lines 231–320)

#### Effective stake updates

Effective stake is updated with:

```solidity
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
        : currentValue - effectiveStake;

    $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
}
```

Key point:

* When `_isIncrease == false`, the function unconditionally computes `currentValue - effectiveStake` with checked arithmetic (Solidity 0.8), so if `currentValue < effectiveStake` it will **underflow and revert**.

#### Where decreases happen

1. **On delegation creation (inside `_delegate`)**, the effective stake is increased:

   ```solidity
   // After adding delegation
   (, , , uint32 completedPeriods) =
       $.protocolStakerContract.getValidationPeriodDetails(_validator);

   // Increase the delegators effective stake in the next period
   _updatePeriodEffectiveStake($, _validator, _tokenId, completedPeriods + 2, true);
   ```
2. **On `requestDelegationExit`**, regardless of whether the delegation is ACTIVE or PENDING, the function always performs a decrease at the end:

   ```solidity
   (, , , uint32 completedPeriods) =
       $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);
   (, uint32 exitBlock) =
       $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

   _updatePeriodEffectiveStake(
       $,
       delegation.validator,
       _tokenId,
       completedPeriods + 2,
       false
   );

   emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
   ```

   When the delegation is ACTIVE and the validator status is `VALIDATOR_STATUS_ACTIVE`, this is the **only** decrease, and it is correct.
3. **On `unstake`**, there is another conditional decrease:

   ```solidity
   (, , , , uint8 currentValidatorStatus, ) =
       $.protocolStakerContract.getValidation(delegation.validator);

   if (
       currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
       delegation.status == DelegationStatus.PENDING
   ) {
       (, , , uint32 oldCompletedPeriods) =
           $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);

       _updatePeriodEffectiveStake(
           $,
           delegation.validator,
           _tokenId,
           oldCompletedPeriods + 2,
           false // decrease
       );
   }
   ```

   This path is meant to cover cases where:

   * The validator was EXITED, or
   * The delegation was still PENDING and never actually started,

   so that any scheduled effective stake for future periods is cleared.

#### Problematic combination

The problematic scenario is when **both** `requestDelegationExit` and `unstake` execute their own decreases for the same delegation:

1. The user’s delegation is ACTIVE when `requestDelegationExit(tokenId)` is called.
2. This call immediately performs a decrease via `_updatePeriodEffectiveStake(..., false)` at a certain future period (based on the validator’s `completedPeriods`).
3. Later, before the user calls `unstake`, the validator’s status becomes `VALIDATOR_STATUS_EXITED` (via the trusted `ProtocolStaker` logic).
4. When the user now calls `unstake(tokenId)`, the `unstake` function sees `currentValidatorStatus == VALIDATOR_STATUS_EXITED` and executes **another** `_updatePeriodEffectiveStake(..., false)` for a (typically later) future period.

Because after step 2 the effective stake has already been subtracted from the `delegatorsEffectiveStake` trace for all future lookup periods, the second decrease in step 4 sees `currentValue == 0` from `upperLookup` at the chosen period, and attempts to compute:

```solidity
updatedValue = 0 - effectiveStake; // underflow
```

This underflow causes the entire `unstake` transaction to revert every time.

## Link to Proof of Concept

<https://gist.github.com/6newbie/42877df1c0596306589076b13d3a8ec2>

## Proof of Concept

Below is a minimal PoC as a Hardhat unit test. It can be added as a new test file: `packages/contracts/test/unit/Stargate/UnstakeExploit_C02.test.ts`.

### Setup

1. Environment
   * Network: Hardhat in‑memory network.
   * Contracts deployed via existing helper `getOrDeployContracts({ forceDeploy: true, config })` with a local config (`createLocalConfig()`), identical to the project’s own unit tests.
   * `ProtocolStakerMock` is used as the staking protocol backend.
2. Actors
   * `user`: delegator (victim) who will experience frozen funds.
   * `validator`: a single validator configured as ACTIVE in the `ProtocolStakerMock`.
3. NFT level configuration
   * Similar to other unit tests: one level with a fixed `vetAmountRequiredToStake` and `scaledRewardFactor`, so that the effective stake is non-zero and simple to reason about.

### Exploit scenario

1. User stakes and delegates an NFT to the validator.
2. The validator completes some periods so that the delegation becomes ACTIVE.
3. While the validator is still ACTIVE, the user calls `requestDelegationExit(tokenId)`.
   * This performs one `_updatePeriodEffectiveStake(..., false)` for a future period, correctly scheduling the removal of the user’s stake.
4. The validator’s status is then changed to EXITED, and its `completedPeriods` is increased (e.g. by the protocol).
5. The user now calls `unstake(tokenId)`:
   * Because the validator status is `EXITED`, `unstake` executes **another** `_updatePeriodEffectiveStake(..., false)`.
   * Since the first decrease has already reduced the future effective stake to zero, this second decrease underflows and causes the entire transaction to revert.
6. Any future attempt to `unstake(tokenId)` will hit the same logic and revert again, freezing the user’s funds.

### PoC code

```ts
import { expect } from "chai";
import {
    MyERC20,
    MyERC20__factory,
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
    Stargate,
    StargateNFTMock,
    StargateNFTMock__factory,
    TokenAuctionMock,
    TokenAuctionMock__factory,
} from "../../../typechain-types";
import { getOrDeployContracts } from "../../helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

describe("PoC: C-2 double-decrease causes permanent unstake revert", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    const LEVEL_ID = 1;

    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let vthoTokenContract: MyERC20;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        // Deploy protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy StargateNFT mock
        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        // Deploy VTHO token to the special energy address
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        // Deploy legacy nodes mock
        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        // Wire config for contracts deployment
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();

        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[2];

        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        // Add validator
        let tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // Configure level & token mocks
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 100, // 1x
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        tx = await stargateNFTMock.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();

        tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
        await tx.wait();

        // Fund Stargate with VTHO for rewards (not strictly needed here but keeps environment consistent)
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("1000000"));
        await tx.wait();
    });

    it("causes unstake to revert after double-decrease of effective stake", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // 1) User stakes and delegates to validator
        let tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        // 2) Make delegation ACTIVE by advancing completed periods
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();

        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        // 3) First decrease: request delegation exit while validator is ACTIVE
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // 4) Validator later becomes EXITED and more periods are completed
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();

        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 4);
        await tx.wait();

        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // 5) Second decrease: unstake will attempt to decrease effective stake again
        // which underflows and reverts, permanently preventing unstake
        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;
    });
});
```
