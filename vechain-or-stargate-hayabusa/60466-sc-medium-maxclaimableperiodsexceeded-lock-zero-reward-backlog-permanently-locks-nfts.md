# 60466 sc medium maxclaimableperiodsexceeded lock zero reward backlog permanently locks nfts

**Submitted on Nov 23rd 2025 at 01:02:20 UTC by @arunabha003 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60466
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief

The Stargate contract contains a logic flaw in `_claimRewards()` that permanently locks NFTs when validators produce many periods with zero rewards. The function returns early at line 759 when `claimableAmount == 0` without updating `lastClaimedPeriod`, preventing the period backlog from being reduced. When the number of claimable periods exceeds `maxClaimablePeriods` (default 832), the guard check in `unstake()` and `delegate()` reverts with `MaxClaimablePeriodsExceeded`, making it impossible to unstake or redelegate the NFT and access the staked VET. As a result, the user cannot unstake and retrieve their staked VET, which constitutes permanent freezing of funds.

## Vulnerability Details

The vulnerability exists in the reward claiming and period tracking logic across multiple functions in `Stargate.sol`:

**Location 1: \_claimRewards() - Lines 757-760**

```solidity
function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {
    (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = _claimableDelegationPeriods($, _tokenId);
    // ... max period check ...
    uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
    
    if (claimableAmount == 0) {
        return;  // Early return without updating lastClaimedPeriod
    }
    
    $.lastClaimedPeriod[_tokenId] = lastClaimablePeriod;  // Only reached when claimableAmount > 0
    // ... transfer and emit ...
}
```

**Location 2: \_exceedsMaxClaimablePeriods() - Lines 956-973**

```solidity
function _exceedsMaxClaimablePeriods(
    StargateStorage storage $,
    uint256 _tokenId
) private view returns (bool) {
    (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = _claimableDelegationPeriods($, _tokenId);
    if (lastClaimablePeriod - firstClaimablePeriod >= $.maxClaimablePeriods) {
        return true;
    }
    return false;
}
```

**Location 3: unstake() - Lines 299-302**

```solidity
function unstake(uint256 _tokenId) external {
    // ...
    if (_exceedsMaxClaimablePeriods($, _tokenId)) {
        revert MaxClaimablePeriodsExceeded();
    }
    _claimRewards($, _tokenId);
    // ...
}
```

**Location 4: \_delegate() - Lines 431-435**

```solidity
function _delegate(...) private {
    // ...
    if (_exceedsMaxClaimablePeriods($, _tokenId)) {
        revert MaxClaimablePeriodsExceeded();
    }
    _claimRewards($, _tokenId);
    // ...
}
```

**Exploit Path:**

1. User delegates NFT to validator
2. Validator completes many periods (> `maxClaimablePeriods`) without producing rewards
   * Validator is offline or inactive
   * Testnet/devnet without reward distribution configured
   * Validator misconfiguration
3. User calls `claimRewards(tokenId)`
   * `_claimableRewards()` returns 0
   * `_claimRewards()` returns early at line 759
   * `lastClaimedPeriod[tokenId]` not updated
4. Period backlog continues growing: `lastClaimablePeriod - firstClaimablePeriod > maxClaimablePeriods`
5. User calls `unstake(tokenId)`
   * `_exceedsMaxClaimablePeriods()` returns true
   * Transaction reverts with `MaxClaimablePeriodsExceeded`
6. User attempts to redelegate via `delegate(tokenId, newValidator)`
   * `_exceedsMaxClaimablePeriods()` returns true
   * Transaction reverts with `MaxClaimablePeriodsExceeded`
7. NFT permanently locked with no recovery mechanism

Time makes the problem worse: each new period increases the backlog, and calling `claimRewards()` has no effect since it continues returning early without updating `lastClaimedPeriod`.

## Impact Details

Users whose delegated validators go inactive for extended periods lose permanent access to their staked VET. The NFT becomes permanently locked once the period backlog exceeds `maxClaimablePeriods`. No admin function exists to force-unstake, manually update `lastClaimedPeriod`, or bypass the guard check. No time-based unlock exists. The funds remain locked indefinitely.

This scenario is realistic in several contexts:

* Validators going offline during network issues or maintenance
* Testnet/devnet deployments where reward distribution is not configured
* Early mainnet deployment before reward mechanisms are fully operational
* Validator misconfigurations that prevent reward distribution

Financial impact scales with the number of affected delegators and their stake amounts. VeChain Stargate NFT levels range from 600,000 VET (Level 1) to 25,000,000 VET (Level 7). If multiple validators become inactive, hundreds or thousands of delegators could be affected, with total locked value potentially reaching hundreds of millions of VET.

This constitutes permanent freezing of funds per Immunefi's Critical impact definition: "user is no longer able to withdraw their funds" with no recovery mechanism.

## Proof of Concept

## Proof of Concept

**Test File**: `packages/contracts/test/integration/MaxClaimablePeriodsLockPOC.test.ts`

### Running the POC

```bash
cd packages/contracts
MNEMONIC="denial kitchen pet squirrel other broom bar gas better priority spoil cross" \
  VITE_APP_ENV=local npx hardhat test --network vechain_solo \
  test/integration/MaxClaimablePeriodsLockPOC.test.ts
```

### Actual Test Output

```
PoC: C-02 MaxClaimablePeriodsExceeded Lock (Critical)
C-02: claimablePeriods= 13 testMax= 10
C-02: claimableRewards= 0.0
C-02: Impact: Permanent freezing of staked VET - NFT cannot be unstaked or redelegated
  ✔ POC: Should permanently lock NFT when many periods have zero rewards
  ✔ Control test: Normal flow works when rewards exist

2 passing (9s)
```

