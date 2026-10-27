# 60373 sc high incorrect effective stake decrement when validator exits causes permanent freezing of user stake

**Submitted on Nov 21st 2025 at 22:40:57 UTC by @x0xmechanic for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60373
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When a validator exits after the delegator has already requested delegation exit, the protocol incorrectly decrements the validator’s effective stake a second time when the user later calls `unstake` or redelegates the NFT. This double decrement causes an arithmetic underflow in `_updatePeriodEffectiveStake`, making both `unstake` and `delegate` revert permanently for that token. As a result, the delegator cannot recover their staked VET, leading to **permanent freezing of funds (Critical)**.

## Vulnerability Details

The core issue is how `Stargate` updates a validator’s effective stake when:

* A **delegation has already EXITED**,
* The **validator itself has also EXITED**, and
* The user later calls **`unstake`** or **`delegate`** again.

Once a delegator has exited, the protocol has **already** decreased the validator’s effective stake for that token via `_updatePeriodEffectiveStake` as part of the normal exit flow when the user calls `requestDelegationExit`. After this point, `delegatorsEffectiveStake[validator].upperLookup(period)` correctly reflects the reduced value (often going to zero if this was the only delegator).

Later, when the user calls `unstake` after both:

* the **delegation is EXITED**, and
* the **validator is in status EXITED**,

we hit the branch:

```solidity
// if the delegation is pending or the validator is exited or unknown
// decrease the effective stake of the previous validator
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
)
```

Concretely, that branch does roughly this:

```solidity
function unstake(uint256 _tokenId) external {
    // ... load delegation + validator
    ( , , , uint32 completedPeriods) =
        $.protocolStakerContract.getValidationPeriodDetails(validator);

    
    _updatePeriodEffectiveStake(
        $,
        validator,
        _tokenId,
        completedPeriods + 2,
        false // "decrease" → subtract this token's stake
    );

    // ... burn NFT, send VET back, etc.
}
```

`_updatePeriodEffectiveStake` is called **again** with `false` (subtract), over periods where the first decrement has already been applied. This leads to:

* An **underflow (panic 0x11)** , or
* **Double subtraction** of their amount from the validator’s effective stake if other delegators remain.

The same double-decrement pattern appears when re-delegating, since `delegate` also calls `_updatePeriodEffectiveStake` under `currentValidatorStatus == VALIDATOR_STATUS_EXITED`. In both cases, once the validator and delegation have exited, subsequent `unstake` and re-`delegate` calls revert due to underflow in `_updatePeriodEffectiveStake`, preventing users from recovering or moving their staked funds.

## Impact Details

* **Critical – Permanent freezing of funds**

In the single delegator case, once:

* The delegation is EXITED, and
* The validator is EXITED

both:

* `unstake(tokenId)` and
* `delegate(tokenId, newValidator)`

permanently revert with panic `0x11` in `_updatePeriodEffectiveStake`. There is no alternative user accessible function that allows withdrawal of the staked VET. Their VET is **permanently frozen**.

In the multi-delegator case:

* The double decrement leads to incorrect `delegatorsEffectiveStake[validator]`, reducing it to zero while some delegations still exist.
* This can:
  * Underpay remaining delegators (they receive rewards as if there were no stake), and/or
  * Break any logic relying on effective stake (reward share, monitoring, etc.).

Even in the multi-delegator case, if the remaining delegators’ combined effective stake is smaller than the exiting delegator’s stake (because of different token levels), the second decrease in `_updatePeriodEffectiveStake` will underflow and cause their subsequent `unstake` or `delegate` calls to revert (freezing of funds).

## Proof of Concept

## Proof of Concept

We show the bug with two tests:

* **Test** "should revert when unstaking after validator exit due to effective stake underflow"

1. A user stakes and delegates the stake to a validator: Effective stake of the validator increases.
2. The user then calls `requestDelegationExit`: Effective stake of the validator becomes zero.
3. The validator exits.
4. The user calls `unstake`: The call reverts because of underflow in `_updatePeriodEffectiveStake`

Same with `delegate`: The user calls `delegate`: The call reverts because of underflow in `_updatePeriodEffectiveStake`

* **Test** "should reduce effective stake twice incorrectly"

