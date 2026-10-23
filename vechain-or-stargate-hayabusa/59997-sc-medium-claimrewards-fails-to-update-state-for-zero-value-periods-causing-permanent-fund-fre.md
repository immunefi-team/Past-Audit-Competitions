# 59997 sc medium claimrewards fails to update state for zero value periods causing permanent fund freeze in unstake&#x20;

* **Submitted on Nov 17th 2025 at 13:07:33 UTC by @hunraj for Audit Comp | Vechain | Stargate Hayabusa**
* **Report ID:** #59997
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

A logic flaw exists in the `_claimRewards` function. When a user has a backlog of claimable periods with a total reward value of zero, the function returns early without updating the user's `lastClaimedPeriod` state. This creates a "trap" state: the user cannot call `unstake` because `unstake` checks `_exceedsMaxClaimablePeriods` and relies on `claimRewards` to advance the checkpoint. Since `claimRewards` returns early, the backlog never shrinks and the user's principal becomes permanently frozen in the contract.

### Vulnerability Details

The protocol uses two safety mechanisms that interact incorrectly:

1. `unstake` Gas Safety: `unstake` reverts if the user has more claimable periods than `maxClaimablePeriods` (e.g., 832) to avoid out-of-gas errors during automatic reward claims. This forces manual clearing of the backlog.
2. `claimRewards` Batching: `claimRewards` is intended to clear the backlog in batches, advancing `lastClaimedPeriod` per batch.

Root cause: `_claimRewards` returns early if the batch's `claimableAmount` is zero, and therefore never updates `lastClaimedPeriod`.

Code excerpt showing the root cause:

```solidity
// In Stargate.sol
function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {
    // ... (logic to determine the first and last period of the current batch) ...
    
    uint256 claimableAmount = _claimableRewards($, _tokenId, 0);

    // [!] VULNERABILITY ROOT CAUSE [!]
    if (claimableAmount == 0) {
        return; // The function exits prematurely.
    }

    // This state update is the entire purpose of the function for clearing the backlog.
    // It is NEVER REACHED if rewards are zero.
    $.lastClaimedPeriod[_tokenId] = lastClaimedPeriod; 

    // ... (transfer logic) ...
}
```

### The Catch-22 Execution Path

{% stepper %}
{% step %}

### Step: Pre-condition

A user (Alice) has an `EXITED` delegation with > maxClaimablePeriods (e.g., 832) claimable periods, all with value 0 VTHO (e.g., delegating to an offline validator).
{% endstep %}

{% step %}

### Step: Alice calls `unstake()`

`unstake()` reverts with `MaxClaimablePeriodsExceeded`. This is the intended gas safety mechanism.
{% endstep %}

{% step %}

### Step: Alice calls `claimRewards()`

`_claimRewards` computes `claimableAmount` for the first batch (e.g., first 832 periods). Because rewards are zero, `claimableAmount == 0` and the function executes `return`. `lastClaimedPeriod` is not updated; no backlog progress is made.
{% endstep %}

{% step %}

### Step: Alice calls `unstake()` again

`unstake()` reverts with `MaxClaimablePeriodsExceeded` for the same reason. Alice cannot progress and is permanently locked out from withdrawing her principal.
{% endstep %}
{% endstepper %}

### Impact

This is a Critical outcome: permanent freezing of user funds. A user can be permanently blocked from unstaking if they accumulate many zero-value claimable periods.

## Recommended Remediation

Always update `lastClaimedPeriod` when processing a batch, even when the batch's reward is zero. The function's responsibility is both to pay rewards and to advance the checkpoint for processed periods. Move the state update before returning for zero-value batches.

Proposed fix (move the state update before the zero-amount return):

