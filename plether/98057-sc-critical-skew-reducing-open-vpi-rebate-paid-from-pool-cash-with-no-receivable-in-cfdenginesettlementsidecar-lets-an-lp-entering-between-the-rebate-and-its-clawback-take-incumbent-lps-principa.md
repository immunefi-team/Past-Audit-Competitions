# \#98057 [SC-Critical] Skew-reducing-open VPI rebate paid from pool cash with no receivable in `CfdEngineSettlementSidecar` lets an LP entering between the rebate and its clawback take incumbent LPs' principal

**Submitted on Nov 4th 2026 at 17:09:30 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98057
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader opens a position that reduces the pool's open skew, the pool immediately pays that position a VPI rebate out of pool cash and records no offsetting claim against it, so the junior tranche NAV drops at once. At close the rebate is clawed back from every current junior holder, lifting NAV again. An attacker funds a junior deposit that activates in the hourly LP epoch between the rebate and its clawback: it buys junior shares at the understated NAV, then shares in the clawback that restores NAV, taking value out of the incumbent junior LPs. In the headline test an incumbent junior LP holding 1,000,000 USDG falls from 1,000,719.82 USDG in an untouched twin run to 1,000,120.49 USDG, a 599.34 USDG loss in one cycle, while the attacker's 5,000,000 USDG junior position gains 599.93 USDG gross and nets 295.93 USDG after its trading cost. The loss repeats every epoch. This is the mirror, on the rebate side, of a separate report on the skew-increasing side of the same VPI accounting.

## Vulnerability Details

On a skew-reducing open the planner sets a pool rebate for the position: `CfdEnginePlanLib.sol:333` sets `delta.poolRebatePayoutAssets` to the magnitude of a negative trade cost. The settlement sidecar then pays that rebate straight out of pool cash with no receivable recorded against the opening position: `CfdEngineSettlementSidecar.sol:442-444` calls `IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets)`. Junior NAV is understated from that moment until the position closes.

A junior deposit that is pending for the current LP epoch is activated during settlement on the junior principal basis: `HousePool.sol:915-919` prices the junior activation on `juniorPrincipal`, and `TrancheVault.sol:1408` mints the entrant's shares on that basis. An entrant whose deposit activates while the rebate is outstanding therefore buys shares at the understated NAV.

At close the rebate is reversed through the VPI close accounting (`CloseAccountingLib.sol:116-118`), so pool cash is restored and junior NAV recovers. The entrant's shares, bought cheap, now share in that recovery, and the value comes from the incumbent junior LPs who held across the whole round trip.

**Who can trigger it, and what it needs.** The attacker is any EOA. It needs a skew in the book to reduce (in the test a separate whale account supplies it), capital to post as the margin of the skew-reducing position, and the stake it deposits as a junior LP. The settlement call that activates the deposit (`router.settleLpEpoch`) is a permissionless keeper call; the test makes it from an unprivileged address. No privileged role, no governance action, no oracle move, and no flash loan: the mark price is held constant throughout. The extraction repeats each hourly LP epoch.

**What the project may cite.** `packages/perps/SECURITY.md:872` states that "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV." That addresses anticipated VPI held in the reserve; it does not cover the cash-timing gap here, where a rebate already paid out of pool cash understates NAV and a same-window entrant then collects the clawback.

**Recommended fix.** Record the skew-reducing-open rebate as a clawback receivable against the opening position instead of paying it out of pool cash at open, so junior NAV is not understated between the rebate and its clawback and an entrant activating in that window cannot buy the understated NAV and then collect the recovery.

## Impact Details

Direct theft of incumbent junior LP principal (impact 15), moved to an entrant that times its deposit into the window between a skew-reducing-open rebate and its clawback. The funds at risk are junior LP principal held in HousePool and the junior TrancheVault. USDG is one-to-one with USD.

Per-cycle figures from the tests:
- `test_S5b_entryBetweenRebateAndClawbackNetProfit`: incumbent junior LP 1,000,000 USDG, entrant stake 5,000,000 USDG. The incumbent falls from 1,000,719.82 USDG in the untouched twin to 1,000,120.49 USDG, a 599.34 USDG loss. The entrant's junior position is worth 5,000,599.93 USDG against its 5,000,000 stake (599.93 USDG gross), and its skew-reducing position round trip costs it 304.00 USDG, for a net 295.93 USDG gain in one cycle.
- `test_S5d_secondCycleRepeats`: the attacker exits its first stake above 5,000,000 USDG (a gain, asserted) and runs the entry again; the incumbent, now holding about 1,000,120.48 USDG, pays again, ending at 999,520.72 USDG against 1,000,120.47 USDG owed in the twin. The extraction repeats each epoch.

