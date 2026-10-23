# 60506 sc high double delegatorseffectivestake decrease permanently prevents single nft from unstaking

**Submitted on Nov 23rd 2025 at 14:38:45 UTC by @cmds for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60506
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In the current `Stargate.sol`, both `requestDelegationExit` and `unstake` independently call `_updatePeriodEffectiveStake(..., false)` on the same delegation, without using `endPeriod` or any other flag to distinguish between “already decreased” and “not yet decreased”. Along the legitimate `ACTIVE → requestDelegationExit → EXITED → unstake` state machine path, this causes a double decrease of the same effective stake; in single-delegator or low-liquidity validator scenarios, the second decrease is applied in a period where the aggregated effective stake is already `0`, triggering a Solidity 0.8.20 arithmetic underflow `revert`. As a result, `unstake()` for that NFT will permanently fail, and the corresponding VET principal and future rewards cannot be withdrawn through any normal protocol entrypoint.

## Vulnerability Details

(1) Conflicting state machine: requestDelegationExit and unstake both decrease the same delegation

The core bug is that two different flows both subtract effective stake for the same delegation, with no guard to ensure it only happens once.

```language
// ✅ First decrease of this delegation's effective stake
_updatePeriodEffectiveStake(
    $,
    delegation.validator,
    _tokenId,
    completedPeriods + 2,
    false  // decrease
);
```

```language
    // ❌ Second decrease for the same delegation when EXITED / PENDING
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false  // decrease again
    );
}
```

Conceptually, the protocol should either:

* **Rule A**: decrease effective stake once in `requestDelegationExit` and never again in `unstake`
* **Rule B**: only decrease in `unstake` while `requestDelegationExit` merely signals to `ProtocolStaker`.

Currently both branches call \_updatePeriodEffectiveStake(..., false) on the same delegation and there is no “already decreased” flag, which is the root cause of the double-decrease.

(2) Existing endPeriod flag is not used as an “already decreased” guard

The contract already has a natural flag to indicate that the user requested an exit:

```language
// ACTIVE branch in requestDelegationExit
if (delegation.endPeriod != type(uint32).max) {
    revert DelegationExitAlreadyRequested(_tokenId);
}
```

\_getDelegationStatus also relies on endPeriod to determine whether the user requested an exit.

However, unstake does not use this information to avoid a second decrease. A minimal missing guard would look like:

```language
bool userRequestedExit = delegation.endPeriod != type(uint32).max;

if (
    !userRequestedExit &&  // ✅ only decrease if we never decreased in requestDelegationExit
    (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    )
) {
    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false
    );
}
```

In other words, the contract already exposes a signal for “this delegation’s effective stake has been removed at exit time” (endPeriod), but unstake ignores it and still performs another \_updatePeriodEffectiveStake(..., false) on the same stake, allowing the double-decrease to happen.

(3) \_updatePeriodEffectiveStake uses bare subtraction with no lower-bound protection

Once the state machine allows double-decrease, the arithmetic in \_updatePeriodEffectiveStake turns it into an underflow revert:

```language
   uint256 updatedValue = _isIncrease
        ? currentValue + effectiveStake
        : currentValue - effectiveStake; // ❌ underflows if currentValue < effectiveStake
```

Under the assumption that Thor’s period counter is monotonically increasing, there is a fully legitimate, non-privileged user flow:

1.The user delegates to some validator V. \_delegate performs an increase at P\_add = C0 + 2, so: delegatorsEffectiveStake\[V] = s > 0 becomes effective starting from P\_add.

2.While the validator is still ACTIVE, the user calls requestDelegationExit(tokenId): this performs the first decrease at P\_exit1 = C1 + 2 (with C1 > C0), removing this s from all future periods ⇒ from P\_exit1 onward, the aggregated value becomes 0.

3.Later, the validator becomes EXITED on the ProtocolStaker side, and \_getDelegationStatus returns DelegationStatus.EXITED.

4.At this point, the user calls unstake(tokenId):

a:The call passes the check that “unstake cannot be called while status is ACTIVE”;

b:currentValidatorStatus == VALIDATOR\_STATUS\_EXITED holds;

c:A second decrease is executed at P\_exit2 = C2 + 2 ≥ P\_exit1. At this time upperLookup(P\_exit2) == 0, so subtracting s again results in 0 - s underflow ⇒ the entire unstake reverts.

## Impact Details

Once this bug is triggered, the funds have already been withdrawn from Thor’s ProtocolStaker and are logically ready to be returned, but the double-decrease underflow in unstake prevents the NFT from being burned and the VET from being transferred back; every subsequent unstake (or related) call follows the same failing path and keeps reverting, leaving the user’s VET and all future rewards effectively permanently locked by protocol logic unless recovered through governance or emergency actions.

## Fix

A minimal viable fix is to use delegation.endPeriod != type(uint32).max inside unstake to determine whether this delegation has already had its effective stake removed via requestDelegationExit. If so, unstake must not call \_updatePeriodEffectiveStake(..., false) again. This ensures that each delegation’s effective stake is decreased at most once, and eliminates the double-decrease underflow path.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L261-L283>

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/main/packages/contracts/contracts/Stargate.sol#L398-L414>

## Proof of Concept

