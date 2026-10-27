# 60049 sc high double effective stake decrement locks delegators unstake reverts due to duplicate effectivestake decrements in exit flow

**Submitted on Nov 17th 2025 at 21:02:32 UTC by @Johnyfwesh for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60049
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

When an ACTIVE delegation requests exit and later the owner calls `unstake()`, the contract can decrement the same delegation's effective stake twice: once in `requestDelegationExit()` and again in `unstake()` via `_updatePeriodEffectiveStake(..., false)`. Both code paths recompute the NFT's `effectiveStake` and perform the subtraction without tracking whether the stake was already removed. This can cause a checked-arithmetic underflow (panic `0x11`) or corrupt checkpoint accounting.

## Finding Description and Impact

* `requestDelegationExit()` subtracts the NFT’s effective stake from the validator checkpoints when an ACTIVE delegation starts exiting.
* `unstake()` repeats the same `_updatePeriodEffectiveStake(..., false)` call when the validator turns `VALIDATOR_STATUS_EXITED` or the delegation is `PENDING`.
* Both entry points compute `effectiveStake` and subtract it using Solidity 0.8 checked arithmetic. There is no per-delegation flag indicating the stake was already removed. A normal lifecycle delegates → requestDelegationExit → unstake thus triggers two decrements against the same checkpoint.

Consequences:

* If the delegator is the only (or primary) contributor for that validator, the second subtraction can underflow and revert `unstake()`, permanently preventing withdrawal and locking funds/rewards.
* If other delegators mask the underflow, the checkpoint becomes under-reported, corrupting reward splits and validator accounting.
* Attackers can grief validators by forcing exit flows to be requested and causing DoS or requiring manual intervention.

### Affected code (examples)

* contracts/Stargate.sol Lines 231-250 (excerpt from `unstake`)

```solidity
function unstake(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    Delegation memory delegation = _getDelegationDetails($, _tokenId);
    DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);

    // if the token is under the maturity period, we cannot unstake it
    if ($.stargateNFTContract.isUnderMaturityPeriod(_tokenId)) {
        revert TokenUnderMaturityPeriod(_tokenId);
    }

    // check the delegation status
    // if the delegation is active, then NFT cannot be unstaked, since the VET is locked in the protocol
    if (delegation.status == DelegationStatus.ACTIVE) {
        revert InvalidDelegationStatus(_tokenId, DelegationStatus.ACTIVE);
    } else if (delegation.status != DelegationStatus.NONE) {
        // if the delegation is pending or exited
        // withdraw the VET from the protocol so we can transfer it back to the caller (which is also the owner of the NFT)
        $.protocolStakerContract.withdrawDelegation(delegation.delegationId);
```

* contracts/Stargate.sol Lines 523-542 (excerpt from `requestDelegationExit`)

```solidity
function requestDelegationExit(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) {
        revert DelegationNotFound(_tokenId);
    }

    Delegation memory delegation = _getDelegationDetails($, _tokenId);

    if (delegation.status == DelegationStatus.PENDING) {
        // if the delegation is pending, we can exit it immediately
        // by withdrawing the VET from the protocol
        $.protocolStakerContract.withdrawDelegation(delegationId);
        emit DelegationWithdrawn(
            _tokenId,
            delegation.validator,
            delegationId,
            delegation.stake,
```

* contracts/Stargate.sol Lines 993-1012 (excerpt from `_updatePeriodEffectiveStake`)

```solidity
function _updatePeriodEffectiveStake(
    StargateStorage storage $,
    address _validator,
    uint256 _tokenId,
    uint32 _period,
    bool _isIncrease
) private {
    // calculate the effective stake
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);

    // get the current effective stake
    uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);

    // calculate the updated effective stake
    uint256 updatedValue = _isIncrease
        ? currentValue + effectiveStake
        : currentValue - effectiveStake;

    // push the updated effective stake
    $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
```

* `_updatePeriodEffectiveStake()` subtracts without verifying the stake wasn't already removed and without ensuring `currentValue >= effectiveStake`. A second subtraction can underflow or corrupt the checkpoint.

## Impact

* Funds frozen: Delegators who followed documented exit flow may be unable to call `unstake()` after `requestDelegationExit()` once the validator exits, due to underflow revert.
* Accounting corruption: Even if underflow is avoided, additional subtraction skews `delegatorsEffectiveStake`, misallocating future rewards.
* Validator DoS: Attackers can grief validators by forcing exits and blocking withdrawals.

## Recommended mitigation steps

{% stepper %}
{% step %}

### Track per-delegation stake removal

Track whether a delegation’s effective stake has already been removed (e.g., a boolean in storage). Skip `_updatePeriodEffectiveStake(..., false)` in `unstake()` if the exit path already removed the stake.
{% endstep %}

{% step %}

### Centralize checkpoint decrease in requestDelegationExit()

Move the checkpoint decrease exclusively into `requestDelegationExit()` for the ACTIVE path and ensure `unstake()` only handles pending delegations that never triggered an exit signal.
{% endstep %}

{% step %}

### Add sanity checks

Add sanity checks such as:

* if (!\_isIncrease && currentValue < effectiveStake) revert so similar regressions fail predictably during testing instead of bricking user funds.
  {% endstep %}
  {% endstepper %}

## Proof of Concept

<details>

<summary>PoC test: DoubleEffectiveStakeDecrement.test.ts (click to expand)</summary>

PoC demonstrates the issue by:

