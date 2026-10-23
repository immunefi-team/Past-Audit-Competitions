# 59443 sc high rithmetic underflow in effective stake accounting causes permanent loss of funds

**Submitted on Nov 12th 2025 at 11:49:21 UTC by @flora for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59443
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### 1. Summary

A logic flaw in the stake accounting mechanism leads to a double-decrease of a user's effective stake under specific, non-user-controlled conditions. When a user requests a delegation exit and the associated validator subsequently exits the network or has its status changed to `QUEUED`, any attempt by the user to `unstake` or re-`delegate` will trigger an arithmetic underflow. This reverts the transaction, permanently locking the user's VET principal in the protocol with no available recovery mechanism for the user.

### 2. Vulnerability Details

{% stepper %}
{% step %}

### Step

A user stakes VET and delegates to a validator. A checkpoint is created for their `effectiveStake`.
{% endstep %}

{% step %}

### Step

The user calls `requestDelegationExit()`. This function correctly decreases the `effectiveStake` for future periods by writing a new checkpoint with a value of zero. This constitutes the first decrease.
{% endstep %}

{% step %}

### Step

The validator's on-chain status changes. This can happen in two ways:

* The validator exits the network (its status becomes `VALIDATOR_STATUS_EXITED`).
* The validator's status is changed to `QUEUED`, which in turn changes the user's `delegation.status` to `PENDING`.
  {% endstep %}

{% step %}

### Step

The user then calls `unstake()` or `delegate()` to a new validator. Both functions contain a conditional check that is now satisfied due to the validator's status change:

```solidity
// Location: Stargate.sol#unstake()
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    // ...
    _updatePeriodEffectiveStake(
        // ...
        false // decrease
    );
}
```

This block is now executed, triggering the second decrease on a stake that has already been zeroed out in preparation for exit.
{% endstep %}

{% step %}

### Step

Inside `_updatePeriodEffectiveStake()`, the `Checkpoints.upperLookup` function retrieves the most recent checkpoint value, which is `0` (from the first decrease). The code then attempts to calculate `currentValue - effectiveStake` (i.e., `0 - effectiveStake`), which causes an arithmetic underflow and reverts the entire transaction.
{% endstep %}
{% endstepper %}

### 3. Impact

* Permanent Loss of Funds: Users who encounter this scenario will have their VET tokens permanently locked in the `ProtocolStaker` contract. The `unstake` and `delegate` functions become irrevocably unusable for their NFT, preventing them from ever retrieving their principal investment.
* Protocol Insolvency for Affected Delegations: The protocol's liability (the `vetAmountStaked` recorded in the NFT) can no longer be settled. This breaks the core accounting invariant that all staked funds must be redeemable.
* No User-Side Mitigation: Once funds are locked, there is no action a user can take to recover them. The only recovery path is through a privileged contract upgrade performed by the `DEFAULT_ADMIN_ROLE`.
* Systemic Risk: While the trigger is conditional on validator status, validator churn is an expected and normal part of any proof-of-stake network's lifecycle. Therefore, any user who has requested to exit their delegation is at risk of permanent fund loss if their validator exits or is temporarily suspended for any reason. The vulnerability punishes users for following the intended protocol flow.

### 4. References

