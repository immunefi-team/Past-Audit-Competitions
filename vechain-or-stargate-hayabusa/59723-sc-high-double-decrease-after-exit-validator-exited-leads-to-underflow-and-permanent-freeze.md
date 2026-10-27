# 59723 sc high double decrease after exit validator exited leads to underflow and permanent freeze

**Submitted on Nov 15th 2025 at 08:50:10 UTC by @yesofcourse for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #59723
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Impacts:
  * Permanent freezing of funds

## Description

### Brief / Intro

When a delegator requests exit while their validator is ACTIVE, the contract schedules a future decrease to the validator’s `delegatorsEffectiveStake` trace.

If, before the user calls `unstake()` or redelegates, the validator later transitions to `EXITED`, `unstake()`/`_delegate()` schedule a second decrease for the same token/validator line.

Because the first decrease already removed this token’s stake at that (or an earlier) checkpoint, the second subtracts from a zero (or too-small) value and triggers a Solidity 0.8 arithmetic underflow (panic 0x11), permanently reverting exit/redelegation.

On mainnet this strands the user’s principal VET.

### Vulnerability details (core snippet)

The protocol mutates per-validator total delegators’ effective stake using `Checkpoints.Trace224`, writing an updated value for a future period:

```solidity
function _updatePeriodEffectiveStake(
    StargateStorage storage $,
    address _validator,
    uint256 _tokenId,
    uint32 _period,
    bool _isIncrease
) private {
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
    uint224 current = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
    uint224 updated = _isIncrease
        ? SafeCast.toUint224(uint256(current) + effectiveStake)
        : SafeCast.toUint224(uint256(current) - effectiveStake); // <- assumes current >= effectiveStake
    $.delegatorsEffectiveStake[_validator].push(_period, updated);
}
```

Two separate call sites schedule decreases:

* User-initiated exit (schedules a decrease one period in the future):

```solidity
// requestDelegationExit()
(,,, uint32 c) = protocol.getValidationPeriodDetails(validator);
_updatePeriodEffectiveStake($, validator, _tokenId, c + 2, false); // first decrease
```

* Validator EXITED (or PENDING) branch in exit/move flows (schedules another decrease):

```solidity
// in unstake() and _delegate()
if (currentValidatorStatus == EXITED || status == PENDING) {
    (,,, uint32 oldCompletedPeriods) = protocol.getValidationPeriodDetails(prevValidator);
    _updatePeriodEffectiveStake($, prevValidator, _tokenId, oldCompletedPeriods + 2, false); // second decrease
}
```

Because the first decrease already “removes” the NFT’s effective stake for that future period, the second call often finds:

```solidity
current = $.delegatorsEffectiveStake[validator].upperLookup(period) == 0;
updated = 0 - effectiveStake; // panic(0x11)
```

This is not an owner/admin/multisig scenario: it’s a natural race between (a) the user having requested exit and (b) the validator later becoming `EXITED` before the user finalizes with `unstake()`/redelegation.

In sole-delegator cases the underflow is guaranteed; in multi-delegator cases it may still underflow (if `current < effectiveStake`) or at minimum double-subtract accounting.

## Reproduced behavior

{% stepper %}
{% step %}

### Steps 1–3: Stake, become ACTIVE, request exit (schedules first decrease)

* Delegate token, advance to ACTIVE.
* Call `requestDelegationExit()` → schedules first decrease at `completedPeriods + 2`.
  {% endstep %}

{% step %}

### Step 4: Validator flips to EXITED before user finalizes

* Validator transitions to `EXITED` (without the user calling `unstake()`).
  {% endstep %}

{% step %}

### Step 5: User calls `unstake()` / redelegates -> revert

* `unstake()` (or redelegation) schedules a second decrease for `oldCompletedPeriods + 2`.
* Both decreases land on the same checkpoint, so the second subtracts from zero and triggers a Solidity underflow (panic 0x11), reverting the transaction and freezing the position.
  {% endstep %}
  {% endstepper %}

## Impact details

{% hint style="danger" %}
Direct user loss (Critical — Permanent freezing of funds):\
For any affected NFT, `unstake()` and redelegation permanently revert on the double-decrease path as long as the validator remains `EXITED`. The user cannot withdraw or move staked VET via the trustless interface.
{% endhint %}

* Blast radius:
  * Deterministic for sole-delegator validators (second decrease subtracts from zero).
  * Probabilistic but likely for multi-delegator validators (if aggregate at the second checkpoint < effectiveStake). Even when it doesn’t underflow, accounting is double-subtracted for an already-exited validator.
* No user-land recovery: there is no user-accessible method to “unschedule” or compensate the duplicate decrease; only an admin upgrade/migration could unbrick affected NFTs.
* Cost to protocol/users: frozen principal VET for each impacted token; potential support load and reputational risk; any downstream logic relying on the trace can be skewed.

## Proof of Concept

Add the following file to `contracts/test/unit/Stargate/DoubleDecreaseFreeze.test.ts` and run with:

npx hardhat test test/unit/Stargate/DoubleDecreaseFreeze.test.ts

from the `packages/contracts` directory.

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
import { TransactionResponse } from "ethers";
import { expect } from "chai";

