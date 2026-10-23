# 60298 sc high duplicate effectivestake decrement path bricks unstake re delegate

**Submitted on Nov 21st 2025 at 05:14:34 UTC by @Rhaydden for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60298
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Finding description and impact

Stargate tracks each validator’s total delegators’ effective stake per period using OZ checkpoints. When a user:

* delegates, it schedules an increase at validator’s next period (completedPeriods + 2),
* requests exit, it schedules a decrease at (completedPeriods + 2),
* and later unstakes while the validator is EXITED or pending, it schedules another decrease at (completedPeriods + 2).

So now the issue here is that [`requestDelegationExit()`](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L523-L571) unconditionally schedules a decrease for the upcoming period.

`unstake()` schedules another decrease if the validator is `EXITED` (or if the delegation is `PENDING`), without checking whether a decrease was already scheduled by a prior exit request. The checkpointing uses `upperLookup`, so once the first decrease writes a 0 from period N+2 onward, a second decrease at period M+2 (M ≥ N) tries to do `0 - effectiveStake` which reverts due to arithmetic underflow.

Here is the code to reference what were talking about above:

In `requestDelegationExit()` (decrease scheduled):

```solidity
// Get latest completed period
(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);
// decrease at next+1 period
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

Then in `unstake()` (second decrease scheduled when validator EXITED or delegation PENDING):

```solidity
(, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(delegation.validator);
if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING) {
    (, , , uint32 oldCompletedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, oldCompletedPeriods + 2, false);
}
```

In `_updatePeriodEffectiveStake()` (unsafe subtract on duplicate decrement):

```solidity
uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
uint256 updatedValue = _isIncrease ? currentValue + effectiveStake : currentValue - effectiveStake;
$.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
```

## Impact

Users who requested exit and whose validator later becomes EXITED cannot call `unstake()` (reverts), effectively freezing their staked VET until an upgrade. Also the same duplicate-decrement condition exists in the redelegation path (when switching validators), risking reverts or incorrect accounting if not underflowing.

Impact- Critical: Permanent freezing of funds.

Consider a simple scenario:

1. User delegates NFT T to validator V.
2. User calls `requestDelegationExit(T)` while ACTIVE. Contract schedules a decrease at period R+2.
3. Validator V later becomes `EXITED`.
4. User calls `unstake(T)`. Contract schedules another decrease at period F+2 (F ≥ R).
5. Checkpoint at F+2 already reflects the first decrease (0), so second subtraction underflows and reverts. User can’t unstake or redelegate.

## Recommended mitigation steps

In `unstake()` (and in the mirrored block inside `_delegate()`), do not schedule a second decrease if the user has already requested exit. We could:

* Compute `bool userAlreadySignaledExit = delegation.endPeriod != type(uint32).max;`
* Replace:
  * if (currentValidatorStatus == EXITED || delegation.status == PENDING) { ... decrease }
* With:
  * if (delegation.status == PENDING || (currentValidatorStatus == EXITED && !userAlreadySignaledExit)) { ... decrease }

Apply the same guard in the redelegation path (`_delegate`), where a previous delegation on an EXITED validator also triggers a decrease.

It'd be good to also add a defensive check in `_updatePeriodEffectiveStake` before decreasing:

* If `!_isIncrease`, ensure `currentValue >= effectiveStake`, otherwise revert with a descriptive custom error (prevents silent state corruption and turns it into a clear failure).

## Proof of Concept

## Proof of concept

Created a test file called `DoubleDecrementUnderflow.test.ts`:

```ts
import { expect } from "chai";
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
import { getOrDeployContracts } from "../../helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";

// PoC for double-decrement bug in delegatorsEffectiveStake
// Sequence:
// 1) delegate -> schedules +effectiveStake at completedPeriods+2 (initially 2)
// 2) make delegation ACTIVE by increasing completedPeriods to 1
// 3) requestDelegationExit -> schedules -effectiveStake at completedPeriods+2 (now 3)
// 4) set validator status to EXITED
// 5) unstake() -> attempts another -effectiveStake at same/equal period (3),
//    causing arithmetic underflow and revert

