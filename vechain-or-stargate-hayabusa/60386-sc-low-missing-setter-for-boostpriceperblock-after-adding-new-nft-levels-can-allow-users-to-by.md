# 60386 sc low missing setter for boostpriceperblock after adding new nft levels can allow users to bypass intended staking boost

**Submitted on Nov 22nd 2025 at 03:57:19 UTC by @sedare for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60386
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In the StargateNFT staking system,when a new level is added via `addLevel`,there is no public/external function for the `LEVEL_OPERATOR_ROLE` to set or update `boostPricePerBlock`.As a result,newly added levels default to a `boostPricePerBlock` of 0.Users staking NFTs at these levels can earn rewards without paying for boost periods,potentially bypassing intended staking mechanics.

## Vulnerability Details

* `addLevel` allows the `LEVEL_OPERATOR_ROLE` to create new NFT levels.
* `updateLevelBoostPricePerBlock` exits in the Levels library but is **not exposed in the public StargateNFT contract,** meaning new levels cannot have boost prices set after creation.
* Newly added levels default to boost price of 0.
* Users staking on these levels can skip paying for boost rewards,effectively bypassing the staking boost mechanism.
* The intended access control\*\*(LEVEL\_OPERATOR\_ROLE)\*\* cannot correct this after level creation.

## Impact Details

* **Serverity:** Medium (economic /design impact rather than direct theft)
* **Impact:** Users may bypass intended staking mechanics,receiving boosts without cost.
* This does **not** directly steal funds or break core contract logic but could affect protocol incentives and fairness.

## References