```solidity
function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {
    (uint32 firstClaimablePeriod, uint32 lastClaimedPeriod) = _claimableDelegationPeriods($, _tokenId);
    
    // Check if there are any periods to process at all.
    if (firstClaimablePeriod > lastClaimedPeriod) {
        return;
    }

    // Determine the end of the current batch.
    uint32 batchEndPeriod = firstClaimablePeriod + $.maxClaimablePeriods - 1;
    if (batchEndPeriod > lastClaimedPeriod) {
        batchEndPeriod = lastClaimedPeriod;
    }

    uint256 claimableAmount = 0;
    // Calculate rewards ONLY if there's a chance to have them.
    if (firstClaimablePeriod <= batchEndPeriod) {
        // This loop would calculate the rewards for the batch [firstClaimablePeriod...batchEndPeriod]
        // For this PoC, we can represent it as the call to _claimableRewards
        claimableAmount = _claimableRewards($, _tokenId, 0);
    }
    
    // [!] FIX: Always update the state, regardless of the amount.
    // The state update now happens BEFORE the early return.
    $.lastClaimedPeriod[_tokenId] = batchEndPeriod;

    if (claimableAmount == 0) {
        // We've updated the period, so we can now safely return.
        return;
    }

    // Proceed with the transfer logic only if there's an amount to send.
    address tokenOwner = $.stargateNFTContract.ownerOf(_tokenId);
    VTHO_TOKEN.safeTransfer(tokenOwner, claimableAmount);

    emit DelegationRewardsClaimed(
        tokenOwner,
        _tokenId,
        $.delegationIdByTokenId[_tokenId],
        claimableAmount,
        firstClaimablePeriod,
        batchEndPeriod
    );
}
```

This ensures progress is always made on clearing claimable periods and prevents permanent lockouts.

## Proof of Concept

A runnable PoC test was provided. The ProtocolStakerMock was slightly modified to allow setting delegator rewards to zero via a helper function so the test can simulate the "unlucky delegator" scenario.

Modification to ProtocolStakerMock.sol (added state variable + helper setter):

```solidity
// In ProtocolStakerMock.sol
contract ProtocolStakerMock is IProtocolStaker {
    // Added state variable, initialized to the original hardcoded value.
    uint256 private mockDelegatorRewards = 0.1 ether;

    // Added helper function to allow the test to change the reward value.
    function helper__setDelegatorRewards(uint256 _rewards) external {
        mockDelegatorRewards = _rewards;
    }

    function getDelegatorsRewards(
        address _validator,
        uint32 _stakingPeriod
    ) external view override returns (uint256 rewards) {
        if (_stakingPeriod > validations[_validator].completedPeriods + 1) {
            return 0;
        }
        // Used the state variable instead of the hardcoded value.
        return mockDelegatorRewards;
    }
    // ... rest of the contract is unchanged
}
```

Runnable Hardhat Test (excerpt):

