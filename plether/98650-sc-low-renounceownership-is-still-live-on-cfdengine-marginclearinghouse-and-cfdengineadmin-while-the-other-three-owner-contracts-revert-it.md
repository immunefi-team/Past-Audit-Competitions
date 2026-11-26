# \#98650 [SC-Low] renounceOwnership() is still live on CfdEngine, MarginClearinghouse and CfdEngineAdmin while the other three owner contracts revert it

**Submitted on Nov 9th 2026 at 09:25:57 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98650
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Brief/Intro

the codebase already treats "permanently giving up ownership" as a hazard and blocks it: HousePool, OrderRouterAdmin and EmergencyPauseCoordinator override `renounceOwnership()` to revert, with comments stating governance must retain recovery authority. three other governance-bearing contracts (CfdEngine, MarginClearinghouse and CfdEngineAdmin) never received that override and still expose OpenZeppelin's one-step renunciation. a single owner call on any of them zeroes the owner forever and takes the owner-only recovery functions down with it, including the engine's degraded-mode exit.

## Vulnerability Details

the intended behavior is documented in the fixed contracts:

```solidity
    /// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
@>  function renounceOwnership() public view override onlyOwner {
        revert OwnableInvalidOwner(address(0));
    }
```

(HousePool.sol:264-267; the same reverting override exists in OrderRouterAdmin.sol:123-127 and EmergencyPauseCoordinator.sol:100-104.)

the other three governance-bearing contracts inherit OpenZeppelin `Ownable2Step` without the override, so `renounceOwnership()` is the stock `Ownable` implementation: a single owner call transfers ownership to `address(0)`:

```solidity
@>  abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {
```

(CfdTreasury.sol:10, inherited by CfdEngine at CfdEngine.sol:46)

```solidity
@>  contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTransient {
```

(MarginClearinghouse.sol:30)

```solidity
@>  contract CfdEngineAdmin is Ownable2Step {
```

(CfdEngineAdmin.sol:17)

`Ownable2Step` guards `transferOwnership` with an accept step, but leaves renunciation one-step and irreversible: there is no accept path that can ever restore an owner once renounced. on the engine this permanently disables `clearDegradedMode` (CfdEngine.sol:658), `recapitalizePool` (CfdEngine.sol:632), `setDependencies` (CfdEngine.sol:345) and `setPool` (CfdEngine.sol:384); on the clearinghouse, the engine binding `setEngine` (MarginClearinghouse.sol:306) and `rescueToken` (MarginClearinghouse.sol:324); on the admin module, all risk/fee/calendar governance.

the asymmetry matters because renunciation is the only ownership action with no undo: a mis-keyed two-step `transferOwnership` can be cancelled by re-proposing, while a renunciation on these three contracts cannot be undone by anyone.

## Impact Details

one owner call on CfdEngine, MarginClearinghouse or CfdEngineAdmin irreversibly destroys that contract's governance and recovery surface. if the engine is renounced and degraded mode later latches (`_enterDegradedModeIfInsolvent`, CfdEngine.sol:1351), there is no exit: `clearDegradedMode` is `onlyOwner` with a zero owner, and the router keeps rejecting every new order with `OrderRouter__DegradedMode` (OrderLiquidationBatchLogic.sol:326-330). the renunciation fix's stated intent (governance must always retain emergency recovery authority) holds on half the owner contracts and not on the other half; applying the same three-line override to the remaining three completes the remediation.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L10

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L46

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L30

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L17

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L264-L267

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L123-L127

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/EmergencyPauseCoordinator.sol#L100-L104

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L666

## Proof of Concept

## Proof of Concept

add this file as `test/poc/RenounceOwnershipGap.t.sol` (uses the repository's `BasePerpTest` fixture, which deploys the full stack owned by the test contract):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {console2} from "forge-std/console2.sol";

/// @notice The prior review's renunciation fix (reverting renounceOwnership overrides) was applied to
///         HousePool, OrderRouterAdmin and EmergencyPauseCoordinator, but CfdEngine (via CfdTreasury),
///         MarginClearinghouse and CfdEngineAdmin still inherit OpenZeppelin's one-step renounceOwnership.
///         One owner call on any of the three zeroes the owner and permanently kills its owner-only
///         recovery surface: for the engine that includes clearDegradedMode, recapitalizePool,
///         setDependencies and setPool.
contract RenounceOwnershipGapTest is BasePerpTest {

    function test_renounceOwnershipLiveOnEngineClearinghouseAdmin() public {
        // Control: the fixed HousePool reverts and keeps its owner.
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        pool.renounceOwnership();
        assertEq(pool.owner(), address(this));

        // Engine: renunciation succeeds and zeroes the owner.
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        console2.log("engine owner after renounce:", engine.owner());

        // The degraded-mode exit is now unreachable for everyone, permanently.
        vm.expectRevert(
            abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this))
        );
        engine.clearDegradedMode();

        // Clearinghouse: same gap, setEngine binding and rescueToken die with the owner.
        clearinghouse.renounceOwnership();
        assertEq(clearinghouse.owner(), address(0));

        // Engine admin module (risk/fee/calendar governance): same gap.
        engineAdmin.renounceOwnership();
        assertEq(engineAdmin.owner(), address(0));
    }
}
```

run from `packages/perps`:

```bash
forge test --match-path test/poc/RenounceOwnershipGap.t.sol -vv
```

output:

```bash
Ran 1 test for test/poc/RenounceOwnershipGap.t.sol:RenounceOwnershipGapTest
[PASS] test_renounceOwnershipLiveOnEngineClearinghouseAdmin() (gas: 58322)
Logs:
  engine owner after renounce: 0x0000000000000000000000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.71ms (3.60ms CPU time)
```

the control leg shows HousePool reverting renunciation and keeping its owner; the three attack legs show renunciation succeeding on the engine, clearinghouse and admin, the engine owner reading `address(0)` afterwards, and `clearDegradedMode()` reverting `OwnableUnauthorizedAccount` because the only exit from degraded mode is owner-only and the owner is gone.