describe("PoC: Stargate double-decrement underflow on unstake", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";

    let stargate: Stargate;
    let stargateNFT: StargateNFTMock;
    let protocolStaker: ProtocolStakerMock;
    let vtho: MyERC20;
    let deployer: HardhatEthersSigner;
    let user: HardhatEthersSigner;
    let validator: HardhatEthersSigner;

    const LEVEL_ID = 1;
    const PERIOD_SIZE = 120;

    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await ethers.getSigners();

        // Deploy mocks
        const nftFactory = new StargateNFTMock__factory(deployer);
        stargateNFT = await nftFactory.deploy();
        await stargateNFT.waitForDeployment();

        const stakerFactory = new ProtocolStakerMock__factory(deployer);
        protocolStaker = await stakerFactory.deploy();
        await protocolStaker.waitForDeployment();

        // Deploy VTHO mock into the energy address
        const vthoFactory = new MyERC20__factory(deployer);
        const vthoImpl = await vthoFactory.deploy(deployer.address, deployer.address);
        await vthoImpl.waitForDeployment();
        const bytecode = await ethers.provider.getCode(vthoImpl);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, bytecode]);

        // Wire mocks into deployment
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFT.getAddress();
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStaker.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargate = contracts.stargateContract;
        vtho = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        user = contracts.otherAccounts[0];
        validator = contracts.otherAccounts[1];

        // Add validator and set active
        await (await protocolStaker.addValidation(validator.address, PERIOD_SIZE)).wait();
        await (await protocolStaker.helper__setStargate(stargate.target)).wait();
        await (
            await protocolStaker.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_ACTIVE)
        ).wait();

        // Configure NFT level and default token data
        await (
            await stargateNFT.helper__setLevel({
                id: LEVEL_ID,
                name: "Strength",
                isX: false,
                maturityBlocks: 10,
                scaledRewardFactor: 150,
                vetAmountRequiredToStake: ethers.parseEther("1"),
            })
        ).wait();
        await (
            await stargateNFT.helper__setToken({
                tokenId: 10000,
                levelId: LEVEL_ID,
                mintedAtBlock: 0,
                vetAmountStaked: ethers.parseEther("1"),
                lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
            })
        ).wait();

        // Fund Stargate with VTHO for any reward emissions
        await (await vtho.connect(deployer).mint(stargate, ethers.parseEther("1000"))).wait();
    });

    it("reverts on unstake after prior requestDelegationExit when validator EXITED (double-decrement)", async () => {
        // 1) stake
        const levelSpec = await stargateNFT.getLevel(LEVEL_ID);
        await (
            await stargate.connect(user).stake(LEVEL_ID, { value: levelSpec.vetAmountRequiredToStake })
        ).wait();
        const tokenId = await stargateNFT.getCurrentTokenId();

        // 2) delegate (schedules +effectiveStake at period = completedPeriods + 2 = 2)
        await (await stargate.connect(user).delegate(tokenId, validator.address)).wait();

        // Make delegation ACTIVE by setting completedPeriods = 1 (current period = 2)
        await (await protocolStaker.helper__setValidationCompletedPeriods(validator.address, 1)).wait();

        // 3) requestDelegationExit (schedules -effectiveStake at period = 1 + 2 = 3)
        await (await stargate.connect(user).requestDelegationExit(tokenId)).wait();

        // 4) set validator to EXITED. Keep completedPeriods = 1 so unstake subtracts again at 3
        await (
            await protocolStaker.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED)
        ).wait();

        // 5) unstake -> second subtraction at same/equal period causes arithmetic underflow
        await expect(stargate.connect(user).unstake(tokenId)).to.be.reverted; // panic underflow
        // If desired, use: .to.be.revertedWithPanic(0x11)
    });
});
```

The poc:

* Stakes and delegates a token to a validator.
* Advances periods so delegation is ACTIVE.
* Calls `requestDelegationExit()` (first scheduled decrease).
* Sets validator status to `EXITED`.
* Calls `unstake()` and expects revert due to the second scheduled decrease hitting 0 total.

Run the poc with `VITE_APP_ENV=local npx hardhat test --network hardhat test/unit/Stargate/DoubleDecrementUnderflow.test.ts`

#### Logs

```typescript
 PoC: Stargate double-decrement underflow on unstake
    ✔ reverts on unstake after prior requestDelegationExit when validator EXITED (double-decrement)


  1 passing (475ms)
```