\[StargateNFT.sol#L302]<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/StargateNFT.sol#L302>)

[Levels.sol#L88](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/libraries/Levels.sol#L88)

[Levels.sol#180](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/libraries/Levels.sol#L180)

## Proof of Concept

## Proof of Concept

```solidity
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { getOrDeployContracts, getStargateNFTErrorsInterface } from "../../helpers";
import {
    Errors,
    Stargate,
    StargateNFT,
    MyERC20,
    MyERC20__factory,
} from "../../../typechain-types";
import { TransactionResponse } from "ethers";

describe("shard-u101: StargateNFT: Levels", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    const config = createLocalConfig();

    let otherAccounts: HardhatEthersSigner[];
    let deployer: HardhatEthersSigner;
    let staker: HardhatEthersSigner;
    let stargateNFTContract: StargateNFT;
    let stargateContract: Stargate;
    let errorsInterface: Errors;
    let vthoTokenContract: MyERC20;
    let tx: TransactionResponse;

    beforeEach(async () => {
        [deployer] = await ethers.getSigners();

        // Deploy full protocol
        const contracts = await getOrDeployContracts({
            forceDeploy: true,
            config,
        });

        otherAccounts = contracts.otherAccounts;
        stargateNFTContract = contracts.stargateNFTContract;
        stargateContract = contracts.stargateContract;
        errorsInterface = await getStargateNFTErrorsInterface();

        // Deploy a fake VTHO token at the reserved VTHO address
        const vthoTokenFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();

        const bytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [
            VTHO_TOKEN_ADDRESS,
            bytecode,
        ]);

        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        // Mint VTHO to staker for rewards / stake operations
        staker = contracts.otherAccounts[0];
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(staker, ethers.parseEther("5000000"));
        await tx.wait();
    });

    describe("Add level", () => {
        it.only(
            "should add level with sequential ID and prove that missing boost price makes boost FREE",
            async () => {
                const levelOperator = otherAccounts[1];

                //
                // ---------------------- Setup: grant level operator role ----------------------
                //
                const grantTx = await stargateNFTContract.grantRole(
                    await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
                    levelOperator.address
                );
                await grantTx.wait();

                const currentLevelIds = await stargateNFTContract.getLevelIds();

                //
                // ---------------------- Define new level ----------------------
                //
                const newLevelAndSupply = {
                    level: {
                        name: "My New Level",
                        isX: false,
                        id: 0, // ignored
                        maturityBlocks: 30n,
                        scaledRewardFactor: 150n,
                        vetAmountRequiredToStake: ethers.parseEther("1"),
                    },
                    cap: 872,
                    circulatingSupply: 0,
                };

                const expectedLevelId =
                    currentLevelIds[currentLevelIds.length - 1] + 1n;

                //
                // ---------------------- Add new level ----------------------
                //
                const addLevelTx = await stargateNFTContract
                    .connect(levelOperator)
                    .addLevel(newLevelAndSupply);
                await addLevelTx.wait();

                //
                // ---------------------- Validate: sequential IDs ----------------------
                //
                expect(await stargateNFTContract.getLevelIds()).to.deep.equal([
                    ...currentLevelIds,
                    expectedLevelId,
                ]);

                //
                // ---------------------- Validate: level data ----------------------
                //
                const newLevel = await stargateNFTContract.getLevel(
                    expectedLevelId
                );

                expect(newLevel.name).to.equal(newLevelAndSupply.level.name);
                expect(newLevel.isX).to.equal(newLevelAndSupply.level.isX);
                expect(newLevel.vetAmountRequiredToStake).to.equal(
                    newLevelAndSupply.level.vetAmountRequiredToStake
                );
                expect(newLevel.scaledRewardFactor).to.equal(
                    newLevelAndSupply.level.scaledRewardFactor
                );
                expect(newLevel.maturityBlocks).to.equal(
                    newLevelAndSupply.level.maturityBlocks
                );

                //
                // ---------------------- Validate: supply struct ----------------------
                //
                const newLevelSupply =
                    await stargateNFTContract.getLevelSupply(expectedLevelId);

                expect(newLevelSupply.cap).to.equal(newLevelAndSupply.cap);
                expect(newLevelSupply.circulating).to.equal(
                    newLevelAndSupply.circulatingSupply
                );

                //
                // ---------------------- BUG PROOF: boost price defaults to ZERO ----------------------
                //
                // If LEVEL_OPERATOR doesn't able to call setBoostPrice(),
                // the contract incorrectly treats boost price = 0,
                // making boosting completely FREE.
                //
                const boostPrice =
                    await stargateNFTContract.boostPricePerBlock(
                        expectedLevelId
                    );
                expect(boostPrice).to.equal(0n); // <---- BUG CONFIRMED

                //
                // ---------------------- Stake and mint tokenId ----------------------
                //
                const tokenId =
                    await stargateContract
                        .connect(staker)
                        .stake.staticCall(expectedLevelId, {
                            value: newLevel.vetAmountRequiredToStake,
                        });

                await stargateContract.connect(staker).stake(expectedLevelId, {
                    value: newLevel.vetAmountRequiredToStake,
                });

                //
                // ---------------------- Call boost() → should cost 0 native token ----------------------
                //
                const userBalanceBefore = await ethers.provider.getBalance(staker.address);
                const boostTx = await stargateNFTContract.connect(staker).boost(tokenId);
                const receipt = (await boostTx.wait())!;
                const userBalanceAfter = await ethers.provider.getBalance(staker.address);

                // gas spent = gasUsed * effectiveGasPrice
                const gasCost = receipt.gasUsed * boostTx.gasPrice;

                // userBalanceBefore - userBalanceAfter should equal gasCost (no extra fee)
                const diff = userBalanceBefore - userBalanceAfter;

                // Allow < 0.001 ETH tolerance
                expect(diff - gasCost).to.be.lt(ethers.parseEther("0.001"));

                //
                // ---------------------- Sanity check: user should NOT be under maturity (boost worked) ----------------------
                //
                const underMaturity =
                    await stargateNFTContract.isUnderMaturityPeriod(tokenId);

                console.log("Under maturity period?", underMaturity);
    
            }
        );
    });
});

```