## Proof of Concept

MinimalStargateBug.sol (contracts/MinimalStargateBug.sol – \_updatePeriodEffectiveStake copied 1:1 from Stargate.sol, with \_calculateEffectiveStake simplified to a direct mapping)

```language
// SPDX-License-Identifier: MIT 
pragma solidity 0.8.20;

import "@openzeppelin/contracts/utils/structs/Checkpoints.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";

/// @notice Keep only the logic related to _updatePeriodEffectiveStake and
///         simulate it manually using the period parameter
contract MinimalStargateBug {
    using Checkpoints for Checkpoints.Trace224;

    // validator => checkpoints(period -> aggregated effectiveStake)
    mapping(address => Checkpoints.Trace224) internal delegatorsEffectiveStake;

    // tokenId => effectiveStake (in the real contract this is computed from level, etc.)
    mapping(uint256 => uint256) internal stakeOfToken;

    function setStake(uint256 tokenId, uint256 amount) external {
        stakeOfToken[tokenId] = amount;
    }

    /// @notice Simulate the increase logic inside _delegate
    function simulateDelegate(
        address validator,
        uint256 tokenId,
        uint32 completedPeriodsAtDelegate
    ) external {
        _updatePeriodEffectiveStake(
            validator,
            tokenId,
            completedPeriodsAtDelegate + 2,
            true
        );
    }

    /// @notice Simulate the first decrease at the end of requestDelegationExit
    function simulateRequestExit(
        address validator,
        uint256 tokenId,
        uint32 completedPeriodsAtExitRequest
    ) external {
        _updatePeriodEffectiveStake(
            validator,
            tokenId,
            completedPeriodsAtExitRequest + 2,
            false
        );
    }

    /// @notice Simulate the second decrease in unstake after the validator is EXITED
    function simulateUnstake(
        address validator,
        uint256 tokenId,
        uint32 completedPeriodsAtUnstake
    ) external {
        _updatePeriodEffectiveStake(
            validator,
            tokenId,
            completedPeriodsAtUnstake + 2,
            false
        );
    }

    /// @notice Helper for debugging: read the aggregated effective stake at a given period
    function effectiveStakeAt(
        address validator,
        uint32 period
    ) external view returns (uint256) {
        return delegatorsEffectiveStake[validator].upperLookup(period);
    }

    // ------- Internal logic identical to Stargate.sol ------- //

    function _calculateEffectiveStake(uint256 tokenId) internal view returns (uint256) {
        return stakeOfToken[tokenId];
    }

    function _updatePeriodEffectiveStake(
        address _validator,
        uint256 _tokenId,
        uint32 _period,
        bool _isIncrease
    ) internal {
        uint256 effectiveStake = _calculateEffectiveStake(_tokenId);
        uint256 currentValue = delegatorsEffectiveStake[_validator].upperLookup(_period);

        uint256 updatedValue = _isIncrease
            ? currentValue + effectiveStake
            : currentValue - effectiveStake; // ★ underflow point

        delegatorsEffectiveStake[_validator].push(
            _period,
            SafeCast.toUint224(updatedValue)
        );
    }
}

```

poc.doubleDecrease.spec.ts (test/unit/Stargate/poc.doubleDecrease.spec.ts)

```language
import { expect } from "chai";
import { ethers } from "hardhat";

describe("MinimalStargateBug double-decrease PoC", function () {
  it("second decrease underflows and reverts", async function () {
    const [deployer] = await ethers.getSigners();

    const BugFactory = await ethers.getContractFactory("MinimalStargateBug", deployer);
    const bug = await BugFactory.deploy();
    await bug.waitForDeployment();

    const validator = ethers.Wallet.createRandom().address;
    const tokenId = 1n;
    const stake = ethers.parseEther("100");

    // set effective stake s > 0
    await bug.setStake(tokenId, stake);

    // 1) delegate at completedPeriods = 1  => add at period 3
    await bug.simulateDelegate(validator, tokenId, 1);
    const valueBeforeExit = await bug.effectiveStakeAt(validator, 100);
    expect(valueBeforeExit).to.equal(stake);

    // 2) request exit at completedPeriods = 10 => first decrease at period 12
    await bug.simulateRequestExit(validator, tokenId, 10);
    const valueAfterFirstDecrease = await bug.effectiveStakeAt(validator, 100);
    expect(valueAfterFirstDecrease).to.equal(0n);

    // 3) unstake at completedPeriods = 20 => second decrease on value 0
    await expect(
      bug.simulateUnstake(validator, tokenId, 20)
    ).to.be.revertedWithPanic(0x11); // arithmetic underflow
  });
});

```

run:

```language
npx hardhat test test/unit/Stargate/poc.doubleDecrease.spec.ts
```

## Step-by-step

1.In a local test setup, deploy Stargate + ProtocolStakerMock and choose a validator V with no existing delegations (single delegator).

2.From a user account, delegate tokenId to validator V via the normal entrypoint (delegate / stakeAndDelegate).

3.While validator V is still ACTIVE, call requestDelegationExit(tokenId) from the same user.

4.Using the mock, change validator V’s status from ACTIVE to EXITED.

5.Call unstake(tokenId) from the user and observe that the transaction reverts with an arithmetic underflow panic.