1. Two users stake and delegate the stake to a validator: Effective stake of the validator increases by the sum of the two stakes.
2. One user then calls `requestDelegationExit`: Effective stake of the validator decreases by half.
3. The validator exits.
4. The user who exited calls `delegate`: The effective stake of the validator becomes zero (the user's stake was incorrectly subtracted again).

Here are the tests:

```solidity
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
import { log } from "../../../scripts/helpers/log";

describe("shard-u2: Stargate: ReportUnstake", () => {
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

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        // Deploy protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy stargateNFT mock
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

        // Deploy contracts
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        // get stargateNFT errors interface
        user = contracts.otherAccounts[0];
        otherUser = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];
        otherValidator = contracts.otherAccounts[3];
        otherAccounts = contracts.otherAccounts;

        // add default validator
        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();

        // set the stargate contract address so it can be used for
        // withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        // set the validator status to active by default so it can be delegated to
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // add other validator
        tx = await protocolStakerMock.addValidation(otherValidator.address, 120);
        await tx.wait();
        // set the stargate contract address so it can be used for
        // withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            otherValidator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // set the mock values in the stargateNFTMock contract
        // set get level response
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        // set get token response
        tx = await stargateNFTMock.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();

        // set the legacy nodes mock
        tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
        await tx.wait();

        // mint some VTHO to the stargate contract so it can reward users
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("50000000"));
        await tx.wait();
    });

    it("should assert the initial state", async () => {
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(0n);
        expect(tokenId).to.equal(10000);
        expect(user.address).to.not.be.equal(otherUser.address);
        expect(user.address).to.not.be.equal(deployer.address);
        expect(otherUser.address).to.not.be.equal(deployer.address);
        expect(await stargateNFTMock.isUnderMaturityPeriod(tokenId)).to.be.false;
        expect(await stargateNFTMock.isXToken(tokenId)).to.be.false;
    });

    it("should revert when unstaking after validator exit due to effective stake underflow", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        // delegate the NFT to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Correctly delegated the NFT to validator", validator.address);

        // check the delegation status
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_PENDING);

        // advance 1 period
        // so the delegation is active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();
        log("\n Set validator completed periods to 1 so the delegation is active");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        const effectiveStakeAfterFirstDelegation = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        2
        );

        console.log("\n🎉 Validator's effective stake after first delegation:"
            , effectiveStakeAfterFirstDelegation);

        // request exit
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Correctly requested to exit the delegation");
        // advance 1 period
        // so the delegation is exited
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        log("\n Set validator completed periods to 2 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        const effectiveStakeAfterValidatorExited = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        3
        );

        console.log("\n🎉 Validator's effective stake after delegator exited:"
            , effectiveStakeAfterValidatorExited);

        // set the validator status to exited
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();

        // Try to unstake
        await expect(
        stargateContract.connect(user).unstake(tokenId)
            ).to.be.revertedWithPanic(0x11);

        // Try to delegate again   
        await expect(
        stargateContract.connect(user).delegate(tokenId, otherValidator.address)
            ).to.be.revertedWithPanic(0x11);
    });
    it("should reduce effective stake twice incorrectly", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId1);

        // delegate the NFT to the validator
        tx = await stargateContract.connect(otherUser).delegate(tokenId1, validator.address);
        await tx.wait();
        log("\n🎉 Correctly delegated the NFT to validator", validator.address);
        
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        // delegate the NFT to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Correctly delegated the NFT to validator", validator.address);

        // check the delegation status
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_PENDING);

        // advance 1 period
        // so the delegation is active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();
        log("\n Set validator completed periods to 1 so the delegation is active");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        const effectiveStakeAfterFirstDelegation = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        2
        );

        console.log("\n🎉 Validator's effective stake after first delegation:"
            , effectiveStakeAfterFirstDelegation);

        // request exit
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Correctly requested to exit the delegation");
        // advance 1 period
        // so the delegation is exited
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        log("\n Set validator completed periods to 2 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );

        const effectiveStakeAfterValidatorExited = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        3
        );

        console.log("\n🎉 Validator's effective stake after delegator exited:"
            , effectiveStakeAfterValidatorExited);

        // set the validator status to exited
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();    

        // Delegate again
        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);

        const callTx = stargateContract.connect(user).delegate(tokenId, otherValidator.address);
        await expect(callTx)
            .to.emit(stargateContract, "DelegationWithdrawn")
            .withArgs(
                tokenId,
                validator.address,
                delegationId,
                levelSpec.vetAmountRequiredToStake,
                levelSpec.id
            );
        await expect(callTx).to.not.emit(stargateContract, "DelegationExitRequested");
        await expect(callTx).to.emit(stargateContract, "DelegationRewardsClaimed").withArgs(
            user.address,
            tokenId,
            delegationId,
            ethers.parseEther("0.05"), // fixed rewards in the mock
            2, // first claimable period, it entered in period 1
            2 // last claimable period, it requested exit in period 2
        );
        await expect(callTx)
            .to.emit(stargateContract, "DelegationInitiated")
            .withArgs(
                tokenId,
                otherValidator.address,
                delegationId + 1n, // new delegation id
                levelSpec.vetAmountRequiredToStake,
                LEVEL_ID,
                100 // probability multiplier
            );
        log("\n🎉 Correctly delegated back the NFT to validator", otherValidator.address);
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_PENDING
        );

        const effectiveStakeAfterSecondDelegation = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        4
        );

        console.log("\n🎉 Validator's effective stake after second delegation:"
            , effectiveStakeAfterSecondDelegation);

        expect(effectiveStakeAfterSecondDelegation).to.equal(ethers.parseEther("0"));    
    });

});
```

We used `TEST_LOGS=1 VITE_APP_ENV=local npx hardhat test --network hardhat test/unit/Stargate/ReportUnstake.test.ts`