describe("shard-u1: Stargate: Double-decrease harness", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

    const LEVEL_ID = 1;
    const PERIOD_SIZE = 120;

    const VALIDATOR_STATUS_QUEUED = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_NONE = 0;
    const DELEGATION_STATUS_PENDING = 1;
    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    let stargateContract: Stargate;
    let stargateNFTMockContract: StargateNFTMock;
    let protocolStakerMockContract: ProtocolStakerMock;
    let vthoTokenContract: MyERC20;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let otherAccounts: HardhatEthersSigner[];
    let validator: HardhatEthersSigner;
    let tx: TransactionResponse;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer, ...otherAccounts] = await ethers.getSigners();

        // Deploy StargateNFT mock
        const stargateNFTMockContractFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMockContract = await stargateNFTMockContractFactory.deploy();
        await stargateNFTMockContract.waitForDeployment();

        // Deploy ProtocolStaker mock
        const protocolStakerMockContractFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMockContract = await protocolStakerMockContractFactory.deploy();
        await protocolStakerMockContract.waitForDeployment();

        // Deploy VTHO token implementation and map it to the special energy address
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();

        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        // Wire mocks into local config
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMockContract.getAddress();
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMockContract.getAddress();

        // Deploy Stargate + rest of the stack using the helper
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });

        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);

        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[1];

        // Add validator to ProtocolStakerMock
        tx = await protocolStakerMockContract.addValidation(validator.address, PERIOD_SIZE);
        await tx.wait();

        // Let the mock know who Stargate is so it can send funds back on withdrawDelegation
        tx = await protocolStakerMockContract.helper__setStargate(stargateContract.target);

        // Start validator as ACTIVE
        tx = await protocolStakerMockContract.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // Configure a basic level and a pre-minted token spec on the NFT mock
        tx = await stargateNFTMockContract.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        tx = await stargateNFTMockContract.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();

        // Fund Stargate with VTHO so reward logic (if any) never reverts
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract, ethers.parseEther("50000000"));
        await tx.wait();
    });

    it("harness sanity: can stake and delegate with mocks wired in", async () => {
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);

        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();

        const tokenId = await stargateNFTMockContract.getCurrentTokenId();

        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        const status = await stargateContract.getDelegationStatus(tokenId);
        expect(status).to.equal(DELEGATION_STATUS_PENDING);
    });

    it("step 1–3: stake → ACTIVE delegation → request exit succeeds", async () => {
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);

        // 1) Stake NFT
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();

        const tokenId = await stargateNFTMockContract.getCurrentTokenId();

        // 2) Delegate to validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        // Make validator advanced enough so delegation is considered ACTIVE
        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
            validator.address,
            10
        );
        await tx.wait();

        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );

        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);

        // 3) User requests delegation exit while validator is ACTIVE
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        // Check that the mock recorded the correct endPeriod = completedPeriods + 1 = 11
        const { startPeriod, endPeriod } =
            await protocolStakerMockContract.getDelegationPeriodDetails(delegationId);

        expect(startPeriod).to.equal(2n); // 0 completed periods at addDelegation time + 2
        expect(endPeriod).to.equal(11n);  // 10 completed periods at exit time + 1
    });

    it("step 4–6 (PoC): user exit already scheduled + validator EXITED -> unstake reverts (double-decrease on same period)", async () => {
    const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);

    // 1) Stake NFT
    let txLocal = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await txLocal.wait();

    const tokenId = await stargateNFTMockContract.getCurrentTokenId();

    // 2) Delegate to validator
    txLocal = await stargateContract.connect(user).delegate(tokenId, validator.address);
    await txLocal.wait();

    // Make delegation ACTIVE (validator has completed 10 periods)
    txLocal = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
        validator.address,
        10
    );
    await txLocal.wait();

    expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
        DELEGATION_STATUS_ACTIVE
    );

    const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);

    // 3) User requests delegation exit while validator is ACTIVE
    txLocal = await stargateContract.connect(user).requestDelegationExit(tokenId);
    await txLocal.wait();

    // Sanity: endPeriod = completedPeriods + 1 = 11 → target decrease checkpoint is 12
    const { startPeriod, endPeriod } =
        await protocolStakerMockContract.getDelegationPeriodDetails(delegationId);
    expect(startPeriod).to.equal(2n);
    expect(endPeriod).to.equal(11n);

    // 4) Without advancing any period, flip validator status to EXITED,
    // keeping completedPeriods at 10 so "oldCompletedPeriods + 2" is also 12
    await (await protocolStakerMockContract.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_EXITED
    )).wait();

    // 5) Try to unstake now.
    //    Expected: Stargate attempts two decreases that land on the SAME checkpoint (period 12),
    //    causing a subtraction underflow and revert (freeze).
    // Solidity 0.8+ arithmetic under/overflow panic code
await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(0x11);


});

});
```

The PoC demonstrates that:

* requesting exit while ACTIVE schedules a first future decrease at `completedPeriods + 2`;
* if the validator later becomes EXITED, `unstake()` schedules a second decrease at `oldCompletedPeriods + 2` for the same token/validator;
* both decreases land on the same checkpoint, so `currentValue` at that period is already `0` when the second decrease runs;
* `_updatePeriodEffectiveStake` then computes `0 - effectiveStake`, triggering a Solidity ≥0.8 underflow panic (0x11);
* `unstake()` (and redelegation) consistently reverts, leaving the position non-exitable.

## References (code links)

<details>

<summary>Relevant code locations</summary>

* `Stargate.sol::_updatePeriodEffectiveStake` (decrease branch assumes `current ≥ effectiveStake`)\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L993-L1013>
* `Stargate.sol::requestDelegationExit` → schedules first decrease at `completedPeriods + 2`\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L568>
* `Stargate.sol::unstake` and `Stargate.sol::_delegate` → on `validatorStatus == EXITED || status == PENDING` schedule second decrease at `oldCompletedPeriods + 2`\
  <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L280>

</details>
