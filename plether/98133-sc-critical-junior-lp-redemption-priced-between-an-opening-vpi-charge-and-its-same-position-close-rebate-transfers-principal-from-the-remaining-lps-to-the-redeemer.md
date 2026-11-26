# \#98133 [SC-Critical] Junior LP redemption priced between an opening VPI charge and its same-position close rebate transfers principal from the remaining LPs to the redeemer

**Submitted on Nov 5th 2026 at 10:27:06 UTC by @BombZee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98133
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Description**

Brief/Intro
A skew-increasing open pays a VPI charge into HousePool cash, and CfdEngineSettlementSidecar records it as LP revenue
in the same transaction. On close, the lifetime clamp in CloseAccountingLib.buildCloseState lets the same position take
that charge back as a rebate when the close reduces skew. Junior share pricing has no term for the reclaimable amount,
so while the position is open every Junior share is priced as if the charge were permanent LP equity.

An attacker with Junior shares in one wallet and a trader account in another files a redeem request, opens a
skew-increasing position 30 seconds before the hourly LP epoch boundary, settles the epoch as the permissionless keeper
so the redemption is funded at a price that includes the charge, and then closes the position. The rebate is paid from
pool cash that now belongs only to the LPs who stayed.

On the production six-currency Pyth basket, with the attacker holding 48.70% of Junior supply, the attacker ends a full
cycle at +171.43 USDG and the honest LP H realizes a principal loss of 381.50 USDG. Against a matched control where the
same trade is made but the attacker redeems after the close, honest LPs lose 1,075.09 USDG and the attacker gains the
same amount.

Vulnerability Details
The opening charge is collected into pool cash and recorded as revenue in CfdEngineSettlementSidecar.sol lines 446 to
463:

    (int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
        delta.account,
        delta.marginDeltaAssets,
        delta.tradeCostAssets,
        host.pool(),
        host.protocolTreasury(),
        delta.executionFeeAssets
    );
    if (delta.tradeCostAssets > 0) {
        uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
        if (poolCashInflowAssets > 0) {
            IHousePool(host.pool())
                .recordClaimantInflow(
                    poolCashInflowAssets,
                    IHousePool.ClaimantInflowKind.Revenue,
                    IHousePool.ClaimantInflowCashMode.CashArrived
                );
        }

HousePool.recordClaimantInflow adds it to accountedAssets at HousePool.sol lines 640 to 641:

    if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
        accountedAssets += amount;
    }

LP pricing is physical assets minus trader claims plus the terminal price delta. buildReconcileSnapshot in
HousePoolAccountingLib.sol lines 60 to 77 has no term for VPI that open positions can still take back:

    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
    if (terminalDelta >= 0) {
        terminalAssets += uint256(terminalDelta);
    } else {
        terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
    }

At the epoch boundary HousePool.settleLpEpoch (line 797), which anyone can reach through OrderRouter.settleLpEpoch
(line 479), reconciles at this price. _fundRedemptionPhase (line 944) funds the matured redemption from it and sends the
USDG to the vault at lines 1007 to 1008:

    settlementToken.safeTransfer(vaultAddress, fundedAssets);
    vault.fundRedeemEpoch(epochId, fundedShares, fundedAssets);

On close, CloseAccountingLib.sol lines 112 to 118 allow a negative (rebate) VPI delta down to the position's own accrued
charge:

    // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
    // where an attacker opens at high depth, reduces pool depth, then closes to extract a
    // net-negative lifetime VPI rebate. ...
    if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
        state.vpiDeltaAssets = -state.proportionalAccrualAssets;
    }

The rebate is paid from pool cash at CfdEngineSettlementSidecar.sol lines 621 to 624:

    if (delta.actionRebatePaidAssets > 0) {
        IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
        clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
    }

At constant depth, a position that opened on the skew-increasing side and closes on the skew-reducing side gets back its
full charge minus fees. The redemption at HousePool.sol line 1007 is funded at the specified NAV, but that NAV includes
a VPI charge the attacker originated and later has rebated to the same economic actor. The timing difference moves value
from the remaining Junior LPs to the exiting LP.