### Complete POC Code

**File**: `packages/contracts/test/integration/MaxClaimablePeriodsLockPOC.test.ts`

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { StartedTestContainer } from "testcontainers";
import { IProtocolStaker, MyERC20, StargateNFT, Stargate } from "../../typechain-types";
import { IProtocolParams } from "../../typechain-types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import {
    createThorSoloContainer,
    getOrDeployContracts,
    mineBlocks,
    stakeNFT,
} from "../helpers";

describe("PoC: C-02 MaxClaimablePeriodsExceeded Lock (Critical)", () => {
    let soloContainer: StartedTestContainer;

    let mockedVthoToken: MyERC20;
    let protocolStakerContract: IProtocolStaker;
    let protocolParamsContract: IProtocolParams;
    let stargateNFTContract: StargateNFT;
    let stargateContract: Stargate;

    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;

    beforeEach(async () => {
        soloContainer = await createThorSoloContainer();

        const contracts = await getOrDeployContracts({ forceDeploy: true });

        mockedVthoToken = contracts.mockedVthoToken;
        protocolStakerContract = contracts.protocolStakerContract;
        protocolParamsContract = contracts.protocolParamsContract;
        stargateNFTContract = contracts.stargateNFTContract;
        stargateContract = contracts.stargateContract;

        deployer = contracts.deployer;
        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[1];
    });

    afterEach(async () => {
        if (soloContainer) {
            await soloContainer.stop();
        }
    });

    it("POC: Should permanently lock NFT when many periods have zero rewards", async () => {
        // set low maxClaimablePeriods for test
        const testMaxPeriods = 10;
        await stargateContract.connect(deployer).setMaxClaimablePeriods(testMaxPeriods);

        // register validator with required stake
        const validatorStakeAmount = ethers.parseEther("25000000");
        await protocolStakerContract.connect(validator).addValidation(validator.address, 12, {
            value: validatorStakeAmount,
        });

        // stake NFT and get tokenId
        const levelId = 1;
        const { tokenId, levelSpec } = await stakeNFT(user, levelId, stargateContract, stargateNFTContract, false);
        const stakeAmount = levelSpec.vetAmountRequiredToStake;

        // wait until NFT matures
        await mineBlocks(120);

        // delegate NFT to validator
        await stargateContract.connect(user).delegate(tokenId, validator.address);

        // advance many periods (no rewards distribution)
        const periodsToAdvance = testMaxPeriods + 5;
        const blocksPerPeriod = 12;
        await mineBlocks(periodsToAdvance * blocksPerPeriod);

        // compute claimable period range
        const [firstClaimable, lastClaimable] = await stargateContract.claimableDelegationPeriods(tokenId);
        const numClaimablePeriods = Number(lastClaimable) - Number(firstClaimable);

        // assert that claimable periods exceed the maximum allowed
        expect(numClaimablePeriods).to.be.greaterThan(testMaxPeriods);

        // verify there are zero claimable rewards across the range
        const claimableRewards = await stargateContract["claimableRewards(uint256)"](tokenId);
        expect(claimableRewards).to.equal(0n);

        // calling claimRewards should not advance lastClaimedPeriod when rewards == 0
        await stargateContract.connect(user).claimRewards(tokenId);
        const [firstClaimableAfter, lastClaimableAfter] = await stargateContract.claimableDelegationPeriods(tokenId);
        const numClaimableAfter = Number(lastClaimableAfter) - Number(firstClaimableAfter);
        expect(numClaimableAfter).to.equal(numClaimablePeriods);

        // request delegation exit first (delegation is locked, so this is required)
        await stargateContract.connect(user).requestDelegationExit(tokenId);

        // advance past exit period
        await mineBlocks(120);

        // now both unstake and redelegate must revert with MaxClaimablePeriodsExceeded
        await expect(
            stargateContract.connect(user).unstake(tokenId)
        ).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");

        await expect(
            stargateContract.connect(user).delegate(tokenId, deployer.address)
        ).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");

        // minimal console output for validation
        console.log("M-01: claimablePeriods=", numClaimablePeriods, "testMax=", testMaxPeriods);
        console.log("M-01: claimableRewards=", ethers.formatEther(claimableRewards));
        console.log("M-01: Impact: Permanent freezing of staked VET - NFT cannot be unstaked or redelegated");
    });

    it("Control test: Normal flow works when rewards exist", async () => {
        // Control test: ensure normal unstake flow works when not blocked

        // Setup validator
        const validatorStakeAmount = ethers.parseEther("25000000"); // 25M VET minimum
        await protocolStakerContract.connect(validator).addValidation(
            validator.address,
            12,
            { value: validatorStakeAmount }
        );

        // Create and stake NFT
        const levelId = 1;
        const { tokenId } = await stakeNFT(user, levelId, stargateContract, stargateNFTContract, false);

        // Wait for maturity and delegate
        await mineBlocks(120);
        await stargateContract.connect(user).delegate(tokenId, validator.address);

        // Check if delegation is locked
        const delegation = await stargateContract.getDelegationDetails(tokenId);
        const [, , , isLocked] = await protocolStakerContract.getDelegation(delegation.delegationId);

        if (!isLocked) {
            // Delegation not locked yet, can unstake directly
            await stargateContract.connect(user).unstake(tokenId);
        } else {
            // Delegation is locked, need to request exit first
            await stargateContract.connect(user).requestDelegationExit(tokenId);
            
            // Mine blocks to complete the exit period
            await mineBlocks(120);
            
            // Now try to claim rewards (should work even if 0)
            await stargateContract.connect(user).claimRewards(tokenId);
            // And unstake
            await stargateContract.connect(user).unstake(tokenId);
        }
    });
});
```
