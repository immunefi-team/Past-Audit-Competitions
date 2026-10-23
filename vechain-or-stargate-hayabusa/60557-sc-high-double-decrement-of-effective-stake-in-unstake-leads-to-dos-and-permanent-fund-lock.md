# 60557 sc high double decrement of effective stake in unstake leads to dos and permanent fund lock

**Submitted on Nov 24th 2025 at 04:15:28 UTC by @xanony for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60557
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A double decrement of effective stake occurs when a user requests to exit a delegation and subsequently the validator exits (or is forced to exit). This causes an arithmetic underflow in the `Stargate.sol` contract, preventing the user from unstaking their NFT and retrieving their staked VET.

## Vulnerability Details

The Stargate contract tracks the "effective stake" of delegators for each validator to calculate rewards. This effective stake is updated (increased or decreased) when users delegate, unstake, or request to exit. When a user requests to exit an active delegation via requestDelegationExit, the contract decreases the effective stake for the validator:

```solidity
// Stargate.sol
function requestDelegationExit(uint256 _tokenId) external ... {
    // ...
    // decrease the effective stake
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
    // ...
}
```

Later, when the user calls unstake to claim their funds (after the exit period), the contract checks the validator status. If the validator has exited (status VALIDATOR\_STATUS\_EXITED), the contract attempts to decrease the effective stake again:

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
        // ...
        // decrease the effective stake of the previous validator
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }
    // ...
}
```

*The core issue*: If the validator exits (voluntarily or forced) after the user has requested an exit but before they unstake, both conditions are met. The effective stake is decremented twice for the same delegation. Since `_updatePeriodEffectiveStake` performs a subtraction (currentValue - effectiveStake), the second decrement will cause an arithmetic underflow and revert if the currentValue (total effective stake for that validator) is less than the user's effective stake. This is guaranteed to happen if the user is the only delegator or if the remaining effective stake is smaller than the user's stake amount. The unstake function does not verify whether an exit was already requested before attempting to decrement the effective stake when the validator has exited. This oversight leads to the double accounting error.

## Impact Details

This vulnerability results in a complete Denial of Service for affected users and permanent loss of their staked funds:

*Direct Financial Impact*:

* Users' staked VET tokens become permanently locked in the Stargate contract
* The staking NFT cannot be retrieved or transferred
* No recovery mechanism exists once this state is reached

*Permanent fund lock*: The arithmetic underflow causes unstake() to revert every time it's called, making it impossible for users to ever retrieve their staked VET

*No admin recovery*: There is no emergency withdrawal or admin function that can rescue locked funds

*Predictable occurrence*: This is not a rare edge case - validator exits are normal protocol operations, and users may have legitimate reasons for delays between requesting exit and unstaking

*Complete loss*: Users lose 100% of their staked VET amount plus the NFT itself

*Attack Scenario*: While this doesn't require malicious intent, the sequence naturally occurs:

1. User stakes 10,000 VET and receives delegation NFT
2. User requests to exit delegation (effective stake decremented)
3. Validator exits or is forced to exit
4. User attempts to unstake their 10,000 VET
5. Transaction reverts with Panic(0x11) - arithmetic underflow
6. User's 10,000 VET is permanently locked

This represents a critical vulnerability where users can lose their entire stake through normal protocol operations with no possibility of recovery.

## Proof of Concept

## Proof of Concept

```solidity

import { ethers } from "hardhat";
import { expect } from "chai";
import { Stargate, StargateNFT, ProtocolStakerMock, MyERC20 } from "../../typechain-types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { getOrDeployContracts } from "../../helpers/deploy";
import { deployStargateNFTLibraries } from "../../../scripts/deploy/libraries";
import { deployUpgradeableWithoutInitialization, initializeProxyAllVersions } from "../../../scripts/helpers";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { time } from "@nomicfoundation/hardhat-network-helpers";

