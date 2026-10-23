# 60173 sc high the phantom claimable periods can permanently lock the staked vet for ended delegations

**Submitted on Nov 19th 2025 at 15:01:52 UTC by @XDZIBECX for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60173
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

In this contract there is a mismatch between how Stargate decides which periods are claimable and how it tracks the last period that actually paid out. After a delegation has cleanly ended and the user has already claimed all legitimate rewards up to the end period, the contract can still report a growing “claimable” window of zero-reward periods. This happens because:

* `_claimableDelegationPeriods()` keeps returning this phantom window.
* `_claimRewards()` is a no-op when the window contains zero rewards and never advances `lastClaimedPeriod`.
* `_exceedsMaxClaimablePeriods()` eventually becomes permanently true for that NFT.
* As a result, `unstake()` and `delegate()` revert with `MaxClaimablePeriodsExceeded`, permanently freezing the staked VET of that NFT, even though the delegation ended and all real rewards were claimed.

A test demonstrating this issue is included below.

***

## Vulnerability Details

Root cause summary:

* Once a delegation has ended and the user has claimed all rewards up to `endPeriod`, `lastClaimedPeriod` becomes `endPeriod`.
* `nextClaimablePeriod = lastClaimedPeriod + 1 = endPeriod + 1`.
* `_claimableDelegationPeriods()` uses the condition `endPeriod > nextClaimablePeriod` to detect ended delegations. After the user has claimed up to `endPeriod`, that condition becomes false, so the function falls through to the "active" branch and returns a fake claimable window from `endPeriod + 1` up to `completedPeriods`, even though no rewards exist for those periods.
* `_claimRewards()` computes `claimableAmount` for that window. Because there are no rewards after `endPeriod`, `claimableAmount == 0` and the function returns early without updating `lastClaimedPeriod`.
* As `completedPeriods` grows (validator keeps producing blocks), the phantom window `[firstClaimablePeriod, lastClaimablePeriod] = [endPeriod + 1, completedPeriods]` grows until `lastClaimablePeriod - firstClaimablePeriod >= maxClaimablePeriods`.
* `_exceedsMaxClaimablePeriods()` then returns true permanently for that NFT.
* Both `unstake()` and `delegate()` check `_exceedsMaxClaimablePeriods()` and revert with `MaxClaimablePeriodsExceeded` before calling `_claimRewards()`. Because `claimRewards()` cannot advance `lastClaimedPeriod` (no rewards in the phantom window), the state can never be corrected on-chain by the user.

Relevant code excerpts (links preserved):

* The ended-delegation branch condition in `_claimableDelegationPeriods()` (bug origin): <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L911C1-L922C10>

```solidity
        // check first for delegations that ended
        // endPeriod is not max if the delegation is exited or requested to exit
        // if the endPeriod is before the current validator period, it means the delegation ended
        // because if its equal it means they requested to exit but the current period is not over yet
        if (
            endPeriod != type(uint32).max &&
            endPeriod < currentValidatorPeriod &&
            endPeriod > nextClaimablePeriod
        ) {
            return (nextClaimablePeriod, endPeriod);
        }
```

* The active branch that can return a fake window: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L924C8-L930C10>

```solidity
        // check that the start period is before the current validator period
        // and if it is, return the start period and the current validator period.
        // we use "less than" because if we use "less than or equal", even
        // if the delegation started, the current period rewards are not claimable
        if (nextClaimablePeriod < currentValidatorPeriod) {
            return (nextClaimablePeriod, completedPeriods);
        }
```

* `_exceedsMaxClaimablePeriods()` which becomes permanently true: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L956C3-L973C6>

```solidity
    (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) =
        _claimableDelegationPeriods($, _tokenId);
    if (firstClaimablePeriod > lastClaimablePeriod) return false;

    if (lastClaimablePeriod - firstClaimablePeriod >= $.maxClaimablePeriods) {
        return true;
    }
    return false;
}
```

* `unstake()` gating on `_exceedsMaxClaimablePeriods()`: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L296C4-L304C1>

```solidity
        // If the NFT has reached the max number of claimable periods, we revert to avoid any loss of rewards
        // In this case, the owner should separatly call the claimRewards() function multiple times to claim all rewards,
        // then call the unstake() function again.
        if (_exceedsMaxClaimablePeriods($, _tokenId)) {
            revert MaxClaimablePeriodsExceeded();
        }
        // ensure that the rewards are claimed
        _claimRewards($, _tokenId);
```