**Capital dependence and honest severity (required disclosure).** The net profit depends on the entrant staking several times the incumbent tranche. At the 5,000,000 USDG stake above the entrant nets 295.93 USDG per cycle. At a 50,000 USDG stake (`test_S5_S34_entryBetweenRebateAndClawback`) the entrant's junior gain is only 11.86 USDG, smaller than the 80.00 USDG cost of its skew-reducing round trip, so the entrant nets a 68.14 USDG loss. The per-cycle extraction is bounded by the installed `vpiFactor` of 0.01e18, about 599 USDG on a 1,000,000 USDG incumbent, roughly 0.06% of the incumbent per cycle, and the profit scales with the entrant's stake relative to the incumbent tranche. Because a profitable attack needs large relative capital, the economic risk sits between Critical (the program's impact text for direct theft of principal) and Medium; the facts are stated here so the severity can be judged on them. A guardian pause of LP settlement would contain the attack, but the program page (Operating Assumptions and Downgrade Criteria) states that monitoring or pause functionality alone does not justify a downgrade.

### Before / after state delta (`test_S5b`, USDG, 6 decimals, per cycle)
| Account / state | Before (untouched twin) | After | Delta |
|---|---|---|---|
| incumbent junior LP (1,000,000) | 1,000,719.82 | 1,000,120.49 | -599.34 |
| entrant junior position (5,000,000 stake) | 5,000,000.00 | 5,000,599.93 | +599.93 gross |
| attacker skew-reducing round trip (500,000 collateral) | 500,000.00 | 499,696.00 | -304.00 |
| attacker net (entrant gross minus round-trip cost) | n/a | n/a | +295.93 |

At a 50,000 USDG entrant stake (`test_S5_S34`): entrant gross +11.86, round-trip cost 80.00, net -68.14 (a loss), which is the capital dependence.

## References
- Vulnerable path, pinned build `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:
  - `packages/perps/src/libraries/CfdEnginePlanLib.sol:333` - the skew-reducing-open rebate is planned
  - `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444` - the rebate is paid from pool cash with no receivable
  - `packages/perps/src/HousePool.sol:915-919` and `packages/perps/src/TrancheVault.sol:1408` - the entrant's junior deposit is activated and priced on the understated NAV
  - `packages/perps/src/libraries/CloseAccountingLib.sol:116-118` - the rebate is reversed at close for every current holder
- Design note the project may cite: `packages/perps/SECURITY.md:872` - anticipated VPI is excluded from terminal `K` and symmetric NAV; it does not cover the cash-timing gap exploited here.
- Not a known issue. No prior audit (Cecuro, pre-audit, collaborative, Cantina) or scan item describes a junior LP entry between a paid-out skew-reducing VPI rebate and its clawback. The nearest scan item is an informational note that a temporary pool-liquidity reduction can amplify a skew-reducing close rebate, a different mechanism; a separate fixed item on a negative VPI term in the initial-margin projection is unrelated. This report and a separate report on the skew-increasing side of the same VPI accounting share the VPI-accounting root and could be filed as one combined report; they are filed separately for clarity.


## Link to Proof of Concept

https://gist.github.com/tdev00/a98bd41a0d465dcea84e7c0c435cdd0b

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the test executes it, risk parameters installed in `setUp`).**
1. The harness installs the risk parameters in `setUp` (`vpiFactor` 0.01e18, max skew ratio 0.4e18, base carry 500 bps, maintenance and initial margin 100 and 150 bps; the release margins are 10 and 20 bps). Re-running the three cited tests with the full release configuration (`setUp` set to margins 10 and 20) gave the same figures. Only the Pyth price feed and the USDG settlement token are mocked; every other contract is the real one.
2. An incumbent junior LP deposits 1,000,000 USDG into the junior tranche, and the deposit is activated through a settled LP epoch so the LP holds live junior shares. Time then advances one hour.
3. A separate whale account and the attacker are each funded with 500,000 USDG of trader collateral in the clearinghouse. The whale opens a 380,000-unit long at price 1e8 with 100,000 USDG margin, which skews the book. Orders are submitted by pranking the router address, which is the authority the production router and keeper use to submit a user's order.
4. The test snapshots state, runs a no-attack twin that only advances to the next LP epoch and reprices (the incumbent is worth 1,000,719.82 USDG), then reverts to the snapshot.
5. Attack run, with the attacker staking 5,000,000 USDG:
   a. The attacker requests a 5,000,000 USDG junior deposit, pending for the next LP epoch; time warps to that epoch's start.
   b. The attacker opens a 380,000-unit short with 100,000 USDG margin at 1e8. It exactly offsets the whale's long, so the open is skew-reducing and the pool pays the position a VPI rebate out of pool cash, understating junior NAV.
   c. One second later an unprivileged address calls `router.settleLpEpoch` (not pranked), which settles the epoch and activates the attacker's pending deposit at the understated NAV, minting its junior shares cheap.
   d. One second later the attacker closes the short, which reverses the rebate at close and restores junior NAV.
   e. The mark price is held at 1e8 at every step; no price move is used. After 30 more seconds and a reprice, the incumbent's junior value and the attacker's activated junior position are read.
6. The attacker's clearinghouse collateral is read before and after the short round trip: 500,000.00 USDG then 499,696.00 USDG, so the round trip cost it 304.00 USDG net.
7. The test asserts `assertLt(held, owed)`: the incumbent ends at 1,000,120.49 USDG against 1,000,719.82 USDG in the twin, a 599.34 USDG loss, while the attacker's position is worth 5,000,599.93 USDG against its 5,000,000 stake (599.93 USDG gross), for a net 295.93 USDG. The attacker's gain comes out of the incumbent LP's principal.

A second test (`test_S5d_secondCycleRepeats`) has the attacker exit its first stake above 5,000,000 USDG (asserted) and run the same entry again; the incumbent pays a second time (ends at 999,520.72 USDG against 1,000,120.47 USDG owed), so the extraction repeats each epoch. A third test (`test_S5_S34_entryBetweenRebateAndClawback`) runs the same route at a 50,000 USDG stake, where the entrant's 11.86 USDG junior gain is smaller than its 80.00 USDG round-trip cost, a net loss, which shows the capital dependence.

**Runnable tests.** Three Foundry tests in one file, run through the `BasePerpTest` harness. Only the Pyth feed and the USDG settlement token are mocked, which the rules permit. The full test file (TriageLpEntry.t.sol) and the captured forge log are in the secret Gist linked below.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy TriageLpEntry.t.sol into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the file imports the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/`:
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/TriageLpEntry.t.sol --match-test test_S5b_entryBetweenRebateAndClawbackNetProfit -vv
   forge test --match-path poc/TriageLpEntry.t.sol --match-test test_S5d_secondCycleRepeats -vv
   forge test --match-path poc/TriageLpEntry.t.sol --match-test test_S5_S34_entryBetweenRebateAndClawback -vv
   ```

The route both runs (twin and attack) share one helper. The attacker requests the junior deposit, opens the skew-reducing short, lets a permissionless settlement activate the deposit, then closes the short:

```solidity
function _rebateRouteSized(
    bool attack,
    uint256 stake,
    uint256 size
) internal returns (uint256 incumbent, uint256 entrantValue) {
    _pushMark(1e8);
    uint256 id = attack ? _request(juniorVault, ENTRANT, stake) : pool.currentLpEpoch() + 1;
    vm.warp(pool.lpEpochStart(id));
    _pushMark(1e8);
    if (attack) {
        _open(ENTRANT, CfdTypes.Side.SHORT, size, 100_000e6, 1e8);
        vm.warp(_now() + 1);
        _pass(1e8);
        vm.warp(_now() + 1);
        _pushMark(1e8);
        _close(ENTRANT, CfdTypes.Side.SHORT, size, 1e8);
    }
    vm.warp(_now() + 30);
    _pushMark(1e8);
    incumbent = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
    if (attack) {
        entrantValue = juniorVault.convertToAssets(juniorVault.claimableDepositShares(id, ENTRANT));
    }
}
```

Headline test:

```solidity
function test_S5b_entryBetweenRebateAndClawbackNetProfit() public {
    _fundJunior(JUNIOR_LP, 1_000_000e6);
    vm.warp(_now() + 1 hours);
    _fundTrader(TRADER, 500_000e6);
    _fundTrader(ENTRANT, 500_000e6);
    uint256 size = 380_000e18;
    _open(TRADER, CfdTypes.Side.LONG, size, 100_000e6, 1e8);
    uint256 stake = 5_000_000e6;
    uint256 branch = vm.snapshotState();
    (uint256 owed,) = _rebateRouteSized(false, stake, size);
    vm.revertToState(branch);
    uint256 legBefore = _settlementBalance(ENTRANT);
    (uint256 held, uint256 entrantValue) = _rebateRouteSized(true, stake, size);
    uint256 legAfter = _settlementBalance(ENTRANT);
    console2.log("incumbent owed, no attack", owed);
    console2.log("incumbent held", held);
    console2.log("entrant value", entrantValue);
    console2.log("entrant stake", stake);
    console2.log("trader leg before", legBefore);
    console2.log("trader leg after", legAfter);
    assertLt(held, owed, "incumbent junior LP is below the run with no attack");
}
```

`test_S5d_secondCycleRepeats` additionally asserts `assertGt(_claimRedeem(juniorVault, exitId, ENTRANT), stake, "first stake out with its gain")` (the attacker exits its stake at a gain) and `assertLt(held, owed, "incumbent junior LP pays again in the second cycle")`.

Captured output (TriageLpEntry.log in the Gist):

```
[PASS] test_S5_S34_entryBetweenRebateAndClawback() (gas: 7836821)
Logs:
  incumbent owed, no attack 1000448635106
  incumbent held 1000436784835
  entrant value 50011862128
  entrant paid 50000000000
  trader leg before 200000000000
  trader leg after 199919999981

[PASS] test_S5b_entryBetweenRebateAndClawbackNetProfit() (gas: 7836163)
Logs:
  incumbent owed, no attack 1000719821187
  incumbent held 1000120485943
  entrant value 5000599934320
  entrant stake 5000000000000
  trader leg before 500000000000
  trader leg after 499695999855

[PASS] test_S5d_secondCycleRepeats() (gas: 14160480)
Logs:
  incumbent before the second cycle 1000120482835
  incumbent owed, no second cycle 1000120473706
  incumbent held 999520719665
  entrant value 5000600353535
  trader leg cost 304000145
```

The file compiles with Solc 0.8.35 and the suite reports all tests passing (0 failed).