Every call in the sequence is public: TrancheVault.requestRedeem (line 606, at least LP_REQUEST_CUTOFF_DURATION = 5
minutes before the boundary), OrderRouter.commitOrder and executeOrder (the keeper is permissionless),
OrderRouter.settleLpEpoch, TrancheVault.claimRedeem (line 1014) and MarginClearinghouse.withdraw. No role, signature
authority or privileged configuration is used during the attack.

The LP wallet and the trader wallet are two ordinary addresses that nothing on-chain links. Junior deposits check only
addresses, caller authorization, pool state, minimum size and capacity (TrancheVault.sol line 562 and lines 1579 to
1596, HousePoolTrancheGateLib.trancheDepositsAllowed). Margin deposits check only for a non-zero account and amount
(MarginClearinghouse.sol lines 348 and 371). OrderRouter.commitOrder (line 85) trades for the caller. There is no
allowlist and no separation between LPs and traders, so one person signing for both addresses meets every precondition.

Impact Details
Honest Junior LPs take a principal loss that is realized when they redeem, at a flat price with no open positions left.
The attacker needs no privileged role.

| Scenario | Attacker | H (honest LP), realized | Other honest LPs |
|---|---|---|---|
| 48.70% of Junior, production 6-feed basket | +171.432533 | -381.499501 | -381.843076 |
| 48.70% of Junior, fixture 2-feed basket | +185.650086 | -389.288809 | -389.641277 |
| 55.54% of Junior, fixture basket | +436.479007 | -560.217863 | -560.741144 |
| 48.70%, three cycles in a row, no reset, fixture basket | +559.407362 | -1,169.220591 | - |
| 25.9% of Junior (negative control) | -316.05 | -47.41 | - |

In the matched control at 48.70% on the production basket, the trade, prices and clock are identical and only the
attacker's redemption moves to after the close. The attacker ends at -903.655833 instead of +171.432533, and all honest
LPs end at +311.745789 instead of -763.342577. The difference is +1,075.088366 to the attacker and -1,075.088366 to
honest LPs. Every scenario conserves the USDG ledger to the atom (assertEq), with treasury and keeper deltas included.

Limitations and specification status
The implementation follows its accounting specification. LP pricing is ACCOUNTING_SPEC section "Symmetric exact marked
NAV" (ACCOUNTING_SPEC.md lines 158 to 172). The specification keeps anticipated VPI outside NAV (ACCOUNTING_SPEC.md
lines 1064 to 1066, SECURITY.md line 872), pays close rebates only from free cash and waives any unfunded part
(ACCOUNTING_SPEC.md lines 1061 to 1063), and accepts that collection timing can move part of old carry revenue to a new
LP (ACCOUNTING_SPEC.md lines 736 to 741).

The specification documents cash-basis recognition, but it does not explicitly address the repeatable cross-epoch
transfer caused by combining a refundable VPI charge, an LP redemption and the payer's subsequent rebate. This finding
is about that interaction between two permitted operations, a redemption at the specified NAV and a clamp-permitted
rebate to the same position, which I consider unintended. It differs from the accepted carry case in three ways: the
revenue is refunded to the actor who paid it, the remaining LPs lose principal rather than a share of revenue, and the
clamp's own comment (CloseAccountingLib.sol lines 112 to 114) names LP sandwich attacks around VPI as the threat it is
meant to stop, while this sequence works at constant depth.

The attack becomes profitable at about 43% of Junior supply. Below that the honest LP still loses, but so does the
attacker. The PoC uses 1.9M of attacker LP against 2.0M of honest Junior, plus 124k of trader collateral. Capital has
to stay in through the async deposit, the 1 hour cooldown and one epoch, so flash loans don't work. The amount per
cycle is bounded by s·c minus fees, where c is at most about 0.0008 of depth at the 0.4 skew cap; at 48.70% the transfer
is about 0.05% of honest equity per cycle. Redemption funding is capped at free cash, so at about 66% ownership the
rest of the request stays queued (test_reserveGateRespected).

The position is open for 35 seconds between open and close execution, on 0.38 of depth in notional. The transfer from
honest LPs does not depend on price and is asserted positive at every tested move from -2 bp to +2 bp. The attacker's
absolute profit moves by about 110 USDG per basket basis point and breaks even at about +1.5 bp adverse.

