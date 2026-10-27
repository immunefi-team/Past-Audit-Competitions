# 60080 sc high unstake exit requests can either lock funds or silently double deduct effective stake after validator exit

**Submitted on Nov 18th 2025 at 10:02:01 UTC by @n0fr33w1f14u for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #60080
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Impact: Permanent freezing of funds

## Description

### Brief / Intro

The `_updatePeriodEffectiveStake()` function is called twice in the unstake flow:

* First decrease in `requestDelegationExit()` when delegation status is ACTIVE
* Later, if the validator status is EXITED or the delegation is still PENDING, `unstake()` decrements the same stake again

When both conditions happen in sequence, the second decrement occurs after the first already removed the position’s stake. If the validator doesn't have any effective stake recorded for that period (for example, the first call brought it to zero), the second subtraction tries to go below zero and triggers an underflow.

### Vulnerability Details

When a user calls `requestDelegationExit()` while delegation is ACTIVE, the function always decreases the effective stake:

```solidity
function requestDelegationExit(uint256 _tokenId) external ... {
    // ...
    if (delegation.status == DelegationStatus.ACTIVE) {
        $.protocolStakerContract.signalDelegationExit(delegationId);
    }
    
    // decrease the effective stake 
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
}
```

Later, when the validator exits and the user calls `unstake()`, the function checks if the validator is EXITED and decreases the effective stake again:

```solidity
function unstake(uint256 _tokenId) external ... {
    // ...
    
    // get the current validator status
    (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
        delegation.validator
    );
    
    // if the delegation is pending or the validator is exited or unknown
    // decrease the effective stake of the previous validator
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED || 
        delegation.status == DelegationStatus.PENDING
    ) {
        // get the completed periods of the previous validator
        (, , , uint32 oldCompletedPeriods) = $
            .protocolStakerContract
            .getValidationPeriodDetails(delegation.validator);

        // decrease the effective stake of the previous validator
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease again
        );
    }
    // ...
}
```

Since the effective stake was already decreased in `requestDelegationExit`, the second decrease in `unstake()` may cause an underflow: 0 - `effectiveStake`.

The `unstake()` function checks if the validator is EXITED to determine whether to decrease effective stake, but it does not check if the exit was already requested via `requestDelegationExit()`. Both functions can decrease the same stake.

## Impact Details

Any user who requests a delegation exit while the position is still active and later unstakes after the validator has exited will trigger an additional `_updatePeriodEffectiveStake(..., false)` inside `unstake()`. Two outcomes are possible:

* Underflow revert: If the first decrease brings the validator’s effective stake for that period down to zero, the second decrease subtracts from zero and reverts, permanently preventing the user from unstaking. Their staked VET remains locked in the contract.
* Silent loss of stake tracking: If other delegations still contribute positive stake to that period, the second decrease succeeds but reduces the validator’s recorded effective stake by the exiting token’s amount twice. This misaccounting silently deprives the validator’s delegators (including the exiting user) of future rewards and may cause downstream reward calculations to underpay or misallocate funds.

Either outcome is severe: users either can’t recover their staked funds or end up with incorrect reward accounting.

## Proof of Concept

This is the high-level flow that demonstrates the issue:

{% stepper %}
{% step %}

### Step: Stake and Delegate

1. User stakes a token and delegates to a validator.
2. Delegation becomes ACTIVE.
   {% endstep %}

{% step %}

### Step: Request Delegation Exit (first decrease)

3. User calls `requestDelegationExit()` while delegation is ACTIVE.
   * This call decreases the effective stake for the upcoming period (first decrement).
     {% endstep %}

{% step %}

### Step: Validator Exits

4. Validator exits (status becomes EXITED).
   {% endstep %}

{% step %}

### Step: Unstake (second decrease)

5. User calls `unstake()`.
   * `unstake()` checks validator status and, because it's EXITED, decreases the effective stake again for the same period (second decrement).
   * If the first decrease brought the recorded effective stake to zero, this second decrease will underflow and revert, locking the user's funds. Otherwise it will silently double-subtract and corrupt stake accounting.
     {% endstep %}
     {% endstepper %}

### Test environment

* OS: Windows 11, PowerShell
* Node.js: v22.20.0
* npm: 10.9.3
* Foundry: 1.3.5-stable
* Hardhat: 2.26.3

Create the file `DoubleDecreaseEffectiveStake.poc.test.ts` in `packages/contracts/test/unit/Stargate` to run this PoC.

Test command:

```bash
cd packages/contracts
yarn hardhat test --network hardhat test/unit/Stargate/DoubleDecreaseEffectiveStake.poc.test.ts
```

Environment variable: VITE\_APP\_ENV=local

PoC test (keep as-is — demonstrates revert with Panic code 0x11 when double-decrement occurs):