* Proof of Concept: [`packages/contracts/test/unit/Stargate/Finding5_POC.test.ts`](broken://pages/b8f3aecdfdfe5a5ed2cd9b3b6dcbbd9035616b55)
* Flawed Logic in `unstake()`: [`packages/contracts/contracts/Stargate.sol#L266-L283`](broken://pages/ed14e4025cdeb5e726ede3c752eb32f2f3462486#L266-L283)
* Flawed Logic in `delegate()`: [`packages/contracts/contracts/Stargate.sol#L398-L413`](broken://pages/ed14e4025cdeb5e726ede3c752eb32f2f3462486#L398-L413)
* Underflow Location in `_updatePeriodEffectiveStake()`: [`packages/contracts/contracts/Stargate.sol#L994-L1012`](broken://pages/ed14e4025cdeb5e726ede3c752eb32f2f3462486#L994-L1012)

## Proof of Concept

```typescript
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

describe("Finding 5: Double Decrease Effective Stake on Exit", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;
    const VALIDATOR_STATUS_QUEUED = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;
    const DELEGATION_STATUS_PENDING = 1;
    const DELEGATION_STATUS_ACTIVE = 2;

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
        
        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[2];

        // Add validator
        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();

        // Set the stargate contract address for withdrawals
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        
        // Set validator status to ACTIVE
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // Configure level in stargateNFTMock
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        // Mint VTHO to stargate contract for rewards
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("50000000"));
        await tx.wait();
    });

    describe("Validator exits network after user requests exit", () => {
        it("should revert unstake due to double effective stake decrease", async () => {
            const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
            const stakeAmount = levelSpec.vetAmountRequiredToStake;

            // Setup: Stake and delegate
            tx = await stargateContract.connect(user).stake(LEVEL_ID, { value: stakeAmount });
            await tx.wait();
            const tokenId = await stargateNFTMock.getCurrentTokenId();

            tx = await stargateNFTMock.helper__setToken({
                tokenId: tokenId,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: stakeAmount,
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            });
            await tx.wait();
            tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
            await tx.wait();

            tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 0);
            await tx.wait();

            tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
            await tx.wait();

            // Make delegation ACTIVE
            tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
            await tx.wait();

            const delegation = await stargateContract.getDelegationDetails(tokenId);
            expect(delegation.status).to.equal(DELEGATION_STATUS_ACTIVE);

            // User requests exit (first decrease executed here)
            tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
            await tx.wait();

            // Validator exits network
            tx = await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);
            await tx.wait();

            const [, , , , validatorStatus] = await protocolStakerMock.getValidation(validator.address);
            expect(validatorStatus).to.equal(VALIDATOR_STATUS_EXITED);

            // Attempting unstake should revert (second decrease causes underflow)
            await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;

            // Verify funds are locked
            const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
            const [, delStake] = await protocolStakerMock.getDelegation(delegationId);
            const token = await stargateNFTMock.getToken(tokenId);
            
            console.log(`Locked stake: ${ethers.formatEther(delStake)} VET`);
            console.log(`Protocol debt: ${ethers.formatEther(token.vetAmountStaked)} VET`);
        });
    });

    describe("Validator status changes to QUEUED after user requests exit", () => {
        it("should revert unstake when delegation becomes PENDING", async () => {
            const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
            const stakeAmount = levelSpec.vetAmountRequiredToStake;

            // Setup: Stake and delegate
            tx = await stargateContract.connect(user).stake(LEVEL_ID, { value: stakeAmount });
            await tx.wait();
            const tokenId = await stargateNFTMock.getCurrentTokenId();

            tx = await stargateNFTMock.helper__setToken({
                tokenId: tokenId,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: stakeAmount,
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            });
            await tx.wait();
            tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
            await tx.wait();

            tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 0);
            await tx.wait();

            tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
            await tx.wait();

            // Make delegation ACTIVE
            tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
            await tx.wait();

            // User requests exit (first decrease)
            tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
            await tx.wait();

            // Validator becomes QUEUED (delegation becomes PENDING)
            tx = await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_QUEUED);
            await tx.wait();

            const [, , , , validatorStatus] = await protocolStakerMock.getValidation(validator.address);
            expect(validatorStatus).to.equal(VALIDATOR_STATUS_QUEUED);

            const delegation = await stargateContract.getDelegationDetails(tokenId);
            console.log(`Delegation status: ${delegation.status} (PENDING)`);

            // Attempting unstake should revert (second decrease via PENDING check)
            await expect(stargateContract.connect(user).unstake(tokenId)).to.be.reverted;
        });
    });

    describe("User cannot re-delegate after exit request when validator exits", () => {
        it("should revert re-delegation due to double effective stake decrease", async () => {
            // Setup second validator
            const otherValidator = (await ethers.getSigners())[3];
            tx = await protocolStakerMock.addValidation(otherValidator.address, 120);
            await tx.wait();
            tx = await protocolStakerMock.helper__setValidatorStatus(otherValidator.address, VALIDATOR_STATUS_ACTIVE);
            await tx.wait();

            const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
            const stakeAmount = levelSpec.vetAmountRequiredToStake;

            // Setup: Stake and delegate to validator 1
            tx = await stargateContract.connect(user).stake(LEVEL_ID, { value: stakeAmount });
            await tx.wait();
            const tokenId = await stargateNFTMock.getCurrentTokenId();

            tx = await stargateNFTMock.helper__setToken({
                tokenId: tokenId,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: stakeAmount,
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            });
            await tx.wait();
            tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
            await tx.wait();

            tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 0);
            await tx.wait();

            tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
            await tx.wait();

            // User requests exit (first decrease)
            tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
            await tx.wait();

            // Validator 1 exits network
            tx = await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);
            await tx.wait();

            // Attempting to re-delegate should revert (delegate() also has double decrease logic)
            await expect(stargateContract.connect(user).delegate(tokenId, otherValidator.address)).to.be.reverted;

            console.log("User cannot unstake OR re-delegate - funds permanently locked");
        });
    });
});
```

## Output

```bash
(base) ➜  audit-comp-vechain-stargate-hayabusa git:(main) ✗ cd packages/contracts
VITE_APP_ENV=local yarn hardhat test test/unit/Stargate/Finding5_POC.test.ts --network hardhat
yarn run v1.22.22
$ /Users/test/Documents/web3-audit/2025/Imm/audit-comp-vechain-stargate-hayabusa/node_modules/.bin/hardhat test test/unit/Stargate/Finding5_POC.test.ts --network hardhat
WARNING: You are currently using Node.js v18.20.8, which is not supported by Hardhat. This can lead to unexpected behavior. See https://hardhat.org/nodejs-versions



****************** WARNING: You are operating on a non-vechain network ******************
- Ensure your hardhat config file has a network that:
- 	1. Is a VeChain valid network (set url and optionally gasPayer parameter)
- 	2. Has the name of the network containing "vechain" (e.g. "vechain_mainnet", "vechain_testnet", "vechain_solo", ...)
-
- This is required to use the VeChain provider and its functions.
- Note that this is only a warning and you can use hardhat without a VeChain network.
- BUT it's possible that some functionalities will not be available.

 ·------------------------|--------------------------------|--------------------------------·
 |  Solc version: 0.8.20  ·  Optimizer enabled: true       ·  Runs: 1                       │
 ·························|··························································
 |  Contract Name         ·  Deployed size (KiB) (change)  ·  Initcode size (KiB) (change)  │
 ·······················|·························································
 |  Clock                 ·                 0.452 (0.000)  ·                 0.509 (0.000)  │
 ·······················|······················································
 |  DataTypes             ·                 0.084 (0.000)  ·                 0.138 (0.000)  │
 ·······················|················································
 |  Errors                ·                 0.084 (0.000)  ·                 0.138 (0.000)  │
 ·······················|·······································
 |  Levels                ·                 7.115 (0.000)  ·                 7.172 (0.000)  │
 ·······················|········································
 |  MintingLogic          ·                 6.549 (0.000)  ·                 6.605 (0.000)  │
 ·······················|······································
 |  Settings              ·                 0.966 (0.000)  ·                 1.022 (0.000)  │
 ·······················|······································
 |  Stargate              ·                19.478 (0.000)  ·                19.731 (0.000)  │
 ·······················|····································
 |  StargateNFT           ·                23.061 (0.000)  ·                23.314 (0.000)  │
 ·······················|··································
 |  StargateProxy         ·                 0.166 (0.000)  ·                 1.016 (0.000)  │
 ·······················|·································
 |  Token                 ·                 4.592 (0.000)  ·                 4.648 (0.000)  │
 ·······················|·································
 |  TokenManager          ·                 4.828 (0.000)  ·                 4.885 (0.000)  │
 ·------------------------|--------------------------------|--------------------------------·


  Finding 5: Double Decrease Effective Stake on Exit
    Validator exits network after user requests exit
Locked stake: 1.0 VET
Protocol debt: 1.0 VET
      ✔ should revert unstake due to double effective stake decrease
    Validator status changes to QUEUED after user requests exit
Delegation status: 1 (PENDING)
      ✔ should revert unstake when delegation becomes PENDING
    User cannot re-delegate after exit request when validator exits
User cannot unstake OR re-delegate - funds permanently locked
      ✔ should revert re-delegation due to double effective stake decrease


  3 passing (801ms)

✨  Done in 3.44s.
```