The profit figures are net of protocol execution fees and keeper bounties only. They do not include gas or Pyth update
fees (the mock fee is 0). A cycle is about 11 transactions plus Pyth updates for the open, the settlement and the close.
Off-protocol hedging is not modelled. Another trader reducing skew inside the window would take part of the rebate.

Relation to known issues
SCAN-31 is about lowering depth to amplify a skew-reducing close rebate. In arm C′ an honest depositor restores depth
in the same settlement, so close-time depth is constant, and the attacker keeps +435.437315 of the +436.479007 profit
(99.76%). The rebate differs between attack and control by 0.09% (1.96 USDG on the production basket). The transfer
comes from who holds shares when the rebate is paid, not from the size of the rebate.

SC-H1 (deposit before senior coupon accrual) involves a different revenue source, the opposite direction (entry rather
than exit) and a different tranche.

SECURITY.md line 1443 ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") covers
the trader's own round trip, which nets zero VPI here as intended. I am not claiming it is violated.

Remediation
An experimental mitigation, attached as mitigation.diff (3 files, about 16 lines, HousePool unchanged), tracks the
signed sum of vpiAccrued over open positions in CfdEngine and subtracts it from terminalLpPriceDeltaAssets in
CfdEngineProtocolLens.getHousePoolInputSnapshot. On the 55.54% sequence the attacker ends at -1,368.80, which is fees
only.

It is not a complete or deployment-safe fix. It moves VPI recognition from open to close, so a depositor who enters
before another trader's close gains +93.67 in the tested case. It also departs from ACCOUNTING_SPEC.md lines 1064 to
1066, and it has not been run against the production test suite. Other options are a live-mode redemption fee at least
equal to the largest single VPI round trip, or pricing LP exits net of open positions' positive vpiAccrued.

References
packages/perps/src/CfdEngineSettlementSidecar.sol:446-463 (opening VPI collected and recorded as pool revenue)
packages/perps/src/HousePool.sol:624-641 (recordClaimantInflow, accountedAssets += amount)
packages/perps/src/libraries/HousePoolAccountingLib.sol:60-77 (buildReconcileSnapshot, no reversible-VPI term)
packages/perps/src/HousePool.sol:797, 944, 1007-1008 (settleLpEpoch, _fundRedemptionPhase, USDG to vault)
packages/perps/src/OrderRouter.sol:479 (settleLpEpoch, permissionless)
packages/perps/src/libraries/CloseAccountingLib.sol:112-118 (lifetime clamp, comment naming LP sandwich attacks)
packages/perps/src/CfdEngineSettlementSidecar.sol:621-624 (close rebate paid from pool cash)
packages/perps/src/TrancheVault.sol:606, 1014, 1478 (requestRedeem, claimRedeem, fundRedeemEpoch)
packages/perps/ACCOUNTING_SPEC.md:158-172, 736-741, 1061-1066
packages/perps/SECURITY.md:872, 1443
script/DeployPerpsArbitrumSepolia.s.sol:147-166, 464-491 (production basket configuration)
packages/perps/DEPLOYMENT.md (release risk parameters used in all tests)

## Link to Proof of Concept

https://gist.github.com/suvamc/ef1c89f53781362cb29fbb4b1e070e91

## Proof of Concept

**Proof of Concept**

Proof of Concept
The PoC is four Foundry test files that run against the frozen commit with the repository's own BasePerpTest fixture.
They go in packages/perps/test/poc/:

VpiPocBase.t.sol: base harness (keeper, independent pool-equity read, mark refresh, commit and execute)
VpiFixture.t.sol: actors, release risk parameters, deposit, open, close and settle helpers
VpiRedemptionTransfer.t.sol: absolute ledger, custody checks, matched controls, 48.70% case and three cycles, on the
fixture oracle
VpiRedemptionTransferPyth.t.sol: production six-feed Pyth basket, matched control, price paths and wide confidence

BasePerpTest.sol (packages/perps/test/perps/) and MockPyth.sol (packages/shared/test-support/) come from the repository
at this commit. The import chain is VpiRedemptionTransferPyth, VpiRedemptionTransfer, VpiFixture, VpiPocBase,
BasePerpTest. Shared helpers sit in abstract contracts, so each test file runs only its own tests.