* `delegate()` also gating similarly: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L429C6-L439C10>

```solidity
        // if the token was previously delegated, check if the rewards for the previous delegation have been claimed
        if (currentDelegationId != 0) {
            if (_exceedsMaxClaimablePeriods($, _tokenId)) {
                // If the NFT has reached the max number of claimable periods, we revert to avoid any loss of rewards
                // In this case, the owner should separatly call the claimRewards() function multiple times to claim all rewards,
                // then call the delegate() function again.
                revert MaxClaimablePeriodsExceeded();
            }
            // claim pending rewards
            _claimRewards($, _tokenId);
        }
```

* `_claimRewards()` returning early when no rewards are available (and thus not updating `lastClaimedPeriod`): <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L736C3-L776C6>

```solidity
    uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
    if (claimableAmount == 0) {
        return;  lastClaimedPeriod is NOT updated here
    }

    address tokenOwner = $.stargateNFTContract.ownerOf(_tokenId);
    $.lastClaimedPeriod[_tokenId] = lastClaimablePeriod;
    VTHO_TOKEN.safeTransfer(tokenOwner, claimableAmount);
}
```

Because `lastClaimedPeriod` never advances beyond `endPeriod`, the phantom window never closes. `_exceedsMaxClaimablePeriods()` remains true and the NFT can never successfully call `unstake()` or `delegate()` — freezing the staked VET.

***

## Impact Details

* Severity: Critical/High due to the potential for permanent freeze of staked VET behind an NFT.
* Once the phantom claimable window after `endPeriod` grows beyond `maxClaimablePeriods`, `_exceedsMaxClaimablePeriods()` becomes permanently `true`.
* From that point, every call to `unstake()` or `delegate()` for that NFT reverts with `MaxClaimablePeriodsExceeded`, even though the delegation ended and no rewards accrue for it.
* The user cannot fix this on-chain: `claimRewards()` sees a zero-reward window and does not advance `lastClaimedPeriod`, so the fake window grows as `completedPeriods` increases and the revert condition never clears.
* Recovery requires an out-of-band admin/upgrade action via `UPGRADER_ROLE`.

***

## Proof of Concept

<details>

<summary>Test demonstrating the permanent freeze (click to expand)</summary>