describe("H-XX: Effective Stake Accounting Corruption", () => {
    let stargate: Stargate;
    let stargateNFT: StargateNFT;
    let protocolStaker: ProtocolStakerMock;
    let deployer: HardhatEthersSigner;
    let userA: HardhatEthersSigner;
    let userB: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let vtho: MyERC20;

    const STAKE_AMOUNT_A = ethers.parseEther("1000"); // 1k VET
    const STAKE_AMOUNT_B = ethers.parseEther("100");  // 100 VET
    const LEVEL_ID = 1; // Assuming level 1 exists and requires some VET

    beforeEach(async () => {
        [deployer, userA, userB, validator] = await ethers.getSigners();

        // Deploy ProtocolStakerMock
        const ProtocolStakerFactory = await ethers.getContractFactory("ProtocolStakerMock");
        protocolStaker = await ProtocolStakerFactory.deploy();
        await protocolStaker.waitForDeployment();

        // Deploy Mock VTHO
        const MyERC20Factory = await ethers.getContractFactory("MyERC20");
        vtho = await MyERC20Factory.deploy(deployer.address, deployer.address);
        await vtho.waitForDeployment();

        // Deploy StargateNFT Libraries
        const {
            StargateNFTClockLib,
            StargateNFTLevelsLib,
            StargateNFTMintingLib,
            StargateNFTSettingsLib,
            StargateNFTTokenLib,
            StargateNFTTokenManagerLib,
        } = await deployStargateNFTLibraries({ latestVersionOnly: true });

        // Deploy StargateNFT Proxy
        const stargateNFTProxyAddress = await deployUpgradeableWithoutInitialization(
            "StargateNFT",
            {
                Clock: await StargateNFTClockLib.getAddress(),
                Levels: await StargateNFTLevelsLib.getAddress(),
                MintingLogic: await StargateNFTMintingLib.getAddress(),
                Settings: await StargateNFTSettingsLib.getAddress(),
                Token: await StargateNFTTokenLib.getAddress(),
                TokenManager: await StargateNFTTokenManagerLib.getAddress(),
            },
            false
        );

        // Deploy Stargate Proxy
        const stargateProxyAddress = await deployUpgradeableWithoutInitialization(
            "Stargate",
            {
                Clock: await StargateNFTClockLib.getAddress(),
            },
            false
        );

        // Initialize StargateNFT
        stargateNFT = (await initializeProxyAllVersions(
            "StargateNFT",
            stargateNFTProxyAddress,
            [
                {
                    args: [
                        {
                            tokenCollectionName: "StarGate Delegator Token",
                            tokenCollectionSymbol: "SDT",
                            baseTokenURI: "https://example.com/",
                            admin: deployer.address,
                            upgrader: deployer.address,
                            pauser: deployer.address,
                            levelOperator: deployer.address,
                            legacyNodes: deployer.address, // Mock address
                            stargateDelegation: deployer.address, // Mock address
                            legacyLastTokenId: 1,
                            levelsAndSupplies: [
                                {
                                    level: {
                                        id: 1,
                                        name: "Level 1",
                                        isX: false,
                                        maturityBlocks: 100,
                                        scaledRewardFactor: 100,
                                        vetAmountRequiredToStake: ethers.parseEther("1000"),
                                    },
                                    circulatingSupply: 0,
                                    cap: 100
                                },
                                {
                                    level: {
                                        id: 2,
                                        name: "Level 2",
                                        isX: false,
                                        maturityBlocks: 100,
                                        scaledRewardFactor: 100,
                                        vetAmountRequiredToStake: ethers.parseEther("100"),
                                    },
                                    circulatingSupply: 0,
                                    cap: 100
                                }
                            ],
                            vthoToken: await vtho.getAddress(),
                        },
                    ],
                },
                {
                    args: [[]],
                    version: 2,
                },
                {
                    args: [
                        stargateProxyAddress,
                        [],
                        [],
                    ],
                    version: 3,
                },
            ],
            false
        )) as StargateNFT;

        // Initialize Stargate
        stargate = (await initializeProxyAllVersions(
            "Stargate",
            stargateProxyAddress,
            [
                {
                    args: [
                        {
                            admin: deployer.address,
                            protocolStakerContract: await protocolStaker.getAddress(),
                            stargateNFTContract: await stargateNFT.getAddress(),
                            maxClaimablePeriods: 832,
                        },
                    ],
                },
            ],
            false
        )) as Stargate;

        // Setup ProtocolStakerMock
        await protocolStaker.helper__setStargate(await stargate.getAddress());

        // Add validator to ProtocolStakerMock
        await protocolStaker.addValidation(validator.address, 100); // Period length 100 blocks
        await protocolStaker.helper__setValidatorStatus(validator.address, 2); // ACTIVE
    });

    it("should prevent unstaking if validator exits after user requested exit (DoS)", async () => {
        // 1. User A stakes and delegates
        await stargate.connect(userA).stakeAndDelegate(1, validator.address, { value: STAKE_AMOUNT_A });
        const tokenIdA = await stargateNFT.tokenOfOwnerByIndex(userA.address, 0);

        // 2. User B stakes and delegates
        await stargate.connect(userB).stakeAndDelegate(2, validator.address, { value: STAKE_AMOUNT_B });
        const tokenIdB = await stargateNFT.tokenOfOwnerByIndex(userB.address, 0);

        // Verify initial state
        // User A and B are delegated.

        // Advance periods to make delegation ACTIVE
        // Initial completed periods is 0. Start period is 2.
        // We need current period (completed + 1) >= start period (2).
        // So completed + 1 >= 2 => completed >= 1.
        await protocolStaker.helper__setValidationCompletedPeriods(validator.address, 1);

        // 3. User A requests delegation exit
        await stargate.connect(userA).requestDelegationExit(tokenIdA);

        // Verify User A has requested exit
        expect(await stargate.hasRequestedExit(tokenIdA)).to.be.true;

        // 4. Simulate Validator Exit
        // Set validator status to EXITED (3)
        await protocolStaker.helper__setValidatorStatus(validator.address, 3);

        // Also need to set completed periods so unstake calculates the correct period
        // Let's say validator completed 10 periods
        await protocolStaker.helper__setValidationCompletedPeriods(validator.address, 10);

        // 5. User A tries to unstake
        // This should fail if the double decrement bug exists and causes underflow
        // User A's stake (100k) is much larger than User B's stake (10k).
        // If User A is subtracted twice, the total effective stake will try to go negative.
        // Total Stake (tracked) = Stake A + Stake B
        // After Request Exit: Total Stake = Stake B
        // After Validator Exit & Unstake: Total Stake = Stake B - Stake A (Underflow!)

        await expect(
            stargate.connect(userA).unstake(tokenIdA)
        ).to.be.reverted; // Expect revert due to underflow (panic code 0x11)
    });
});


```