```javascript
//  packages/contracts/DoubleDecreaseEffectiveStake.poc.test.ts
// at packages/contracts run command below
// $env:VITE_APP_ENV="local"; yarn hardhat test --network hardhat test/unit/Stargate/DoubleDecreaseEffectiveStake.poc.test.ts
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

describe("POC: Double Decrease Effective Stake Bug", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let otherAccounts: HardhatEthersSigner[];
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;

    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        // protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        //  stargateNFT mock
        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        //  VTHO token to the energy address
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        //  legacy nodes mock
        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        //  contracts
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[2];
        otherAccounts = contracts.otherAccounts;

        // add default validator
        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();

        // set the stargate contract address so it can be used for withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();

        // set the validator status to active by default so it can be delegated to
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // set initial completed periods
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            10
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

        // mint VTHO to stargate contract
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract.target, ethers.parseEther("50000000"));
        await tx.wait();
    });

    it("demonstrate double decrease of effective stake when requestDelegationExit is called on active delegation and then unstake is called after validator exits", async () => {
        let currentPeriod = 10n;

        // Step 1: Stake a token
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        
        // Update the token in the mock after staking
        tx = await stargateNFTMock.helper__setToken({
            tokenId: Number(tokenId),
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: levelSpec.vetAmountRequiredToStake,
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();
    
        console.log("═════════════");
        
        console.log("\n Step 1: Staked token with id:", tokenId.toString());
        console.log(`Token ID: ${tokenId.toString()}`);
        console.log(`Staked Amount: ${ethers.formatEther(levelSpec.vetAmountRequiredToStake)} VET`);

        // Step 2: Delegate the token to validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        console.log("\n Step 2: Delegated token to validator");
        console.log(`Validator Address: ${validator.address}`);

        // Advance periods to make delegation ACTIVE
        currentPeriod = 12n;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            Number(currentPeriod - 1n)
        );
        await tx.wait();

        // Verify delegation is ACTIVE
        const delegationStatusBeforeExit = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatusBeforeExit).to.equal(DELEGATION_STATUS_ACTIVE);
        console.log("\n Step 3: Delegation is now ACTIVE");
        console.log(`Current Period: ${currentPeriod.toString()}`);
        console.log(`Delegation Status: ACTIVE (${delegationStatusBeforeExit.toString()})`);

        // Get the effective stake of the token
        const tokenEffectiveStake = await stargateContract.getEffectiveStake(tokenId);
        console.log(" Token effective stake:", tokenEffectiveStake.toString());

        // Check initial effective stake for the validator at next period (currentPeriod + 2)
        // This is the period where the delegation will be active
        const nextPeriod = currentPeriod + 2n;
        let effectiveStakeBeforeExit = await stargateContract.getDelegatorsEffectiveStake(
            validator.address,
            Number(nextPeriod)
        );
        console.log("Initial effective stake for validator at period", nextPeriod.toString() + ":", effectiveStakeBeforeExit.toString());
        expect(effectiveStakeBeforeExit).to.equal(tokenEffectiveStake);

        // Step 4: Request delegation exit while delegation is ACTIVE
        // This should decrease effective stake ONCE (BUG: This is the first decrease)
        console.log("\n call requestDelegationExit() - this will decrease effective stake");
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        console.log("\n Step 4: Requested delegation exit (delegation is ACTIVE)");
        console.log(`requestDelegationExit() completed`);

        // Check effective stake after requestDelegationExit
        // It should be decreased once
        let effectiveStakeAfterExitRequest = await stargateContract.getDelegatorsEffectiveStake(
            validator.address,
            Number(nextPeriod)
        );
        console.log(`Decreased from ${effectiveStakeBeforeExit.toString()} to ${effectiveStakeAfterExitRequest.toString()}`);
        expect(effectiveStakeAfterExitRequest).to.equal(0n, "Effective stake should be decreased to 0 after requestDelegationExit");

        // Step 5: Set validator status to EXITED
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        );
        await tx.wait();
        console.log("\n Step 5: Validator status set to EXITED");
        console.log(`Validator Status: EXITED (${VALIDATOR_STATUS_EXITED})`);

        // Advance periods
        currentPeriod = 20n;
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(
            validator.address,
            Number(currentPeriod - 1n)
        );
        await tx.wait();
        console.log(`Advanced to Period: ${currentPeriod.toString()}`);

        // Verify delegation status is exited
        const delegationStatusAfterValidatorExit = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatusAfterValidatorExit).to.equal(DELEGATION_STATUS_EXITED);
        console.log("\n Step 6: Delegation status is now EXITED");
        console.log(`Delegation Status: EXITED (${delegationStatusAfterValidatorExit.toString()})`);

        // Check effective stake before unstake (should still be 0 from the first decrease)
        let effectiveStakeBeforeUnstake = await stargateContract.getDelegatorsEffectiveStake(
            validator.address,
            Number(nextPeriod)
        );
        console.log(" Effective stake before unstake:", effectiveStakeBeforeUnstake.toString());
        expect(effectiveStakeBeforeUnstake).to.equal(0n);

        console.log("\n call unstake(), decrease effective stake again");
        console.log(`Current effective stake is: ${effectiveStakeBeforeUnstake.toString()}`);
        console.log(`Attempting to decrease will cause: 0 - ${tokenEffectiveStake.toString()}`);
        
        await expect(
            stargateContract.connect(user).unstake(tokenId)
        ).to.be.revertedWithPanic(0x11); // Panic code 0x11

    });

});
```

Notes:

* All links and repository paths are preserved as provided.
* The PoC test demonstrates the underflow revert (Panic 0x11) when the double-decrement occurs.