```ts
import { expect } from "chai";
import {
  MyERC20,
  MyERC20__factory,
  ProtocolStakerMock,
  ProtocolStakerMock__factory,
  Stargate,
  StargateNFTMock,
  StargateNFTMock__factory,
} from "../../../typechain-types";
import { getOrDeployContracts } from "../../helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

describe("shard-u5: Stargate: Permanent freezing of staked VET after ended delegation", () => {
  const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

  const LEVEL_ID = 1;
  const START_PERIOD = 10;
  const END_PERIOD = 20;
  const MAX_CLAIMABLE_PERIODS = 8;
  const VALIDATOR_STATUS_ACTIVE = 2;
  const DELEGATION_STATUS_EXITED = 3;

  let stargate: Stargate;
  let stargateNFT: StargateNFTMock;
  let protocolStaker: ProtocolStakerMock;
  let vtho: MyERC20;

  let deployer: HardhatEthersSigner;
  let user: HardhatEthersSigner;
  let validator: HardhatEthersSigner;

  it("Ended delegation can permanently freeze staked VET for NFT", async () => {

    //Deploy mocks + Stargate via existing helper
    process.env.VITE_APP_ENV = "local";
    const config = createLocalConfig();

    [deployer] = await ethers.getSigners();

    // Deploy ProtocolStaker mock
    const protocolStakerFactory = new ProtocolStakerMock__factory(deployer);
    protocolStaker = await protocolStakerFactory.deploy();
    await protocolStaker.waitForDeployment();

    // Deploy StargateNFT mock
    const stargateNFTFactory = new StargateNFTMock__factory(deployer);
    stargateNFT = await stargateNFTFactory.deploy();
    await stargateNFT.waitForDeployment();

    // Fake VTHO token at the built-in address
    const vthoImpl = await new MyERC20__factory(deployer).deploy(
      deployer.address,
      deployer.address
    );
    await vthoImpl.waitForDeployment();
    const vthoCode = await ethers.provider.getCode(vthoImpl);
    await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, vthoCode]);
    vtho = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

    // Plug mocks into config so getOrDeployContracts wires Stargate correctly
    config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStaker.getAddress();
    config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFT.getAddress();
    config.MAX_CLAIMABLE_PERIODS = MAX_CLAIMABLE_PERIODS;

    const { stargateContract, otherAccounts } = await getOrDeployContracts({
      forceDeploy: true,
      config,
    });

    stargate = stargateContract;
    user = otherAccounts[0];
    validator = otherAccounts[2];

    // Wire mock staker to Stargate
    await (await protocolStaker.helper__setStargate(stargate.target)).wait();

    // Setup validator & level, mint VTHO to Stargate
    // Add validator and set ACTIVE
    await (await protocolStaker.addValidation(validator.address, 120)).wait();
    await (
      await protocolStaker.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_ACTIVE
      )
    ).wait();

    // Start validator just before START_PERIOD (completed = START - 2 => current = START - 1)
    await (
      await protocolStaker.helper__setValidationCompletedPeriods(
        validator.address,
        START_PERIOD - 2
      )
    ).wait();

    // Configure level
    const vetAmountRequired = ethers.parseEther("1");
    await (
      await stargateNFT.helper__setLevel({
        id: LEVEL_ID,
        name: "Level-1",
        isX: false,
        maturityBlocks: 0, // no maturity to block unstake/delegate
        scaledRewardFactor: 100,
        vetAmountRequiredToStake: vetAmountRequired,
      })
    ).wait();

    // Mint some VTHO into Stargate for rewards
    await (await vtho.mint(stargate.target, ethers.parseEther("1000000"))).wait();
    // User stakes & delegates once
    await stargate
      .connect(user)
      .stake(LEVEL_ID, { value: vetAmountRequired })
      .then((tx) => tx.wait());

    const tokenId = await stargateNFT.getCurrentTokenId();

    // Set token data in mock (required for getToken() to work)
    await (
      await stargateNFT.helper__setToken({
        tokenId: tokenId,
        levelId: LEVEL_ID,
        mintedAtBlock: 0,
        vetAmountStaked: vetAmountRequired,
        lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
      })
    ).wait();

    await stargate
      .connect(user)
      .delegate(tokenId, validator.address)
      .then((tx) => tx.wait());

    const delegationId = await stargate.getDelegationIdOfToken(tokenId);

    // Set delegation start / end in mock
    await (
      await protocolStaker.helper__setDelegationStartPeriod(delegationId, START_PERIOD)
    ).wait();
    await (
      await protocolStaker.helper__setDelegationEndPeriod(delegationId, END_PERIOD)
    ).wait();

    //  End delegation & claim all real rewards up to END_PERIOD
    // Move validator so current period is END + 1 (delegation ended)
    await (
      await protocolStaker.helper__setValidationCompletedPeriods(
        validator.address,
        END_PERIOD
      )
    ).wait();

    const status = await stargate.getDelegationStatus(tokenId);
    expect(status).to.equal(DELEGATION_STATUS_EXITED);

    // Repeatedly claim until we've claimed up to END_PERIOD.
    // We want lastClaimedPeriod >= END_PERIOD so nextClaimablePeriod = END + 1.
    // This is critical for the bug to manifest.
    for (let i = 0; i < 50; i++) {
      const [first, last] = await stargate.claimableDelegationPeriods(tokenId);
      // No more claimable periods
      if (first === 0n && last === 0n) break;

      // Claim rewards
      await stargate.connect(user).claimRewards(tokenId).then((tx) => tx.wait());

      // Check if we've claimed up to or past END_PERIOD
      // After claiming, lastClaimedPeriod is updated to the last period we claimed
      if (last >= BigInt(END_PERIOD)) {
        // Verify we've actually reached END_PERIOD by checking claimable periods again
        const [nextFirst, nextLast] = await stargate.claimableDelegationPeriods(tokenId);
        // If nextFirst > END_PERIOD, we've successfully claimed up to END_PERIOD
        if (nextFirst > BigInt(END_PERIOD)) {
          break;
        }
      }
    }
    // Push validator periods far beyond END to trigger phantom window
    const bigCompletedPeriods = END_PERIOD + MAX_CLAIMABLE_PERIODS + 50;
    await (
      await protocolStaker.helper__setValidationCompletedPeriods(
        validator.address,
        bigCompletedPeriods
      )
    ).wait();

    // Now check claimable periods - if we haven't fully claimed, claim one more time
    let [firstClaimable, lastClaimable] =
      await stargate.claimableDelegationPeriods(tokenId);

    // If firstClaimable is still <= END_PERIOD, we need to claim one more time
    // to push lastClaimedPeriod to END_PERIOD (so nextClaimablePeriod = END_PERIOD + 1)
    if (firstClaimable <= BigInt(END_PERIOD) && lastClaimable === BigInt(END_PERIOD)) {
      // Claim one more time to ensure we've claimed up to END_PERIOD
      await stargate.connect(user).claimRewards(tokenId).then((tx) => tx.wait());
      [firstClaimable, lastClaimable] = await stargate.claimableDelegationPeriods(tokenId);
    }

    // claimable period range has shifted completely after END_PERIOD
    // After claiming up to END_PERIOD, nextClaimablePeriod = END_PERIOD + 1
    // Because of the bug, we get (firstClaimable, lastClaimable) = (END+1, completedPeriods)
    // The condition "endPeriod > nextClaimablePeriod" is false (20 > 21 = false)
    // So it falls through to return (nextClaimablePeriod, completedPeriods)
    expect(firstClaimable).to.be.greaterThan(BigInt(END_PERIOD));
    expect(lastClaimable).to.equal(BigInt(bigCompletedPeriods));

    const gap = lastClaimable - firstClaimable + 1n;
    const maxClaimable = await stargate.getMaxClaimablePeriods();
    expect(gap).to.be.greaterThan(BigInt(maxClaimable));

    // This means _exceedsMaxClaimablePeriods() == true for this NFT.

    // Show that unstake() and delegate() are now blocked

    await expect(stargate.connect(user).unstake(tokenId))
      .to.be.revertedWithCustomError(stargate, "MaxClaimablePeriodsExceeded");

    await expect(
      stargate.connect(user).delegate(tokenId, validator.address)
    ).to.be.revertedWithCustomError(stargate, "MaxClaimablePeriodsExceeded");

    // Show claimRewards() cannot clear this state (permanent freeze)
   
    // Optional: observe current claimable rewards (may be 0 or >0 depending on mock)
    const claimableBefore = await stargate.claimableRewards(tokenId);

    // User tries to "fix" by claiming again
    const userVthoBefore = await vtho.balanceOf(user.address);
    await stargate.connect(user).claimRewards(tokenId).then((tx) => tx.wait());
    const userVthoAfter = await vtho.balanceOf(user.address);

    // Even after an extra claim:
    // - claimable periods remain after END_PERIOD
    // - lastClaimedPeriod is not advanced past the phantom range
    const [firstAfterClaim, lastAfterClaim] =
      await stargate.claimableDelegationPeriods(tokenId);

    expect(firstAfterClaim).to.be.greaterThan(BigInt(END_PERIOD));
    expect(lastAfterClaim).to.equal(BigInt(bigCompletedPeriods));

    // And unstake is STILL blocked → user has no on-chain way to recover their VET
    await expect(stargate.connect(user).unstake(tokenId))
      .to.be.revertedWithCustomError(stargate, "MaxClaimablePeriodsExceeded");

    // (Delegate remains blocked as well)
    await expect(
      stargate.connect(user).delegate(tokenId, validator.address)
    ).to.be.revertedWithCustomError(stargate, "MaxClaimablePeriodsExceeded");

  });
});
```

The test output:

```
hard-u5: Stargate: Permanent freezing of staked VET after ended delegation
    ✔ Ended delegation can permanently freeze staked VET for NFT (786ms)


  1 passing (789ms)

Done in 3.95s.
```

</details>

***

## References

All relevant code links are included inline in the Vulnerability Details section (kept as originally referenced).

***

If you want, I can:

* Suggest specific code-level fixes (patches) to correctly handle ended delegations and advance `lastClaimedPeriod` even when claimableAmount==0, or change the ended-delegation condition to treat `endPeriod >= nextClaimablePeriod` as ended, etc.
* Provide a minimal unit test that asserts the fixed behavior.
* Draft a changelog entry / mitigation guidance for operators (e.g., recommend an immediate upgrade or admin intervention).