```javascript
import { expect } from "chai";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import {
    Stargate,
    StargateNFTMock,
    ProtocolStakerMock,
    StargateNFTMock__factory,
    ProtocolStakerMock__factory,
    MyERC20__factory,
} from "../../typechain-types";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { getOrDeployContracts } from "../helpers";

describe("PoC: Permanent Fund Freeze in unstake", () => {
    let deployer: HardhatEthersSigner;
    let alice: HardhatEthersSigner;
    let validatorX: HardhatEthersSigner;
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    const LEVEL_ID = 1;
    const MAX_CLAIMABLE_PERIODS = 10;
    const VET_STAKED = ethers.parseEther("100");
    let aliceTokenId: bigint;

    beforeEach(async () => {
        // --- 1. SETUP THE FULL TEST ENVIRONMENT ---
        const config = createLocalConfig();
        [deployer, alice, validatorX] = await ethers.getSigners();
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();
        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();
        const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(deployer.address, deployer.address);
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(await tokenContract.getAddress());
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        config.MAX_CLAIMABLE_PERIODS = MAX_CLAIMABLE_PERIODS;
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        await protocolStakerMock.helper__setStargate(await stargateContract.getAddress());

        // --- 2. SIMULATE ALICE'S JOURNEY ---
        await stargateNFTMock.helper__setLevel({ id: LEVEL_ID, name: "Test Level", isX: false, maturityBlocks: 0, scaledRewardFactor: 100, vetAmountRequiredToStake: VET_STAKED });
        await protocolStakerMock.addValidation(validatorX.address, 120);
        await protocolStakerMock.helper__setValidatorStatus(validatorX.address, 2);
        await stargateContract.connect(alice).stake(LEVEL_ID, { value: VET_STAKED });
        aliceTokenId = await stargateNFTMock.getCurrentTokenId();
        await stargateNFTMock.helper__setToken({ tokenId: aliceTokenId, levelId: LEVEL_ID, mintedAtBlock: 1, vetAmountStaked: VET_STAKED, lastVetGeneratedVthoClaimTimestamp_deprecated: 0 });
        await stargateContract.connect(alice).delegate(aliceTokenId, validatorX.address);

        // --- 3. CREATE THE "UNLUCKY DELEGATOR" SCENARIO ---
        const INACTIVE_PERIODS = 25;
        const lastCompletedPeriod = INACTIVE_PERIODS + 1;
        await protocolStakerMock.helper__setValidationCompletedPeriods(validatorX.address, lastCompletedPeriod);
        await protocolStakerMock.helper__setDelegatorRewards(0); // CRITICAL STEP: SIMULATE ZERO REWARDS
        await stargateContract.connect(alice).requestDelegationExit(aliceTokenId);
        await protocolStakerMock.helper__setValidationCompletedPeriods(validatorX.address, lastCompletedPeriod + 1);
        const status = await stargateContract.getDelegationStatus(aliceTokenId);
        expect(status).to.equal(3); // 3 = EXITED
    });

    it("should TRAP Alice, preventing her from unstaking her funds", async () => {
        // --- VERIFY THE TRAP CONDITIONS ---
        const [first, last] = await stargateContract.claimableDelegationPeriods(aliceTokenId);
        const numPeriods = last - first + BigInt(1);
        console.log(`       - Alice has ${numPeriods} claimable periods.`);
        console.log(`       - Max claimable periods is ${MAX_CLAIMABLE_PERIODS}.`);
        expect(numPeriods).to.be.greaterThan(MAX_CLAIMABLE_PERIODS);
        const batchRewards = await stargateContract["claimableRewards(uint256,uint32)"](aliceTokenId, 0);
        expect(batchRewards).to.equal(0);
        console.log(`       - Alice's rewards for the first batch: ${batchRewards} VTHO.`);

        // --- SPRING THE TRAP ---
        console.log("\n       ATTEMPT 1: Alice tries to unstake...");
        await expect(stargateContract.connect(alice).unstake(aliceTokenId)).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");
        console.log("       ✅ Unstake correctly reverted with MaxClaimablePeriodsExceeded.");

        // --- PROVE THE FREEZE IS PERMANENT ---
        console.log("\n       ATTEMPT 2: Alice tries to claim rewards to clear the periods...");
        const claimTx = await stargateContract.connect(alice).claimRewards(aliceTokenId);
        await claimTx.wait();
        console.log("       ✅ claimRewards() executed, but the underlying state did not change.");
        const [firstAfterClaim, lastAfterClaim] = await stargateContract.claimableDelegationPeriods(aliceTokenId);
        const numPeriodsAfterClaim = lastAfterClaim - firstAfterClaim + BigInt(1);
        expect(numPeriodsAfterClaim).to.equal(numPeriods);
        console.log(`       - Alice STILL has ${numPeriodsAfterClaim} claimable periods.`);

        console.log("\n       ATTEMPT 3: Alice tries to unstake one last time...");
        await expect(stargateContract.connect(alice).unstake(aliceTokenId)).to.be.revertedWithCustomError(stargateContract, "MaxClaimablePeriodsExceeded");
        console.log("       ✅ Unstake reverted AGAIN. Alice's funds are permanently frozen.");
        console.log("\n       --- VULNERABILITY CONFIRMED ---");
    });
});
```

Test output from the PoC run:

```
PoC: Permanent Fund Freeze in unstake
       - Alice has 26 claimable periods.
       - Max claimable periods is 10.
       - Alice's rewards for the first batch: 0 VTHO.

       ATTEMPT 1: Alice tries to unstake...
       ✅ Unstake correctly reverted with MaxClaimablePeriodsExceeded.

       ATTEMPT 2: Alice tries to claim rewards to clear the periods...
       ✅ claimRewards() executed, but the underlying state did not change.
       - Alice STILL has 26 claimable periods.

       ATTEMPT 3: Alice tries to unstake one last time...
       ✅ Unstake reverted AGAIN. Alice's funds are permanently frozen.

       --- VULNERABILITY CONFIRMED ---
    ✔ should TRAP Alice, preventing her from unstaking her funds
```

***

If you want, I can:

* Produce a patch/PR diff applying the proposed fix in the repository structure referenced.
* Run through alternative mitigations (e.g., special-case `unstake` to allow manual exception flows).