1. Staking an NFT and delegating so the validator checkpoint records the effective stake.
2. Calling `requestDelegationExit()` while the delegation is ACTIVE, which triggers the first decrement.
3. Transitioning the validator to `VALIDATOR_STATUS_EXITED` and calling `unstake()`, which triggers the second decrement and causes a panic `0x11` revert.

Test file to add to packages/contracts/test/unit/Stargate/DoubleEffectiveStakeDecrement.test.ts:

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import type {
    ProtocolStakerMock,
    Stargate,
    StargateNFTMock,
    StargateProxy,
} from "../../../typechain-types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

describe("poc: requestDelegationExit double-decrements delegatorsEffectiveStake", () => {
    const LEVEL_ID = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;
    const ACTIVE_COMPLETED_PERIODS = 5;
    const CHECKPOINT_PERIOD = ACTIVE_COMPLETED_PERIODS + 2;
    const STAKE_AMOUNT = ethers.parseEther("1");

    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let protocolStakerMock: ProtocolStakerMock;
    let stargateNFTMock: StargateNFTMock;
    let stargate: Stargate;

    beforeEach(async () => {
        [deployer, user, validator] = await ethers.getSigners();

        const protocolStakerFactory = await ethers.getContractFactory("ProtocolStakerMock");
        protocolStakerMock = (await protocolStakerFactory.deploy()) as ProtocolStakerMock;
        await protocolStakerMock.waitForDeployment();

        const stargateNFTFactory = await ethers.getContractFactory("StargateNFTMock");
        stargateNFTMock = (await stargateNFTFactory.deploy()) as StargateNFTMock;
        await stargateNFTMock.waitForDeployment();

        const clockFactory = await ethers.getContractFactory("Clock");
        const clockLib = await clockFactory.deploy();
        await clockLib.waitForDeployment();

        const stargateFactory = await ethers.getContractFactory("Stargate", {
            libraries: {
                Clock: await clockLib.getAddress(),
            },
        });
        const stargateImplementation = (await stargateFactory.deploy()) as Stargate;
        await stargateImplementation.waitForDeployment();

        const stargateProxyFactory = await ethers.getContractFactory("StargateProxy");
        const stargateProxy = (await stargateProxyFactory.deploy(
            await stargateImplementation.getAddress(),
            "0x"
        )) as StargateProxy;
        await stargateProxy.waitForDeployment();

        stargate = stargateFactory.attach(await stargateProxy.getAddress()) as Stargate;

        await (await stargate.initialize({
            admin: deployer.address,
            protocolStakerContract: await protocolStakerMock.getAddress(),
            stargateNFTContract: await stargateNFTMock.getAddress(),
            maxClaimablePeriods: 1000,
        })).wait();

        await (await protocolStakerMock.helper__setStargate(await stargate.getAddress())).wait();

        await (await protocolStakerMock.addValidation(validator.address, 120)).wait();
        await (
            await protocolStakerMock.helper__setValidatorStatus(
                validator.address,
                VALIDATOR_STATUS_ACTIVE
            )
        ).wait();

        await (
            await stargateNFTMock.helper__setLevel({
                id: LEVEL_ID,
                name: "Strength",
                isX: false,
                maturityBlocks: 0,
                scaledRewardFactor: 100,
                vetAmountRequiredToStake: STAKE_AMOUNT,
            })
        ).wait();
        await (
            await stargateNFTMock.helper__setToken({
                tokenId: 10000,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: STAKE_AMOUNT,
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            })
        ).wait();
        await (await stargateNFTMock.helper__setIsUnderMaturityPeriod(false)).wait();
    });

    it("reverts on unstake because effective stake is subtracted twice", async () => {
        // Phase 1: stake and delegate so the validator checkpoint records our effective stake.
        const stakeTx = await stargate.connect(user).stake(LEVEL_ID, { value: STAKE_AMOUNT });
        await stakeTx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();

        const delegateTx = await stargate.connect(user).delegate(tokenId, validator.address);
        await delegateTx.wait();

        await (
            await protocolStakerMock.helper__setValidationCompletedPeriods(
                validator.address,
                ACTIVE_COMPLETED_PERIODS
            )
        ).wait();
        const effectiveBeforeExit = await stargate.getDelegatorsEffectiveStake(
            validator.address,
            CHECKPOINT_PERIOD
        );
        expect(effectiveBeforeExit).to.equal(STAKE_AMOUNT);

        // Phase 2: requestDelegationExit while the delegation is ACTIVE to trigger the first decrement.
        await (await stargate.connect(user).requestDelegationExit(tokenId)).wait();
        const effectiveAfterExit = await stargate.getDelegatorsEffectiveStake(
            validator.address,
            CHECKPOINT_PERIOD
        );
        expect(effectiveAfterExit).to.equal(0n);

        // Phase 3: validator exits and unstake hits the same decrement path, causing an underflow revert.
        await (
            await protocolStakerMock.helper__setValidatorStatus(
                validator.address,
                VALIDATOR_STATUS_EXITED
            )
        ).wait();
        await expect(stargate.connect(user).unstake(tokenId)).to.be.revertedWithPanic(0x11);
    });
});
```

Run the targeted test:

```bash
npx hardhat test test/unit/Stargate/DoubleEffectiveStakeDecrement.test.ts
```

Expected result:

```
poc: requestDelegationExit double-decrements delegatorsEffectiveStake
  ✔ reverts on unstake because effective stake is subtracted twice
```

</details>