The tests use the release risk parameters from DEPLOYMENT.md: vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintenance 10
bps, initial 20 bps, execution fee 4 bps, protocolShareBps 0, Senior cap 40M / 80%. MockPyth replaces Pyth attestation
verification. Feed values, confidence intervals and publication times are supplied explicitly, and Plether's oracle
selection and settlement logic are unchanged. USDG is the fixture's test token. The attack uses no vm.store, vm.etch
or role grants.

VpiRedemptionTransferPyth.t.sol installs the production oracle in setUp through
OrderRouterAdmin.proposeOracleConfig/finalizeOracleConfig and proposeRouterConfig/finalizeRouterConfig, with the 72
hour timelocks. The feeds are EUR/USD, USD/JPY (inverted), GBP/USD, USD/CAD (inverted), USD/SEK (inverted) and USD/CHF
(inverted), using the real Pyth IDs, with weights 0.576/0.136/0.119/0.091/0.042/0.036, base prices from the deploy
script, adverse-confidence multiplier 2500, and all other oracle policy at contract defaults (staleness 60 s,
settlement window 15 s, divergence 5 s, confidence cap 10 bps). Ticks use Pyth-native exponents, per-component
confidence of 0.15 to 0.25 bp, and publish times staggered 0 to 3 seconds per feed. Order execution uses each feed's
first tick after the commit, and settlement requires every component to be at or after the epoch boundary. The scenario
runs on a Tuesday or Wednesday at 10:00 UTC, and the test asserts it is neither in the FAD window nor oracle-frozen.

Reproduction steps against a checkout of a82d4fef:

    git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init
    # copy the four test files into packages/perps/test/poc/
    forge test --root packages/perps --match-path "test/poc/VpiRedemptionTransfer.t.sol" \
      --skip "test/perps/**/*.t.sol" -vv -j 1 --memory-limit 8589934592 --gas-limit 18000000000000
    forge test --root packages/perps --match-path "test/poc/VpiRedemptionTransferPyth.t.sol" \
      --skip "test/perps/**/*.t.sol" -vv -j 1 --memory-limit 8589934592 --gas-limit 18000000000000

Tested with Forge 1.5.1-stable, the version pinned in DEPLOYMENT.md. The first run compiles the package with via_ir.

Test sequence (_pArm(2, ...) in VpiRedemptionTransferPyth.t.sol, _absArm(2, ...) in VpiRedemptionTransfer.t.sol):

1. H deposits 1,000,000 USDG and X deposits 1,900,000 USDG into Junior through the async request, epoch settlement and
   claim, then waits out the 1 hour cooldown. X holds 4,870 bps of Junior supply.
2. T deposits margin into the clearinghouse.
3. X calls requestRedeem(allShares) with more than 5 minutes left before the boundary. The test asserts that the
   request id equals the boundary epoch.
4. At B - 30, T calls commitOrder(LONG, 0.38 x depth, margin). At B - 26 the keeper calls executeOrder with the first
   post-commit tick of each feed.
5. At B + 3, X calls OrderRouter.settleLpEpoch with ticks at or after B, then claimRedeem. The test asserts that X is
   fully paid while T's position is still open.
6. At B + 5, T commits the close, and at B + 9 the keeper executes it. T withdraws everything.
7. Full unwind: X holds no shares and has no pending request, and H redeems all shares in the next epoch. The test
   asserts no open position, no clearinghouse balance, no trader claims, and all USDG with known holders.

The control arm (kind 0) is identical except that X files its redemption after the close.

Captured output from VpiRedemptionTransferPyth.t.sol, production basket, 48.70%:

    Ran 3 tests for test/poc/VpiRedemptionTransferPyth.t.sol:VpiRedemptionTransferPythTest
    [PASS] test_pythBasket_attackVsControl() (gas: 15782640)
    Logs:
      production basket mark (1e8) at setup 99769483
      A trade-only control (production basket)
         X share of junior supply (bps) 4870
         attacker in / out 2024135000000 2023231344167
         ATTACKER PROFIT (A)
      -903655833
         H deposited / H redeemed (realized) 1000000000000 1000155776940
      155776940
         other honest LP equity delta / treasury delta / keeper delta
      155968849
      591710044
      200000
         VPI open inflow / rebate paid / X funded at attacked boundary 2804016984 2210340898 0
         basket mark at open / at close (1e8), exposure seconds 99769483 99769483 35
      C attack (production basket)
         X share of junior supply (bps) 4870
         attacker in / out 2024135000000 2024306432533
         ATTACKER PROFIT (A)
      171432533
         H deposited / H redeemed (realized) 1000000000000 999618500499
      -381499501
         other honest LP equity delta / treasury delta / keeper delta
      -381843076
      591710044
      200000
         VPI open inflow / rebate paid / X funded at attacked boundary 2804016984 2212301329 1901369117416
         basket mark at open / at close (1e8), exposure seconds 99769483 99769483 35
      INCREMENTAL C - A: attacker / H / other honest LPs / all honest LPs
      1075088366
      -537276441
      -537811925
      -1075088366
      rebate A / rebate C 2210340898 2212301329

test_pythBasket_pricePathGrid runs both arms on the same linear price path: settlement sees half the move and the close
sees the full move. A positive move is adverse to the LONG position. Values in USDG:

| Basket move | Attack | Control | Attack - control | H, attack - control |
|---|---|---|---|---|
| -2 bp | +392.508590 | -753.358596 | +1,145.867186 | -572.648224 |
| -1 bp | +284.416118 | -826.942027 | +1,111.358145 | -555.402297 |
| -0.5 bp | +229.861705 | -864.628133 | +1,094.489838 | -546.972344 |
| 0 | +171.432533 | -903.655833 | +1,075.088366 | -537.276441 |
| +0.5 bp | +113.035959 | -942.751742 | +1,055.787701 | -527.630915 |
| +1 bp | +58.511602 | -980.415086 | +1,038.926688 | -519.204607 |
| +2 bp | -49.765697 | -1,054.081810 | +1,004.316113 | -501.907939 |

At every move the test asserts that attack minus control is positive for the attacker and negative for H. With
confidence widened 5x to about 1 bp per component (test_pythBasket_wideConfidence), the attack ends at +129.297926, the
control at -932.077826, and the difference is +1,061.375752.

Captured output from VpiRedemptionTransfer.t.sol, fixture basket, 48.70%, absolute ledger in USDG atoms:

    [PASS] test_profitableBelowHalfOwnership() (gas: 7139414)
      BEFORE
        ATTACKER X wallet / T wallet / T clearinghouse 0 124135000000 0
        VICTIM   H wallet / H shares 0 1000009798098907
        PROTOCOL pool raw / accounted / claims 3902000000000 3902000000000 0
      MID (attack complete, before final exits)
        ATTACKER X wallet / T wallet / T clearinghouse 1901372031942 122948618144 0
        PROTOCOL pool raw / accounted / claims 2001221069914 2001221069914 0
                 treasury / keeper 593080000 200000
      AFTER (complete unwind, H redeemed)
        ATTACKER X wallet / T wallet / T clearinghouse 1901372031942 122948618144 0
        VICTIM   H wallet / H shares 999610711191 0
        PROTOCOL pool raw / accounted / claims 1001610358723 1001610358723 0
      C below 50%
         X share of junior supply (bps) 4870
         attacker in / out 2024135000000 2024320650086
      185650086
         H deposited / H redeemed (realized) 1000000000000 999610711191
      -389288809
         other honest LP equity delta / treasury delta / keeper delta
      -389641277
      593080000
      200000
         VPI open inflow / rebate paid / X funded at attacked boundary 2817016004 2223936004 1901372031942

The ledger conserves exactly: 185.650086 - 389.288809 - 389.641277 + 593.080000 + 0.200000 = 0.

VpiRedemptionTransfer.t.sol passes 7/7 (attack full ledger, matched controls A to F and C′, below-threshold control,
48.70% case, three cycles, reserve gate, price sensitivity), and VpiRedemptionTransferPyth.t.sol passes 3/3. Both
commands were run as written on a fresh worktree of a82d4fef with no tracked changes, submodules at their pinned
commits, and only the four attached files added.

Attachments: VpiPocBase.t.sol, VpiFixture.t.sol, VpiRedemptionTransfer.t.sol, VpiRedemptionTransferPyth.t.sol,
mitigation.diff.
