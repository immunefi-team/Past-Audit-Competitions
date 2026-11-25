# \#99145 [SC-Critical] Refundable VPI Inflates Junior NAV, Enabling Profitable Extraction of Passive LP Principal

**Submitted on Nov 12th 2026 at 04:24:34 UTC by @xRakan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99145
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

### Summary

An ordinary trader opens a position that pays positive virtual price impact (VPI) into the HousePool. Those USDG increase the pool's physical assets immediately. While the position remains open, its accrued VPI represents potentially refundable value. Whether any rebate is produced depends on the close-time VPI calculation; the lifetime clamp bounds it by the accrued VPI. HousePool's terminal NAV counts the VPI cash but does not account for this potential close-time outcome when it prices Junior redemptions.

A coalition LP can request and claim a redemption after the VPI payment enters NAV but before the trader closes. If the close-time calculation produces a rebate, the trader can receive it from remaining free pool cash, subject to the lifetime clamp and any waiver for insufficient cash. The redeeming LP keeps the share of cash already included in its exit price; the Junior LPs who remain fund any rebate that is paid.

At the primary launch-parameter point and a fixed 1.0 mark, the coalition realizes **+1,047.865777 USDG** and passive Junior LP V has a realized full-exit change of **−1,104.190974 USDG**. Same-time controls instead produce V gains of **+200.189681 to +212.317465 USDG** over its starting value. The measured loss does not require a rebate greater than the VPI previously accrued, price movement, or oracle manipulation.

## Vulnerability Details
### Root cause

[HousePool.settleLpEpoch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L840) reconciles the pool before it funds matured redemption requests. Its canonical [terminal-NAV snapshot](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L78) starts from physical assets, trader claims, and terminal price PnL. It has no input or adjustment for outstanding VPI:

```solidity
function buildReconcileSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
) internal pure returns (ReconcileSnapshot memory snapshot) {
    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
    if (terminalDelta >= 0) {
        terminalAssets += uint256(terminalDelta);
    } else {
        terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
    }

    if (terminalAssets >= terminalLiabilities) {
        snapshot.distributable = terminalAssets - terminalLiabilities;
    } else {
        snapshot.deficit = terminalLiabilities - terminalAssets;
    }
}
```

The Engine stores each open position's signed lifetime VPI; positive means the trader paid the pool, and negative means the pool paid a rebate. On close, the lifetime clamp prevents the signed balance from going below zero:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The close planner pays the action rebate out of free pool cash and waives an unfunded remainder:

```solidity
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

The test verifies sufficient free cash and the actual wallet and pool balances for the primary sequence. Positive accrued VPI is potentially refundable, not a fixed payout: the close-time calculation determines the rebate under the lifetime clamp, and the planner waives any portion that free pool cash cannot fund. The close VPI delta reaches the full −5,460.874219 USDG lifetime clamp, but that is not the net cash payout: ActionRebateSettled paid +4,354.874219 USDG after the 1,106.000000 USDG close execution fee; the waived amount and realized carry were both 0.000000 USDG. The accounting defect is the timing: the exit snapshot distributes the collected VPI as LP value before that contingent close-time outcome is known. This report does not claim that LP entry and exit use different NAV formulas; the same terminal snapshot is used for both.

### Why this is a supported path

- Trader B commits and executes its own order through the Router. LP A requests redemption and permissionlessly calls the matured epoch path through the Router; B then closes and withdraws its free margin.
- The actors need no owner, guardian, publisher, compromised keeper, stolen key, reentrancy, or signature forgery. The test also executes B's order through a separate keeper; the coalition result changes only by the 0.20 USDG clearinghouse credit.
- Every price observation uses the repository's MockPyth test fixture at a fixed 1.0 mark. No price, confidence, or publish-time manipulation occurs.
- The run uses production Router, Engine, HousePool, MarginClearinghouse, and tranche-vault code from the pinned source. No production contract is changed. The protocol is not deployed on Mainnet, so these are test measurements, not losses observed in a deployed system.

### Preconditions

1. A and B coordinate through separate LP and trader accounts. At the primary point A begins with 3,000,000 USDG of Junior shares; B has 165,930 USDG in funded account value.
2. The pool has free cash to pay the close rebate. The primary sequence verifies that the VPI reserve, trader claim, and clearinghouse balance are zero after B fully closes and withdraws.
3. B opens a LONG at 39.5% of the 7,000,001 USDG pool depth: 2,765,000 USDG notional. It pays 5,460.874219 USDG of VPI.
4. TrancheVault enforces a one-hour post-deposit cooldown and HousePool uses a shared one-hour LP epoch. A's cooldown has elapsed; the PoC places its redemption request in B's open-position epoch, settles at the next boundary through the public Router path, then claims the funded request. It advances protocol time to satisfy the cooldown and epoch rather than bypassing them ([redemption request and cooldown](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631); [one-hour LP epoch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L178)). The same 1.0 mark is used through open, settlement, and close.
5. The test uses the checked-in launch values: vpiFactor 0.01e18, maxSkewRatio 0.4e18, baseCarryBps 500, and executionFeeBps 4. These values are listed in the [pinned deployment defaults](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L140).

These are capital and execution conditions, not reasons to reduce the measured technical impact.

### Attack path

1. Passive LP V and coalition LP A hold Junior shares in a pool with 7,000,001 USDG of assets. V's initial share value is 999,996.286141 USDG; A's is 2,999,988.858436 USDG.
2. B opens a 2,765,000 USDG LONG through the Router. The fixed-price fill transfers 5,460.874219 USDG of VPI into the pool.
3. A requests its matured redemption. At the next hourly epoch, a permissionless caller settles the pool while B's position is open. A claims 3,003,254.591487 USDG and has no remaining shares, pending request, refundable shares, or claimable assets.
4. At the same mark, B closes. The close VPI delta reaches the full accrued lifetime clamp (−5,460.874219 USDG); after the 1,106.000000 USDG close execution fee, the actual net action rebate transferred is +4,354.874219 USDG, with no waiver. Realized carry is 0.000000 USDG. B settles any trader claim, withdraws its free margin, and ends with no position, clearinghouse balance, trader claim, or VPI reserve.
5. In a subsequent epoch, V requests and completes a full redemption. Its actual claim is 998,892.095167 USDG, a realized change of −1,104.190974 USDG from its starting value.

## Impact Details
Amounts are in USDG. “V change” is the actual amount transferred by a completed requestRedeem → settlement → claimRedeem sequence, compared with V's starting share value. The final passive-LP claim is measured in a state snapshot after the position closes; that state is reverted after its returned assets and wallet transfer are checked. The earlier conservation checkpoint uses V's share value and is not presented as including that later claim. All coalition profit figures include protocol-level trading charges and carry but exclude external transaction gas costs. The primary and price-sensitivity runs measured 0.000000 USDG pending and realized carry at close.

| Scenario | Coalition realized net | Passive V actual full-exit change | VPI paid at open |
| --- | ---: | ---: | ---: |
| 39.5% attack | **+1,047.865777** | **−1,104.190974** | 5,460.874219 |
| 35% attack | +597.103209 | −815.809690 | 4,287.499387 |
| 25% attack | −100.639232 | −326.938469 | 2,187.499687 |
| 10% attack | −361.166443 | +13.325137 | 349.999950 |
| A stays in the pool; B trades | — | +212.317465 | 5,460.874219 |
| B closes before A's redemption is settled | — | +200.189681 | 5,460.874219 |
| Separate keeper executes the 39.5% attack | +1,047.665777 | −1,104.190974 | 5,460.874219 |

At 39.5% skew, A claims 3,003,254.591487 USDG. B's wallet moves from 165,930.000000 to 163,712.132726 USDG after the close and free-margin withdrawal, a net change of −2,217.867274 USDG. Their combined realized net is +1,047.865777 USDG. V starts at 999,996.286141 USDG and receives 998,892.095167 USDG. The test's attack-time tracked-balance conservation sum, including tranche holders, treasury, and keeper clearinghouse credit, is zero.

The fixed-midpoint result is not risk-free. In a separate Router replay with valid MockPyth timestamps and 1 bp confidence, changing the midpoint by +2 bp before close left the coalition +397.705905 USDG, while +5 bp produced −432.125895 USDG; linear interpolation places the break-even near +3.44 bp. External hedge depth and cost were not modeled.

The 25% case intentionally demonstrates that the coalition need not profit for the victim's principal to decrease: the coalition change is −100.639232 USDG while V changes by −326.938469 USDG. At 10% skew, the coalition changes by −361.166443 USDG and V changes by +13.325137 USDG from its starting value. These negative cases bound the profitable range and are not used to claim universal profitability.

**Six-cycle repeatability.** The test performs six cycles over 61,140 seconds and 17 hourly epochs. It excludes 177,149.084457 USDG of B's additional funding from the coalition result. After A's final full redemption and B's final margin withdrawal, A's wallet holds 3,019,555.328655 USDG and B's holds 163,836.352929 USDG. Their combined realized net after B's additional funding is **+6,253.738691 USDG**. V's starting value is 999,996.286141 USDG; its actual final claim is 993,367.630890 USDG, a **−6,628.655251 USDG** principal change. The preceding share-value checkpoint's tracked deltas sum to −1 USDG atom; the later V claim is a separate state-snapshot measurement.

**Entry-side mirror.** With an unrelated O LONG open, O has +3,120.499219 USDG of accrued VPI. B's opposing SHORT receives −3,118.066739 USDG and has a 3,118.066739 USDG nonwithdrawable reserve. A deposits 8,000,000 USDG and activates at the depressed Junior NAV before B closes. The measured 80% A-share path gives A +2,467.866114 USDG, B −1,267.285743 USDG, and a coalition net of **+1,200.580371 USDG**. V starts at 999,990.715363 USDG and receives 999,041.087071 USDG after O closes: **−949.628292 USDG**. In the matched B-idle control V receives 1,000,288.955415 USDG: +298.240052 USDG. Thus the attack/control difference in V's realized exit is −1,247.868344 USDG. The V exit is measured inside a state snapshot after O's ordinary close; the snapshot is reverted after measurement. This mirror is secondary and requires an 8,000,000 USDG deposit plus the unrelated O position.

At the 60% share point, the coalition realizes +583.617404 USDG and V's actual exit changes by −640.102445 USDG. At 40%, the coalition loses 33.351680 USDG while V's actual exit changes by −330.775663 USDG. Their matched B-idle V exits change by +295.385742 USDG and +292.331631 USDG, respectively. At 40%, the coalition change is −33.351680 USDG; this case is not described as profitable.

## Duplicate and known-issue analysis

The closest listed scan issue is finding #31, “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate.” Its stated root is reducing execution-time depth so the close curve produces a larger rebate than the unmanipulated curve. A's redemption also lowers pool depth here. A controlled differential in a disposable copy held only the depth passed to the close VPI calculation at the exact net depth observed just after B opened (7,005,461.874219 USDG); the actual redemption, Router call order, Pool cash and NAV settlement remained unchanged. At zero confidence and a fixed 1.0 mark, the normal-depth attack produced coalition +1,047.865777 USDG and V's actual exit −1,104.190974 USDG. Its close VPI delta reached −5,460.874219 USDG; after the close execution fee, the net action rebate paid was +4,354.874219 USDG. With the close VPI depth held at the post-open value, the close VPI delta was −5,456.617377 USDG against +5,460.874219 USDG accrued, so it remained within the lifetime clamp and below the accrued amount. The net action rebate was +4,350.617377 USDG, the coalition still realized +1,043.608935 USDG, and V's actual exit was −1,102.062553 USDG. The depth-dependent difference was only 4.256842 USDG of coalition profit; most of the principal transfer persists without the excessive rebate described in #31. The differential test isolates NAV-timing extraction from close-depth rebate amplification. The coalition remains profitable without relying on an excessive close rebate, establishing a distinct accounting mechanism from finding #31 while retaining the protocol's normal lifetime VPI clamp.

The listed SC Audit Studio finding SC-H1 concerns a deposit before senior yield is accrued and dilution of prior depositors' yield. SC-M3 concerns fresh negative VPI omitted from post-open initial-margin projection. Neither describes a passive Junior LP's principal being transferred through redemption pricing before an already-accrued VPI rebate is paid. The listed Cecuro report and the collaborative and Cantina reports were also reviewed; no exact match for this root was identified.

The strongest counterargument is explicit documentation. [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L871-L873) and [ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1064-L1066) say anticipated VPI and its reserve are excluded from terminal NAV. They describe the mechanism but do not state that an exiting LP can keep principal later paid as another trader's close rebate by the LPs who remain. The listed [VPI limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444) and [LP accounting limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1380-L1399) do not name that principal transfer.

[ACCOUNTING_SPEC.md:737-740](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L737-L740) describes collection-time ownership of accrued carry, including carry collected after a new LP's activation. This differs from VPI because carry represents collected revenue, whereas positive VPI remains potentially refundable at position close.

In the reproduced attack, Junior redemption is priced using pool assets that include collected VPI. The subsequent trader close pays a net action rebate of 4,354.874219 USDG after the 1,106.000000 USDG close execution fee, reducing the assets backing the remaining LPs. The documented VPI NAV policy does not explicitly address this coordinated redemption-and-close sequence or its demonstrated transfer of passive Junior LP principal.

The report does not rely on [ACCOUNTING_SPEC.md:1288](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1284-L1292) or [WHITEPAPER.md:1793](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1790-L1797) as violated guarantees. Those passages describe shared entry/exit NAV accounting, which the PoC also uses.

## Mitigation

A candidate fix adds a signed aggregate of VPI accrued over live positions, passes it into the HousePool input snapshot, and adjusts terminal NAV by that signed amount. Positive aggregate VPI is conservatively treated as potentially refundable value; it is not a promise that each position receives a rebate. Actual payment depends on the close-time VPI calculation and lifetime clamp, and any portion exceeding available free pool cash is waived. Negative aggregate VPI represents rebate cash already paid and recoverable from a dedicated reserve. The candidate patch does not change the close clamp, reserve operations, or cash payout path.

The signed sum is safe only while each negative position remains fully backed. The pinned open and close planners require the per-account reserve to equal or exceed the negative VPI target; [MarginClearinghouse](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L841-L905) protects that bucket from generic action charges and only transfers it to the pool through the close path. The patch test opens independent positions O and B with +3,120.499219 USDG and −3,118.066739 USDG accrued VPI. Their signed aggregate is +2.432480 USDG; the test checks that exact sum and verifies B's separate reserve equals 3,118.066739 USDG. It then checks the aggregate and reserve through B's partial close, full close, O's close, and snapshot restoration. Partial close leaves O's positive VPI intact; closing B returns the aggregate to O's balance; closing O returns it to zero. This tests that the signed accounting is backed by per-account reserve state rather than by an unbacked cross-account offset.

```diff
--- a/packages/perps/src/CfdEngine.sol
+++ b/packages/perps/src/CfdEngine.sol
@@ -114,6 +114,11 @@
     uint256[2] public sideCarryIndex;
     /// @notice Wall-clock Unix timestamp through which each side carry index (`0` LONG; `1` SHORT) has been advanced.
     uint64[2] public sideCarryTimestamp;
+    /// @notice Signed sum of `vpiAccrued` over all live positions, in 6-decimal settlement token units.
+    /// @dev Positive accrued VPI represents potentially refundable value on close and is conservatively excluded from
+    ///      distributable LP NAV; the close calculation and available cash determine any payment. Negative accrued
+    ///      VPI is backed by an account-specific reserve and recoverable on close.
+    int256 public outstandingVpiAccruedAssets;
 
     /// @notice Account carry checkpointed but not yet physically collected, in 6-decimal settlement token units.
     mapping(address => uint256) public unsettledCarryAssets;
@@ -1266,6 +1271,7 @@
         pos.lastCarryTimestamp = position.lastCarryTimestamp;
         pos.borrowBaseAssets = newBorrowBaseAssets;
         pos.lastCarryIndex = _currentSideCarryIndex(position.side, block.timestamp, _poolAssetsForCarry());
+        outstandingVpiAccruedAssets += position.vpiAccrued - pos.vpiAccrued;
         pos.vpiAccrued = position.vpiAccrued;
         _applySideBorrowBaseDelta(position.side, 0, newBorrowBaseAssets);
     }
@@ -1280,6 +1286,7 @@
         if (pos.lots > 0 || pos.borrowBaseAssets > 0) {
             _applySideBorrowBaseDelta(pos.side, pos.borrowBaseAssets, 0);
         }
+        outstandingVpiAccruedAssets -= pos.vpiAccrued;
         delete _positions[account];
     }
 

--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
@@ -68,6 +68,7 @@
             snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
         );
         snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+        snapshot.outstandingVpiAccruedAssets = engineContract.outstandingVpiAccruedAssets();
         snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;

--- a/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol
+++ b/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol
@@ -28,6 +28,7 @@
         uint256 maxLiabilityAssets;
         uint256 supplementalReservedAssets;
         int256 terminalLpPriceDeltaAssets;
+        int256 outstandingVpiAccruedAssets;
         uint64 terminalNavBookVersion;
         uint256 traderClaimBalanceAssets;
         bool hasOpenPositions;

--- a/packages/perps/src/libraries/HousePoolAccountingLib.sol
+++ b/packages/perps/src/libraries/HousePoolAccountingLib.sol
@@ -62,6 +62,15 @@
     ) internal pure returns (ReconcileSnapshot memory snapshot) {
         uint256 terminalAssets = engineSnapshot.physicalAssets;
         uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
+        int256 outstandingVpi = engineSnapshot.outstandingVpiAccruedAssets;
+        if (outstandingVpi > 0) {
+            // Positive accrued VPI represents potentially refundable value on close and is conservatively excluded
+            // from distributable LP NAV; actual payment depends on the close-time calculation.
+            terminalLiabilities += uint256(outstandingVpi);
+        } else if (outstandingVpi < 0) {
+            // Rebate already paid out is reclaimed on close from the dedicated nonwithdrawable VPI reserve.
+            terminalAssets += uint256(-(outstandingVpi + 1)) + 1;
+        }
         int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
         if (terminalDelta >= 0) {
             terminalAssets += uint256(terminalDelta);

```

## References
- Current [Immunefi Plether Scope](https://immunefi.com/audit-competition/plether/scope/) — impact 15, in-scope code, listed reports, known issues, and PoC requirements.
- Current [Immunefi Plether Information](https://immunefi.com/audit-competition/plether/information/) — competition target and deployment status.
- Current [Immunefi Plether Resources](https://immunefi.com/audit-competition/plether/resources/) — source repository pinned to the commit cited above.
- [Pinned HousePool reconciliation and settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L840).
- [Pinned HousePool terminal accounting](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L78).
- [Pinned position VPI storage](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L70-L74) and [position settlement updates](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L509-L520).
- [Pinned OrderRouter permissionless epoch entrypoint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484), [TrancheVault redemption request](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631), and [redemption claim](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1014-L1021).
- [Pinned negative-VPI reserve target](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L47-L55), [open and close reserve checks](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L343-L354), [partial/full-close reserve target](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L825-L835), and [reserve lock, release and consume lifecycle](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L841-L905), and [protected action-reserve floor](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L975-L985).
- [Pinned close lifetime clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118) and [free-cash action-rebate payment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L980).
- [Pinned initial release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L140).
- [Listed scan report, finding #31](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1748); [SC Audit Studio pre-audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf).
- [SECURITY.md VPI and LP accounting limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L865-L876), [VPI limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444), and [LP accounting limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1380-L1399).
- [ACCOUNTING_SPEC.md carry collection-time policy](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L731-L744), [anticipated VPI exclusion](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1061-L1067), and [shared entry/exit NAV rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1284-L1292).
- [WHITEPAPER.md deposit-epoch description](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1790-L1797).


## Proof of Concept

## Proof of Concept

### Reproduction

Frozen commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Save the code below as `packages/perps/test/perps/VpiNavTimingExtraction.t.sol` and run the command shown below. The inline PoC preserves all 24 tests and executable Solidity statements from the successfully tested original source. Only comments and formatting were removed to satisfy the submission character limit. The actual Foundry execution passed all 24 tests with exit code 0.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract VpiNavExitTest is BasePerpTest {

uint256 internal constant PX=1e8;
address internal constant A=address(0xA11CE1);
address internal constant B=address(0xB0B2);
address internal constant V=address(0xD1C71);
address internal constant SLP=address(0x5E410);
address internal constant KEEPER=address(0xC001);
uint256 internal constant V_DEP=1_000_000e6;
uint256 internal constant A_DEP=3_000_000e6;
uint256 internal constant S_DEP=2_000_000e6;
uint8 internal constant ATTACK=0;
uint8 internal constant NO_REDEEM=1;
uint8 internal constant CLOSE_FIRST=2;

function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
return CfdTypes.RiskParams({
vpiFactor: _vpiFactor(),
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

function _vpiFactor() internal pure virtual returns (uint256) {
return 0.01e18;
}

function _initialJuniorDeposit() internal pure override returns (uint256) {
return 0;
}

function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
return 1_000_000e6;
}

function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
return 1e6;
}

function setUp() public override {
super.setUp();
IOrderRouterAdminHost.RouterConfig memory rc=_routerConfig();
rc.minOpenNotionalAssets=1_000e6;
rc.adverseConfidenceMultiplierBps=2_500;
rc.maxPendingOrders=5;
rc.positionProtectionTriggerBountyAssets=200_000;
_setRouterConfig(rc);
_setSeniorCapacity(40_000_000e6,8_000);
vm.warp(SETUP_TIMESTAMP + 7 days);
vm.roll(vm.getBlockNumber() + 10);
_fundJunior(V,V_DEP);
_fundJunior(A,A_DEP);
_fundSenior(SLP,S_DEP);
}
uint256[] internal tickTime;
function _now() internal view returns (uint256) {
return vm.getBlockTimestamp();
}

function _publish() internal {
vm.roll(vm.getBlockNumber() + 1);
uint256 last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
if (_now()<=last) vm.warp(last + 1);
last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
uint64 head=router.nextExecuteId();
uint64 next=router.nextCommitId();
for (uint64 id=head; id!=0&&id<next; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
if (c>=last&&c + 1<_now()) tickTime.push(c + 1);
}
tickTime.push(_now());
}

function _writeTick(MockPyth.FeedUpdate[] memory u,uint256 off,uint256 i) internal view {
uint256 prev=i==0 ? 0 : tickTime[i - 1];
for (uint256 k; k<2; ++k) {
u[off + k]=MockPyth.FeedUpdate({
id: bytes32(k + 1),
price: MockPyth.MockPrice({
price: int64(int256(PX)),
conf: 0,
expo: -8,
publishTime: tickTime[i],
prevPublishTime: prev
})
});
}
}

function _live() internal view returns (bytes[] memory data) {
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2);
_writeTick(u,0,tickTime.length - 1);
data=new bytes[](1);
data[0]=abi.encode(u);
}
function _firstTickAfter(uint256 c) internal view returns (bool found,uint256 idx) {
for (uint256 i=tickTime.length; i>0; --i) {
if (tickTime[i - 1]<=c) break;
idx=i - 1;
found=true;
}
}
function _hist(uint64 maxId) internal view returns (bytes[] memory data) {
uint64 head=router.nextExecuteId();
uint256[] memory idxs=new uint256[](uint256(maxId - head + 2));
uint256 n;
uint256 lastIdx=type(uint256).max;
for (uint64 id=head; id<=maxId; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
(bool f,uint256 idx)=_firstTickAfter(c);
if (f&&idx!=lastIdx) {
idxs[n++]=idx;
lastIdx=idx;
}
}
idxs[n++]=tickTime.length - 1;
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2 * n);
for (uint256 j; j<n; ++j) {
_writeTick(u,2 * j,idxs[j]);
}
data=new bytes[](1);
data[0]=abi.encode(u);
}
uint256 internal nonce;
bool internal bExecutes=true;
function _req(CfdTypes.Side side,uint256 sizeDelta,uint256 margin,bool isClose)
internal
returns (OrderV3Types.OrderRequest memory r)
{
r.clientOrderId=bytes32(++nonce);
r.side=side;
r.sizeDelta=sizeDelta;
r.marginDelta=margin;
r.targetPrice=(side==CfdTypes.Side.LONG)==isClose ? type(uint256).max : 1;
r.isClose=isClose;
uint256 maxW=router.maxExecutionWindowSeconds();
r.bounds.submitBy=uint64(_now() + maxW);
r.bounds.executionWindowSeconds=uint32(maxW);
r.bounds.allowedExecutionModes=7;
r.bounds.expectedConfigHash=router.lifecycleBook().currentExecutionConfigHash();
r.bounds.maxExecutionBountyAssets=type(uint256).max;
r.bounds.maxExecutionNotionalAssets=type(uint256).max;
r.bounds.maxGrossAccountDebitAssets=type(uint256).max;
r.bounds.maxActionChargeAssets=type(uint256).max;
r.bounds.maxExplicitFeesAssets=type(uint256).max;
r.bounds.maxPostPositionSize=type(uint256).max;
r.bounds.maxPostLeverageBps=type(uint32).max;
}
function _commitExec(address u,OrderV3Types.OrderRequest memory r) internal returns (bool executed) {
_publish();
router.updateMarkPrice(_live());
vm.prank(u);
uint64 id=router.commitOrder(r);
_publish();
bytes[] memory data=_hist(id);
vm.prank(bExecutes ? B : KEEPER);
OrderV3Types.ExecutionResult memory res=router.executeOrder(id,data);
executed=res.status==OrderV3Types.LifecycleStatus.Executed;
if (!executed) {
console2.log("ORDER NOT EXECUTED status",uint256(res.status));
console2.log("  pendingReason",uint256(res.pendingReason));
console2.log("  terminalReason",uint256(res.terminalReason));
}
}
function _settleEpoch() internal {
_publish();
router.updateMarkPrice(_live());
router.settleLpEpoch(_live());
}
function _toNextEpoch(uint256 offset) internal {
uint256 t=_now();
vm.warp(t - (t % 1 hours) + 1 hours + offset);
}
struct Out {
uint256 aStart;
uint256 aEnd;
uint256 bStart;
uint256 bEnd;
uint256 vStart;
uint256 vEndConv;
uint256 vEndExit;
uint256 treasuryDelta;
uint256 poolTA;
uint256 vpiPaidOpen;
uint256 aClaimed;
uint256 aSharesLeft;
uint256 navPriceBeforeSettle;
uint256 vAfterSettle;
bool ok;
}
function _bValue() internal view returns (uint256) {
return settlementToken.balanceOf(B) + clearinghouse.balanceAssets(B) + engine.traderClaimBalanceAssets(B);
}
function _run(uint256 skewBps,uint8 mode) internal returns (Out memory o) {
_toNextEpoch(60);
_publish();
router.updateMarkPrice(_live());
uint256 depth=pool.totalAssets();
uint256 notional=(depth * skewBps) / 10_000;
notional=(notional / 100e6) * 100e6;
uint256 sizeDelta=notional * 1e12;
uint256 margin=(notional * 3) / 100 + 5e6;
_fundTrader(B,margin * 2 + 20e6);
uint256 treas0=clearinghouse.balanceAssets(engine.protocolTreasury());
o.aStart=juniorVault.convertToAssets(juniorVault.balanceOf(A));
o.vStart=juniorVault.convertToAssets(juniorVault.balanceOf(V));
o.bStart=_bValue();
uint256 aWallet0=settlementToken.balanceOf(A);
uint256 seedJ0=juniorVault.convertToAssets(juniorVault.balanceOf(juniorVault.seedReceiver()));
uint256 seedS0=seniorVault.convertToAssets(seniorVault.balanceOf(seniorVault.seedReceiver()));
uint256 slp0=seniorVault.convertToAssets(seniorVault.balanceOf(SLP));
uint256 keeper0=settlementToken.balanceOf(KEEPER);
uint256 keeperCH0=clearinghouse.balanceAssets(KEEPER);
console2.log("--- scenario: skewBps",skewBps);
console2.log("mode (0 attack, 1 no-redeem, 2 B-closes-first)",uint256(mode));
console2.log("pool depth totalAssets",depth);
console2.log("notional",notional);
bool opened=_commitExec(B,_req(CfdTypes.Side.LONG,sizeDelta,margin,false));
require(opened,"open did not execute");
{
(uint256 psize,uint256 pmargin,,,,,int256 vpiAcc)=engine.positions(B);
o.vpiPaidOpen=vpiAcc>0 ? uint256(vpiAcc) : 0;
console2.log("B position size(18d)",psize);
console2.log("B position margin",pmargin);
console2.log("B vpiAccrued after open (6d, + = charge)",vpiAcc);
}
console2.log("pool totalAssets after open",pool.totalAssets());
console2.log("junior totalAssets (live) after open",juniorVault.totalAssets());
console2.log("V convertToAssets after open",juniorVault.convertToAssets(juniorVault.balanceOf(V)));
uint256 reqId;
uint256 requestedShares;
if (mode==ATTACK) {
vm.warp(_now() + 30);
uint256 sh=juniorVault.maxRequestRedeem(A);
requestedShares=sh;
vm.prank(A);
reqId=juniorVault.requestRedeem(sh,A,A);
console2.log("A requested redeem shares",sh);
(,,,uint256 maxJw)=pool.getPendingTrancheState();
console2.log("junior max withdraw assets (live)",maxJw);
}
if (mode==CLOSE_FIRST) {
bool closedF=_commitExec(B,_req(CfdTypes.Side.LONG,sizeDelta,0,true));
require(closedF,"early close did not execute");
vm.warp(_now() + 30);
uint256 sh2=juniorVault.maxRequestRedeem(A);
requestedShares=sh2;
vm.prank(A);
reqId=juniorVault.requestRedeem(sh2,A,A);
}
o.navPriceBeforeSettle=juniorVault.convertToAssets(juniorVault.balanceOf(V));
_toNextEpoch(5);
if (mode!=NO_REDEEM) _settleEpoch();
o.vAfterSettle=juniorVault.convertToAssets(juniorVault.balanceOf(V));
console2.log("V convertToAssets right after settle",o.vAfterSettle);
if (mode!=NO_REDEEM) {
uint256 claimable=juniorVault.claimableRedeemRequest(reqId,A);
require(claimable==requestedShares,"A redemption was not fully funded");
uint256 aWalletBeforeClaim=settlementToken.balanceOf(A);
console2.log("A claimable redeem shares",claimable);
vm.prank(A);
o.aClaimed=juniorVault.claimRedeem(reqId,claimable,A,A);
require(o.aClaimed>0,"A redemption assets are zero");
require(settlementToken.balanceOf(A)==aWalletBeforeClaim + o.aClaimed,"A claim transfer mismatch");
require(juniorVault.balanceOf(A)==0,"A shares remain after full exit");
require(juniorVault.pendingRedeemRequest(reqId,A)==0,"A redeem shares remain pending");
require(juniorVault.refundableRedeemRequest(reqId,A)==0,"A redeem shares are refundable");
require(juniorVault.claimableRedeemAssets(reqId,A)==0,"A redeem assets remain claimable");
console2.log("A claimed assets",o.aClaimed);
}
console2.log("pool totalAssets after A claim",pool.totalAssets());
vm.warp(_now() + 20);
if (mode!=CLOSE_FIRST) {
bool closed=_commitExec(B,_req(CfdTypes.Side.LONG,sizeDelta,0,true));
console2.log("B close executed",closed);
require(closed,"close did not execute");
}
if (engine.traderClaimBalanceAssets(B)>0) {
vm.prank(B);
engine.settleTraderClaim(B);
}
uint256 free=clearinghouse.getAccountAssetBuckets(B).freeSettlementAssets;
uint256 bWalletBeforeMarginWithdrawal=settlementToken.balanceOf(B);
if (free>0) {
vm.prank(B);
clearinghouse.withdrawMargin(free);
}
require(
settlementToken.balanceOf(B)==bWalletBeforeMarginWithdrawal + free,
"B free margin was not transferred to wallet"
);
{
(uint256 psize2,,,,,,)=engine.positions(B);
require(psize2==0,"B not flat");
}
require(clearinghouse.balanceAssets(B)==0,"B clearinghouse balance remains");
require(engine.traderClaimBalanceAssets(B)==0,"B trader claim remains");
require(clearinghouse.vpiRebateReserveAssets(B)==0,"B VPI reserve remains");
console2.log("B USDG wallet after close and margin withdrawal",settlementToken.balanceOf(B));
console2.log("B clearinghouse assets after close and margin withdrawal",clearinghouse.balanceAssets(B));
console2.log("B trader claim after close",engine.traderClaimBalanceAssets(B));
console2.log("B VPI reserve after close",clearinghouse.vpiRebateReserveAssets(B));
console2.log("aggregate trader claims after close",engine.totalTraderClaimBalanceAssets());
console2.log("Pool raw USDG balance after close",settlementToken.balanceOf(address(pool)));
console2.log("Pool totalAssets after close",pool.totalAssets());
o.aSharesLeft=juniorVault.balanceOf(A);
o.aEnd=(settlementToken.balanceOf(A) - aWallet0) + juniorVault.convertToAssets(o.aSharesLeft);
o.bEnd=_bValue();
o.vEndConv=juniorVault.convertToAssets(juniorVault.balanceOf(V));
o.treasuryDelta=clearinghouse.balanceAssets(engine.protocolTreasury()) - treas0;
o.poolTA=pool.totalAssets();
o.ok=true;
console2.log("-- decomposition of deltas (end - start), USDG 6-dec --");
console2.log("delta A",int256(o.aEnd) - int256(o.aStart));
console2.log("delta B",int256(o.bEnd) - int256(o.bStart));
console2.log("delta V",int256(o.vEndConv) - int256(o.vStart));
int256 dSeedJ=int256(juniorVault.convertToAssets(juniorVault.balanceOf(juniorVault.seedReceiver()))) - int256(seedJ0);
int256 dSeedS=int256(seniorVault.convertToAssets(seniorVault.balanceOf(seniorVault.seedReceiver()))) - int256(seedS0);
int256 dSlp=int256(seniorVault.convertToAssets(seniorVault.balanceOf(SLP))) - int256(slp0);
int256 dKeeper=int256(settlementToken.balanceOf(KEEPER)) - int256(keeper0);
int256 dKeeperCH=int256(clearinghouse.balanceAssets(KEEPER)) - int256(keeperCH0);
console2.log("delta junior seed holder",dSeedJ);
console2.log("delta senior seed holder",dSeedS);
console2.log("delta senior LP (coupon recipient)",dSlp);
console2.log("delta external keeper wallet",dKeeper);
console2.log("delta external keeper clearinghouse",dKeeperCH);
console2.log("delta treasury (fees)",int256(o.treasuryDelta));
int256 conservationDelta =
(int256(o.aEnd) - int256(o.aStart)) + (int256(o.bEnd) - int256(o.bStart))
+ (int256(o.vEndConv) - int256(o.vStart)) + dSeedJ + dSeedS + dSlp + dKeeper + dKeeperCH
+ int256(o.treasuryDelta);
console2.log("sum of tracked deltas (including LP holders and recipients)",conservationDelta);
require(conservationDelta>=-1&&conservationDelta<=1,"tracked USDG deltas do not conserve within 1 atom");
uint256 snap=vm.snapshotState();
{
uint256 vs=juniorVault.maxRequestRedeem(V);
if (vs>0) {
vm.warp(_now() + 1 hours + 1);
}
uint256 vShares=juniorVault.balanceOf(V);
require(vs==vShares&&vShares>0,"passive LP full redeem unavailable");
vm.prank(V);
uint256 rid=juniorVault.requestRedeem(vShares,V,V);
_toNextEpoch(5);
_settleEpoch();
uint256 cl=juniorVault.claimableRedeemRequest(rid,V);
require(cl==vShares,"passive LP redeem was not fully funded");
uint256 vWalletBeforeClaim=settlementToken.balanceOf(V);
vm.prank(V);
o.vEndExit=juniorVault.claimRedeem(rid,cl,V,V);
require(o.vEndExit>0,"passive LP full-exit assets are zero");
require(settlementToken.balanceOf(V)==vWalletBeforeClaim + o.vEndExit,"V claim transfer mismatch");
console2.log("V USDG wallet after full redemption",settlementToken.balanceOf(V));
console2.log("V shares after full redemption",juniorVault.balanceOf(V));
console2.log("V claimable redeem assets after claim",juniorVault.claimableRedeemAssets(rid,V));
console2.log("Pool raw USDG after V full redemption",settlementToken.balanceOf(address(pool)));
console2.log("Pool totalAssets after V full redemption",pool.totalAssets());
require(juniorVault.balanceOf(V)==0,"passive LP shares remain after full exit");
require(juniorVault.pendingRedeemRequest(rid,V)==0,"passive LP shares remain pending");
require(juniorVault.refundableRedeemRequest(rid,V)==0,"passive LP shares are refundable");
require(juniorVault.claimableRedeemAssets(rid,V)==0,"passive LP redeem assets remain claimable");
}
vm.revertToState(snap);
}
function _report(Out memory o) internal pure {
console2.log("== RESULT (USDG 6-dec) ==");
console2.log("A start (convertToAssets of A shares)",o.aStart);
console2.log("A end   (claimed + remaining shares conv)",o.aEnd);
console2.log("B start",o.bStart);
console2.log("B end",o.bEnd);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
int256 aNet=int256(o.aEnd) - int256(o.aStart);
int256 bNet=int256(o.bEnd) - int256(o.bStart);
console2.log("A net",aNet);
console2.log("B net",bNet);
console2.log("COALITION NET",coalition);
console2.log("V start (convertToAssets)",o.vStart);
console2.log("V end   (convertToAssets)",o.vEndConv);
console2.log("V end   (realized full exit)",o.vEndExit);
console2.log("V delta (conv)",int256(o.vEndConv) - int256(o.vStart));
console2.log("V delta (realized exit)",int256(o.vEndExit) - int256(o.vStart));
console2.log("treasury delta",o.treasuryDelta);
console2.log("pool totalAssets end",o.poolTA);
console2.log("open VPI paid by B",o.vpiPaidOpen);
console2.log("A shares left",o.aSharesLeft);
}
function testExitCapture_Attack() public {
Out memory o=_run(3_950,ATTACK);
_report(o);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
assertGt(coalition,0,"primary coalition profit was not realized");
assertLt(o.vEndExit,o.vStart,"passive LP principal loss was not realized");
}
function testExitCapture_Attack_10pct() public {
Out memory o=_run(1_000,ATTACK);
_report(o);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
assertLt(coalition,0,"10% skew case should remain unprofitable");
assertGe(o.vEndExit,o.vStart,"10% skew case should not lose passive principal");
}
function testExitCapture_Attack_25pct() public {
Out memory o=_run(2_500,ATTACK);
_report(o);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
assertLt(coalition,0,"25% skew case should remain unprofitable");
assertLt(o.vEndExit,o.vStart,"25% skew case should still measure passive principal loss");
}
function testExitCapture_Attack_35pct() public {
Out memory o=_run(3_500,ATTACK);
_report(o);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
assertGt(coalition,0,"35% coalition profit was not realized");
assertLt(o.vEndExit,o.vStart,"35% passive LP principal loss was not realized");
}
function testExitCapture_Control() public {
Out memory o=_run(3_950,NO_REDEEM);
_report(o);
assertGe(o.vEndExit,o.vStart,"no-redemption control should not lose passive principal");
}
function testExitCapture_Control_CloseFirst() public {
Out memory o=_run(3_950,CLOSE_FIRST);
_report(o);
assertGe(o.vEndExit,o.vStart,"close-first control should not lose passive principal");
}
function testExitCapture_Attack_KeeperExecutes() public {
bExecutes=false;
Out memory o=_run(3_950,ATTACK);
_report(o);
int256 coalition=int256(o.aEnd + o.bEnd) - int256(o.aStart + o.bStart);
assertGt(coalition,0,"keeper-executed coalition profit was not realized");
assertLt(o.vEndExit,o.vStart,"keeper-executed path did not harm passive principal");
}
}
contract VpiNavRepeatabilityTest is BasePerpTest {
uint256 internal constant PX=1e8;
address internal constant A=address(0xA11CE1);
address internal constant B=address(0xB0B2);
address internal constant V=address(0xD1C71);
address internal constant SLP=address(0x5E410);
address internal constant KEEPER=address(0xC001);
uint256 internal constant V_DEP=1_000_000e6;
uint256 internal constant A_DEP=3_000_000e6;
uint256 internal constant S_DEP=2_000_000e6;
uint8 internal constant ATTACK=0;
uint8 internal constant NO_REDEEM=1;
uint8 internal constant CLOSE_FIRST=2;
function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
return CfdTypes.RiskParams({
vpiFactor: _vpiFactor(),
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
function _vpiFactor() internal pure virtual returns (uint256) {
return 0.01e18;
}
function _initialJuniorDeposit() internal pure override returns (uint256) {
return 0;
}
function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
return 1_000_000e6;
}
function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
return 1e6;
}
function setUp() public override {
super.setUp();
IOrderRouterAdminHost.RouterConfig memory rc=_routerConfig();
rc.minOpenNotionalAssets=1_000e6;
rc.adverseConfidenceMultiplierBps=2_500;
rc.maxPendingOrders=5;
rc.positionProtectionTriggerBountyAssets=200_000;
_setRouterConfig(rc);
_setSeniorCapacity(40_000_000e6,8_000);
vm.warp(SETUP_TIMESTAMP + 7 days);
vm.roll(vm.getBlockNumber() + 10);
_fundJunior(V,V_DEP);
_fundJunior(A,A_DEP);
_fundSenior(SLP,S_DEP);
}
uint256[] internal tickTime;
function _now() internal view returns (uint256) {
return vm.getBlockTimestamp();
}
function _publish() internal {
vm.roll(vm.getBlockNumber() + 1);
uint256 last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
if (_now()<=last) vm.warp(last + 1);
last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
uint64 head=router.nextExecuteId();
uint64 next=router.nextCommitId();
for (uint64 id=head; id!=0&&id<next; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
if (c>=last&&c + 1<_now()) tickTime.push(c + 1);
}
tickTime.push(_now());
}
function _writeTick(MockPyth.FeedUpdate[] memory u,uint256 off,uint256 i) internal view {
uint256 prev=i==0 ? 0 : tickTime[i - 1];
for (uint256 k; k<2; ++k) {
u[off + k]=MockPyth.FeedUpdate({
id: bytes32(k + 1),
price: MockPyth.MockPrice({
price: int64(int256(PX)),
conf: 0,
expo: -8,
publishTime: tickTime[i],
prevPublishTime: prev
})
});
}
}
function _live() internal view returns (bytes[] memory data) {
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2);
_writeTick(u,0,tickTime.length - 1);
data=new bytes[](1);
data[0]=abi.encode(u);
}
function _firstTickAfter(uint256 c) internal view returns (bool found,uint256 idx) {
for (uint256 i=tickTime.length; i>0; --i) {
if (tickTime[i - 1]<=c) break;
idx=i - 1;
found=true;
}
}
function _hist(uint64 maxId) internal view returns (bytes[] memory data) {
uint64 head=router.nextExecuteId();
uint256[] memory idxs=new uint256[](uint256(maxId - head + 2));
uint256 n;
uint256 lastIdx=type(uint256).max;
for (uint64 id=head; id<=maxId; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
(bool f,uint256 idx)=_firstTickAfter(c);
if (f&&idx!=lastIdx) {
idxs[n++]=idx;
lastIdx=idx;
}
}
idxs[n++]=tickTime.length - 1;
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2 * n);
for (uint256 j; j<n; ++j) {
_writeTick(u,2 * j,idxs[j]);
}
data=new bytes[](1);
data[0]=abi.encode(u);
}
uint256 internal nonce;
bool internal bExecutes=true;
function _req(CfdTypes.Side side,uint256 sizeDelta,uint256 margin,bool isClose)
internal
returns (OrderV3Types.OrderRequest memory r)
{
r.clientOrderId=bytes32(++nonce);
r.side=side;
r.sizeDelta=sizeDelta;
r.marginDelta=margin;
r.targetPrice=(side==CfdTypes.Side.LONG)==isClose ? type(uint256).max : 1;
r.isClose=isClose;
uint256 maxW=router.maxExecutionWindowSeconds();
r.bounds.submitBy=uint64(_now() + maxW);
r.bounds.executionWindowSeconds=uint32(maxW);
r.bounds.allowedExecutionModes=7;
r.bounds.expectedConfigHash=router.lifecycleBook().currentExecutionConfigHash();
r.bounds.maxExecutionBountyAssets=type(uint256).max;
r.bounds.maxExecutionNotionalAssets=type(uint256).max;
r.bounds.maxGrossAccountDebitAssets=type(uint256).max;
r.bounds.maxActionChargeAssets=type(uint256).max;
r.bounds.maxExplicitFeesAssets=type(uint256).max;
r.bounds.maxPostPositionSize=type(uint256).max;
r.bounds.maxPostLeverageBps=type(uint32).max;
}
function _commitExec(address u,OrderV3Types.OrderRequest memory r) internal returns (bool executed) {
_publish();
router.updateMarkPrice(_live());
vm.prank(u);
uint64 id=router.commitOrder(r);
_publish();
bytes[] memory data=_hist(id);
vm.prank(bExecutes ? B : KEEPER);
OrderV3Types.ExecutionResult memory res=router.executeOrder(id,data);
executed=res.status==OrderV3Types.LifecycleStatus.Executed;
if (!executed) {
console2.log("ORDER NOT EXECUTED status",uint256(res.status));
console2.log("  pendingReason",uint256(res.pendingReason));
console2.log("  terminalReason",uint256(res.terminalReason));
}
}
function _settleEpoch() internal {
_publish();
router.updateMarkPrice(_live());
router.settleLpEpoch(_live());
}
function _toNextEpoch(uint256 offset) internal {
uint256 t=_now();
vm.warp(t - (t % 1 hours) + 1 hours + offset);
}
uint256 internal aReqId;
uint256 internal bInjected;
struct Snap {
uint256 t;
uint256 aVal;
int256 bVal;
uint256 v;
uint256 seedJ;
uint256 seedS;
uint256 slp;
uint256 keeper;
uint256 treas;
uint256 poolTA;
}
struct Times {
uint256 open;
uint256 redeemSettle;
uint256 close;
uint256 depositClaim;
uint256 waitedForCooldown;
}
function _bValue() internal view returns (uint256) {
return settlementToken.balanceOf(B) + clearinghouse.balanceAssets(B) + engine.traderClaimBalanceAssets(B);
}
function _snap() internal view returns (Snap memory s) {
s.t=_now();
uint256 escrow;
if (aReqId!=0) {
escrow=juniorVault.pendingRedeemRequest(aReqId,A) + juniorVault.refundableRedeemRequest(aReqId,A);
}
s.aVal=settlementToken.balanceOf(A) + juniorVault.convertToAssets(juniorVault.balanceOf(A))
+ juniorVault.convertToAssets(escrow);
s.bVal=int256(_bValue()) - int256(bInjected);
s.v=juniorVault.convertToAssets(juniorVault.balanceOf(V));
s.seedJ=juniorVault.convertToAssets(juniorVault.balanceOf(juniorVault.seedReceiver()));
s.seedS=seniorVault.convertToAssets(seniorVault.balanceOf(seniorVault.seedReceiver()));
s.slp=seniorVault.convertToAssets(seniorVault.balanceOf(SLP));
s.keeper=settlementToken.balanceOf(KEEPER);
s.treas=clearinghouse.balanceAssets(engine.protocolTreasury());
s.poolTA=pool.totalAssets();
}
function _d(uint256 x,uint256 y) internal pure returns (int256) {
return int256(x) - int256(y);
}
function _coal(Snap memory s) internal pure returns (int256) {
return int256(s.aVal) + s.bVal;
}
function _cycle(uint256 k,uint256 skewBps,bool aRedeems,bool aRedeposits,bool bTrades)
internal
returns (Times memory tm)
{
if (k==0) {
} else if (!aRedeems) {
_toNextEpoch(60);
} else {
uint256 cd=juniorVault.lastDepositTime(A) + 1 hours;
if (_now()<cd) {
tm.waitedForCooldown=cd + 1 - _now();
vm.warp(cd + 1);
}
if (_now() % 1 hours>40 minutes) _toNextEpoch(60);
}
_publish();
router.updateMarkPrice(_live());
uint256 notional=(pool.totalAssets() * skewBps) / 10_000;
notional=(notional / 100e6) * 100e6;
uint256 sizeDelta=notional * 1e12;
uint256 margin=(notional * 3) / 100 + 5e6;
if (bTrades) {
uint256 need=margin * 2 + 20e6;
uint256 free=clearinghouse.getAccountAssetBuckets(B).freeSettlementAssets;
if (free<need) {
_fundTrader(B,need - free);
bInjected += need - free;
}
}
tm.open=_now();
if (bTrades) require(_commitExec(B,_req(CfdTypes.Side.LONG,sizeDelta,margin,false)),"open failed");
uint256 reqId;
uint256 requestedShares;
if (aRedeems) {
vm.warp(_now() + 30);
uint256 sh=juniorVault.maxRequestRedeem(A);
requestedShares=sh;
vm.prank(A);
reqId=juniorVault.requestRedeem(sh,A,A);
aReqId=reqId;
console2.log("  A requested redeem shares",sh);
}
_toNextEpoch(5);
if (aRedeems) _settleEpoch();
tm.redeemSettle=_now();
if (aRedeems) {
uint256 claimable=juniorVault.claimableRedeemRequest(reqId,A);
require(claimable>0&&claimable<=requestedShares,"A redemption claimable shares are invalid");
if (aRedeposits) require(claimable==requestedShares,"six-cycle A redemption was not fully funded");
uint256 aWalletBeforeClaim=settlementToken.balanceOf(A);
vm.prank(A);
uint256 got=juniorVault.claimRedeem(reqId,claimable,A,A);
require(got>0,"A redemption assets are zero");
require(settlementToken.balanceOf(A)==aWalletBeforeClaim + got,"A claim transfer mismatch");
require(juniorVault.claimableRedeemAssets(reqId,A)==0,"A redeem assets remain claimable");
if (aRedeposits) {
require(juniorVault.balanceOf(A)==0,"A shares remain after repeatable cycle redemption");
require(juniorVault.pendingRedeemRequest(reqId,A)==0,"six-cycle A redeem shares remain pending");
require(juniorVault.refundableRedeemRequest(reqId,A)==0,"six-cycle A redeem shares are refundable");
}
console2.log("  A claimable redeem SHARES",claimable);
console2.log("  A claimed ASSETS",got);
console2.log("  A redeem shares still pending (escrow)",juniorVault.pendingRedeemRequest(reqId,A));
console2.log("  A redeem refundable shares",juniorVault.refundableRedeemRequest(reqId,A));
}
vm.warp(_now() + 20);
if (bTrades) {
require(_commitExec(B,_req(CfdTypes.Side.LONG,sizeDelta,0,true)),"close failed");
if (engine.traderClaimBalanceAssets(B)>0) {
vm.prank(B);
engine.settleTraderClaim(B);
}
(uint256 psize,,,,,,)=engine.positions(B);
require(psize==0,"B not flat");
}
tm.close=_now();
if (aRedeems&&aRedeposits) {
uint256 amt=settlementToken.balanceOf(A);
vm.startPrank(A);
settlementToken.approve(address(juniorVault),amt);
uint256 did=juniorVault.requestDeposit(amt,A,A);
vm.stopPrank();
uint256 act=juniorVault.depositEpochStart(did);
if (_now()<act) vm.warp(act);
_publish();
router.updateMarkPrice(_live());
_settleEpoch();
uint256 cl=juniorVault.claimableDepositRequest(did,A);
require(cl==amt,"deposit not settled");
vm.prank(A);
juniorVault.claimDeposit(did,cl,A,A);
tm.depositClaim=_now();
}
}
function _logCycle(uint256 k,Snap memory s0,Snap memory sp,Snap memory sn,Times memory tm) internal pure {
console2.log("====== cycle index",k);
console2.log("  t_open (unix)",tm.open);
console2.log("  t_redeemSettle",tm.redeemSettle);
console2.log("  t_close",tm.close);
console2.log("  t_depositClaim",tm.depositClaim);
console2.log("  A seconds waited for cooldown before cycle",tm.waitedForCooldown);
console2.log("  coalition net this cycle",_coal(sn) - _coal(sp));
console2.log("  coalition net CUMULATIVE",_coal(sn) - _coal(s0));
console2.log("  A delta cumulative",_d(sn.aVal,s0.aVal));
console2.log("  B delta cumulative",sn.bVal - s0.bVal);
console2.log("  V delta this cycle (convertToAssets)",_d(sn.v,sp.v));
console2.log("  V delta CUMULATIVE",_d(sn.v,s0.v));
console2.log("  V value now",sn.v);
console2.log("  seed holder delta this cycle",_d(sn.seedJ,sp.seedJ));
console2.log("  seed holder delta CUMULATIVE",_d(sn.seedJ,s0.seedJ));
console2.log("  senior seed delta CUMULATIVE",_d(sn.seedS,s0.seedS));
console2.log("  senior LP delta CUMULATIVE",_d(sn.slp,s0.slp));
console2.log("  treasury delta CUMULATIVE",_d(sn.treas,s0.treas));
console2.log("  keeper delta CUMULATIVE",_d(sn.keeper,s0.keeper));
console2.log("  pool totalAssets",sn.poolTA);
}
function _vRealizedExit() internal returns (uint256 got) {
uint256 snap=vm.snapshotState();
uint256 vShares=juniorVault.balanceOf(V);
uint256 maxShares=juniorVault.maxRequestRedeem(V);
require(maxShares==vShares&&vShares>0,"passive LP full redeem unavailable");
vm.prank(V);
uint256 rid=juniorVault.requestRedeem(vShares,V,V);
_toNextEpoch(5);
_settleEpoch();
uint256 cl=juniorVault.claimableRedeemRequest(rid,V);
require(cl==vShares,"passive LP redeem was not fully funded");
require(cl==vShares,"passive LP full redemption is not funded");
uint256 vWalletBeforeClaim=settlementToken.balanceOf(V);
vm.prank(V);
got=juniorVault.claimRedeem(rid,cl,V,V);
require(got>0,"passive LP exit assets are zero");
require(settlementToken.balanceOf(V)==vWalletBeforeClaim + got,"V claim transfer mismatch");
require(juniorVault.balanceOf(V)==0,"passive LP shares remain after full exit");
require(juniorVault.pendingRedeemRequest(rid,V)==0,"passive LP shares remain pending");
require(juniorVault.refundableRedeemRequest(rid,V)==0,"passive LP shares are refundable");
require(juniorVault.claimableRedeemAssets(rid,V)==0,"passive LP redeem assets remain claimable");
vm.revertToState(snap);
}
function _runN(uint256 n,uint256 skewBps,bool aRedeems,bool aRedeposits,bool bTrades,uint256 gapHours)
internal
returns (Snap memory s0,Snap memory sn,uint256 tFirstOpen,uint256 tLastEnd)
{
_toNextEpoch(60);
_publish();
router.updateMarkPrice(_live());
s0=_snap();
Snap memory sp=s0;
for (uint256 k; k<n; ++k) {
Times memory tm=_cycle(k,skewBps,aRedeems,aRedeposits,bTrades);
if (k==0) tFirstOpen=tm.open;
sn=_snap();
_logCycle(k + 1,s0,sp,sn,tm);
tLastEnd=aRedeposits&&aRedeems ? tm.depositClaim : tm.close;
sp=sn;
if (!aRedeems&&gapHours>0) vm.warp(_now() + gapHours * 1 hours);
}
}
function _summary(Snap memory s0,Snap memory sn,uint256 n,uint256 tFirstOpen,uint256 tLastEnd) internal view {
console2.log("######## SUMMARY after cycles",n);
console2.log("elapsed seconds first open -> last end",tLastEnd - tFirstOpen);
console2.log("elapsed epochs (hourly) first open -> last end",(tLastEnd - tFirstOpen + 3599) / 3600);
console2.log("average seconds per cycle (floor)",(tLastEnd - tFirstOpen) / n);
console2.log("start A value",s0.aVal);
console2.log("start B value (signed)",s0.bVal);
console2.log("start V value",s0.v);
console2.log("start seed holder value",s0.seedJ);
console2.log("start pool totalAssets",s0.poolTA);
console2.log("COALITION NET cumulative",_coal(sn) - _coal(s0));
console2.log("V delta cumulative (convertToAssets)",_d(sn.v,s0.v));
console2.log("seed holder delta cumulative",_d(sn.seedJ,s0.seedJ));
console2.log("bInjected (B top-ups, excluded)",bInjected);
int256 conservationDelta =
_coal(sn) - _coal(s0) + _d(sn.v,s0.v) + _d(sn.seedJ,s0.seedJ) + _d(sn.seedS,s0.seedS)
+ _d(sn.slp,s0.slp) + _d(sn.keeper,s0.keeper) + _d(sn.treas,s0.treas);
console2.log("sum tracked deltas at share-value checkpoint",conservationDelta);
require(conservationDelta>=-1&&conservationDelta<=1,"six-cycle USDG deltas do not conserve within 1 atom");
}
function testSixCycles_ShareValue() public {
uint256 n=6;
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(n,3_950,true,true,true,0);
_summary(s0,sn,n,t0,t1);
uint256 vExit=_vRealizedExit();
console2.log("V realized full exit (assets)",vExit);
console2.log("V realized exit delta vs start",_d(vExit,s0.v));
assertGt(int256(s0.v) - int256(sn.v),0,"V cumulative loss (convertToAssets) must be > 0");
assertGt(_coal(sn) - _coal(s0),0,"coalition cumulative must be > 0");
}
function testLargerCoalitionShare() public {
_fundJunior(A,5_000_000e6);
console2.log("A shares value before",juniorVault.convertToAssets(juniorVault.balanceOf(A)));
console2.log("junior totalAssets before",juniorVault.totalAssets());
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(1,3_950,true,false,true,0);
_summary(s0,sn,1,t0,t1);
uint256 vExit=_vRealizedExit();
console2.log("V realized full exit (assets)",vExit);
console2.log("V realized exit delta vs start",_d(vExit,s0.v));
}
function testSingleCycleBaseline() public {
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(1,3_950,true,false,true,0);
_summary(s0,sn,1,t0,t1);
console2.log("V realized exit delta vs start",_d(_vRealizedExit(),s0.v));
}
function testControl_NoRedeem() public {
uint256 n=6;
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(n,3_950,false,false,true,2);
_summary(s0,sn,n,t0,t1);
uint256 vExit=_vRealizedExit();
console2.log("V realized exit delta vs start",_d(vExit,s0.v));
assertGe(vExit,s0.v,"control: realized V principal must not decrease");
assertGe(sn.v,s0.v,"control: V share value must not lose");
assertGe(sn.seedJ,s0.seedJ,"control: seed holder must not lose");
}
function testControl_LPChurnOnly() public {
uint256 n=6;
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(n,3_950,true,true,false,0);
_summary(s0,sn,n,t0,t1);
}
function testSixCycles_RealizedFinalExit() public {
uint256 n=6;
(Snap memory s0,Snap memory sn,uint256 t0,uint256 t1)=_runN(n,3_950,true,true,true,0);
_summary(s0,sn,n,t0,t1);
uint256 eligible=juniorVault.lastDepositTime(A) + 1 hours + 1;
if (_now()<eligible) vm.warp(eligible);
uint256 aShares=juniorVault.balanceOf(A);
require(aShares>0,"coalition LP has no shares to redeem");
vm.prank(A);
uint256 aRequest=juniorVault.requestRedeem(aShares,A,A);
_toNextEpoch(5);
_settleEpoch();
uint256 aClaimable=juniorVault.claimableRedeemRequest(aRequest,A);
require(aClaimable==aShares,"coalition full redemption was not funded");
uint256 aWalletBeforeClaim=settlementToken.balanceOf(A);
vm.prank(A);
uint256 aClaimed=juniorVault.claimRedeem(aRequest,aClaimable,A,A);
require(aClaimed>0,"coalition redemption assets are zero");
require(settlementToken.balanceOf(A)==aWalletBeforeClaim + aClaimed,"A final claim transfer mismatch");
require(juniorVault.balanceOf(A)==0,"coalition LP shares remain");
require(juniorVault.pendingRedeemRequest(aRequest,A)==0,"coalition redeem shares remain pending");
require(juniorVault.refundableRedeemRequest(aRequest,A)==0,"coalition redeem shares are refundable");
require(juniorVault.claimableRedeemAssets(aRequest,A)==0,"coalition redeem assets remain claimable");
(uint256 bPosition,,,,,,)=engine.positions(B);
require(bPosition==0,"coalition trader is not flat");
uint256 bClaim=engine.traderClaimBalanceAssets(B);
if (bClaim>0) {
vm.prank(B);
engine.settleTraderClaim(B);
}
uint256 bFree=clearinghouse.getAccountAssetBuckets(B).freeSettlementAssets;
uint256 bWalletBeforeMarginWithdrawal=settlementToken.balanceOf(B);
if (bFree>0) {
vm.prank(B);
clearinghouse.withdrawMargin(bFree);
}
require(
settlementToken.balanceOf(B)==bWalletBeforeMarginWithdrawal + bFree,
"B final free margin was not transferred to wallet"
);
require(clearinghouse.balanceAssets(B)==0,"B clearinghouse balance remains after withdrawal");
require(engine.traderClaimBalanceAssets(B)==0,"B trader claim remains after settlement");
require(clearinghouse.vpiRebateReserveAssets(B)==0,"B VPI reserve remains after full close");
uint256 aWalletEnd=settlementToken.balanceOf(A);
uint256 bWalletEnd=settlementToken.balanceOf(B);
int256 realizedCoalitionDelta=int256(aWalletEnd + bWalletEnd) - int256(s0.aVal + bInjected);
uint256 vRealizedExit=_vRealizedExit();
int256 vRealizedDelta=_d(vRealizedExit,s0.v);
(uint256 bPositionAfter,,,,,,)=engine.positions(B);
console2.log("A final redeem assets",aClaimed);
console2.log("A realized wallet balance",aWalletEnd);
console2.log("B realized wallet balance after margin withdrawal",bWalletEnd);
console2.log("B external USDG capital supplied",bInjected);
console2.log("coalition realized delta after six cycles",realizedCoalitionDelta);
console2.log("passive LP realized full-exit assets",vRealizedExit);
console2.log("passive LP realized delta",vRealizedDelta);
console2.log("pool token balance",settlementToken.balanceOf(address(pool)));
console2.log("pool totalAssets",pool.totalAssets());
console2.log("aggregate trader claims",engine.totalTraderClaimBalanceAssets());
console2.log("B VPI rebate reserve",clearinghouse.vpiRebateReserveAssets(B));
console2.log("B remaining position size",bPositionAfter);
console2.log("A remaining shares",juniorVault.balanceOf(A));
console2.log("A pending redeem shares",juniorVault.pendingRedeemRequest(aRequest,A));
console2.log("treasury balance delta",int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(s0.treas));
assertGt(realizedCoalitionDelta,0,"six-cycle coalition profit must be realized in USDG");
assertLt(vRealizedExit,s0.v,"passive LP must realize a principal loss");
assertEq(engine.totalTraderClaimBalanceAssets(),0,"trader claims remain after all closes");
assertEq(clearinghouse.balanceAssets(B),0,"B clearinghouse balance remains after margin withdrawal");
assertEq(clearinghouse.vpiRebateReserveAssets(B),0,"VPI reserve remains after full close");
assertEq(bPositionAfter,0,"coalition trader position remains");
assertEq(juniorVault.balanceOf(A),0,"coalition shares remain after full exit");
assertEq(juniorVault.pendingRedeemRequest(aRequest,A),0,"coalition redemption remains pending");
}
}
contract VpiNavEntryTest is BasePerpTest {
uint256 internal constant PX=1e8;
address internal constant A=address(0xA11CE1);
address internal constant B=address(0xB0B2);
address internal constant V=address(0xD1C71);
address internal constant SLP=address(0x5E410);
address internal constant KEEPER=address(0xC001);
address internal constant O=address(0x0E6A41C);
uint256 internal constant V_DEP=1_000_000e6;
uint256 internal constant A_DEP=3_000_000e6;
uint256 internal constant S_DEP=2_000_000e6;
uint8 internal constant ATTACK=0;
uint8 internal constant NO_REDEEM=1;
uint8 internal constant CLOSE_FIRST=2;
function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
return CfdTypes.RiskParams({
vpiFactor: _vpiFactor(),
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
function _vpiFactor() internal pure virtual returns (uint256) {
return 0.01e18;
}
function _initialJuniorDeposit() internal pure override returns (uint256) {
return 0;
}
function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
return 1_000_000e6;
}
function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
return 1e6;
}
function setUp() public override {
super.setUp();
IOrderRouterAdminHost.RouterConfig memory rc=_routerConfig();
rc.minOpenNotionalAssets=1_000e6;
rc.adverseConfidenceMultiplierBps=2_500;
rc.maxPendingOrders=5;
rc.positionProtectionTriggerBountyAssets=200_000;
_setRouterConfig(rc);
_setSeniorCapacity(40_000_000e6,8_000);
vm.warp(SETUP_TIMESTAMP + 7 days);
vm.roll(vm.getBlockNumber() + 10);
_fundJunior(V,V_DEP);
_fundSenior(SLP,S_DEP);
}
uint256[] internal tickTime;
function _now() internal view returns (uint256) {
return vm.getBlockTimestamp();
}
function _publish() internal {
vm.roll(vm.getBlockNumber() + 1);
uint256 last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
if (_now()<=last) vm.warp(last + 1);
last=tickTime.length==0 ? 0 : tickTime[tickTime.length - 1];
uint64 head=router.nextExecuteId();
uint64 next=router.nextCommitId();
for (uint64 id=head; id!=0&&id<next; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
if (c>=last&&c + 1<_now()) tickTime.push(c + 1);
}
tickTime.push(_now());
}
function _writeTick(MockPyth.FeedUpdate[] memory u,uint256 off,uint256 i) internal view {
uint256 prev=i==0 ? 0 : tickTime[i - 1];
for (uint256 k; k<2; ++k) {
u[off + k]=MockPyth.FeedUpdate({
id: bytes32(k + 1),
price: MockPyth.MockPrice({
price: int64(int256(PX)),
conf: 0,
expo: -8,
publishTime: tickTime[i],
prevPublishTime: prev
})
});
}
}
function _live() internal view returns (bytes[] memory data) {
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2);
_writeTick(u,0,tickTime.length - 1);
data=new bytes[](1);
data[0]=abi.encode(u);
}
function _firstTickAfter(uint256 c) internal view returns (bool found,uint256 idx) {
for (uint256 i=tickTime.length; i>0; --i) {
if (tickTime[i - 1]<=c) break;
idx=i - 1;
found=true;
}
}
function _hist(uint64 maxId) internal view returns (bytes[] memory data) {
uint64 head=router.nextExecuteId();
uint256[] memory idxs=new uint256[](uint256(maxId - head + 2));
uint256 n;
uint256 lastIdx=type(uint256).max;
for (uint64 id=head; id<=maxId; ++id) {
if (router.lifecycleBook().pendingIntent(id).account==address(0)) continue;
uint256 c=router.lifecycleBook().orderTiming(id).commitTimestamp;
(bool f,uint256 idx)=_firstTickAfter(c);
if (f&&idx!=lastIdx) {
idxs[n++]=idx;
lastIdx=idx;
}
}
idxs[n++]=tickTime.length - 1;
MockPyth.FeedUpdate[] memory u=new MockPyth.FeedUpdate[](2 * n);
for (uint256 j; j<n; ++j) {
_writeTick(u,2 * j,idxs[j]);
}
data=new bytes[](1);
data[0]=abi.encode(u);
}
uint256 internal nonce;
bool internal bExecutes=true;
function _req(CfdTypes.Side side,uint256 sizeDelta,uint256 margin,bool isClose)
internal
returns (OrderV3Types.OrderRequest memory r)
{
r.clientOrderId=bytes32(++nonce);
r.side=side;
r.sizeDelta=sizeDelta;
r.marginDelta=margin;
r.targetPrice=(side==CfdTypes.Side.LONG)==isClose ? type(uint256).max : 1;
r.isClose=isClose;
uint256 maxW=router.maxExecutionWindowSeconds();
r.bounds.submitBy=uint64(_now() + maxW);
r.bounds.executionWindowSeconds=uint32(maxW);
r.bounds.allowedExecutionModes=7;
r.bounds.expectedConfigHash=router.lifecycleBook().currentExecutionConfigHash();
r.bounds.maxExecutionBountyAssets=type(uint256).max;
r.bounds.maxExecutionNotionalAssets=type(uint256).max;
r.bounds.maxGrossAccountDebitAssets=type(uint256).max;
r.bounds.maxActionChargeAssets=type(uint256).max;
r.bounds.maxExplicitFeesAssets=type(uint256).max;
r.bounds.maxPostPositionSize=type(uint256).max;
r.bounds.maxPostLeverageBps=type(uint32).max;
}
function _commitExec(address u,OrderV3Types.OrderRequest memory r) internal returns (bool executed) {
return _commitExecBy(u,r,bExecutes ? B : KEEPER);
}
function _commitExecBy(address u,OrderV3Types.OrderRequest memory r,address exec) internal returns (bool executed) {
_publish();
router.updateMarkPrice(_live());
vm.prank(u);
uint64 id=router.commitOrder(r);
_publish();
bytes[] memory data=_hist(id);
vm.prank(exec);
OrderV3Types.ExecutionResult memory res=router.executeOrder(id,data);
executed=res.status==OrderV3Types.LifecycleStatus.Executed;
if (!executed) {
console2.log("ORDER NOT EXECUTED status",uint256(res.status));
console2.log("  pendingReason",uint256(res.pendingReason));
console2.log("  terminalReason",uint256(res.terminalReason));
}
}
function _settleEpoch() internal {
_publish();
router.updateMarkPrice(_live());
router.settleLpEpoch(_live());
}
function _toNextEpoch(uint256 offset) internal {
uint256 t=_now();
vm.warp(t - (t % 1 hours) + 1 hours + offset);
}
struct Out {
uint256 aDep;
uint256 aEnd;
uint256 bStart;
uint256 bEnd;
uint256 oStart;
uint256 oEnd;
uint256 vStart;
uint256 vEndConv;
uint256 vEndAfterOClose;
uint256 vEndRealizedAfterOClose;
uint256 treasuryDelta;
uint256 rebateR;
uint256 poolTaBeforeBOpen;
uint256 poolTaAfterBOpen;
uint256 aShares;
uint256 aWalletEnd;
uint256 aSharesLeft;
int256 dSeedJ;
int256 dSeedS;
int256 dSlp;
int256 dKeeper;
int256 dKeeperCH;
bool conservationComplete;
}
function _val(address x) internal view returns (uint256) {
return settlementToken.balanceOf(x) + clearinghouse.balanceAssets(x) + engine.traderClaimBalanceAssets(x);
}
function _run(uint256 aDep,bool bTrades) internal returns (Out memory o) {
_toNextEpoch(60);
_publish();
router.updateMarkPrice(_live());
uint256 depth=pool.totalAssets();
uint256 notional=(depth * 3_950) / 10_000;
notional=(notional / 100e6) * 100e6;
uint256 sizeDelta=notional * 1e12;
uint256 margin=(notional * 3) / 100 + 5e6;
_fundTrader(O,margin * 2 + 20e6);
_fundTrader(B,margin * 2 + 20e6);
uint256 treas0=clearinghouse.balanceAssets(engine.protocolTreasury());
o.aDep=aDep;
o.conservationComplete=aDep>0||bTrades;
o.vStart=juniorVault.convertToAssets(juniorVault.balanceOf(V));
o.bStart=_val(B);
o.oStart=_val(O);
uint256 seedJ0=juniorVault.convertToAssets(juniorVault.balanceOf(juniorVault.seedReceiver()));
uint256 seedS0=seniorVault.convertToAssets(seniorVault.balanceOf(seniorVault.seedReceiver()));
uint256 slp0=seniorVault.convertToAssets(seniorVault.balanceOf(SLP));
uint256 keeper0=settlementToken.balanceOf(KEEPER);
uint256 keeperCH0=clearinghouse.balanceAssets(KEEPER);
console2.log("--- scenario: A deposit",aDep);
console2.log("B trades",bTrades);
console2.log("pool depth (pre) totalAssets",depth);
console2.log("junior totalAssets / supply",juniorVault.totalAssets(),juniorVault.totalSupply());
console2.log("notional (each of O, B)",notional);
require(_commitExecBy(O,_req(CfdTypes.Side.LONG,sizeDelta,margin,false),KEEPER),"O open failed");
{
(,,,,,,int256 vpiO)=engine.positions(O);
require(vpiO>0,"O did not pay positive VPI");
console2.log("O vpiAccrued (+ = paid)",vpiO);
}
console2.log("pool totalAssets after O open",pool.totalAssets());
uint256 rid;
if (aDep>0) {
settlementToken.mint(A,aDep);
vm.startPrank(A);
settlementToken.approve(address(juniorVault),aDep);
rid=juniorVault.requestDeposit(aDep,A,A);
vm.stopPrank();
console2.log("A deposit request id",rid);
console2.log("A deposit activation start",juniorVault.depositEpochStart(rid));
console2.log("now",_now());
}
if (bTrades) {
o.poolTaBeforeBOpen=pool.totalAssets();
vm.warp(_now() + 30);
require(_commitExec(B,_req(CfdTypes.Side.SHORT,sizeDelta,margin,false)),"B open failed");
o.poolTaAfterBOpen=pool.totalAssets();
(uint256 psz,,,,,,int256 vpiB)=engine.positions(B);
require(vpiB<0,"B did not receive negative VPI");
o.rebateR=uint256(-vpiB);
require(
clearinghouse.vpiRebateReserveAssets(B)==o.rebateR,
"negative VPI is not fully reserve-backed"
);
console2.log("B size",psz);
console2.log("B vpiAccrued (- = rebate received)",vpiB);
console2.log("B vpiRebateReserve",clearinghouse.vpiRebateReserveAssets(B));
console2.log("pool totalAssets before/after B open",o.poolTaBeforeBOpen,o.poolTaAfterBOpen);
}
console2.log("V value pre-settle",juniorVault.convertToAssets(juniorVault.balanceOf(V)));
_toNextEpoch(5);
uint256 jTa0=juniorVault.totalAssets();
if (aDep>0) _settleEpoch();
console2.log("junior totalAssets pre/post settle",jTa0,juniorVault.totalAssets());
console2.log("junior supply post settle",juniorVault.totalSupply());
console2.log("V value post-settle",juniorVault.convertToAssets(juniorVault.balanceOf(V)));
if (aDep>0) {
uint256 cl=juniorVault.claimableDepositRequest(rid,A);
vm.prank(A);
o.aShares=juniorVault.claimDeposit(rid,cl,A,A);
console2.log("A claimed shares",o.aShares);
console2.log("A share of junior (bps)",(o.aShares * 10_000) / juniorVault.totalSupply());
}
if (bTrades) {
vm.warp(_now() + 30);
console2.log("B reserve before close",clearinghouse.vpiRebateReserveAssets(B));
uint256 taPreClose=pool.totalAssets();
require(_commitExec(B,_req(CfdTypes.Side.SHORT,sizeDelta,0,true)),"B close failed");
console2.log("pool totalAssets pre/post B close",taPreClose,pool.totalAssets());
if (engine.traderClaimBalanceAssets(B)>0) {
vm.prank(B);
engine.settleTraderClaim(B);
}
uint256 free=clearinghouse.getAccountAssetBuckets(B).freeSettlementAssets;
uint256 bWalletBeforeMarginWithdrawal=settlementToken.balanceOf(B);
if (free>0) {
vm.prank(B);
clearinghouse.withdrawMargin(free);
}
require(
settlementToken.balanceOf(B)==bWalletBeforeMarginWithdrawal + free,
"B entry free margin was not transferred to wallet"
);
(uint256 psz2,,,,,,)=engine.positions(B);
require(psz2==0,"B not flat");
require(clearinghouse.balanceAssets(B)==0,"B clearinghouse balance remains");
require(engine.traderClaimBalanceAssets(B)==0,"B trader claim remains");
require(clearinghouse.vpiRebateReserveAssets(B)==0,"B reserve remains after full close");
}
console2.log("V value after B close",juniorVault.convertToAssets(juniorVault.balanceOf(V)));
if (aDep>0) {
_toNextEpoch(10);
uint256 sh=juniorVault.maxRequestRedeem(A);
require(sh==juniorVault.balanceOf(A),"A full entry-side redemption unavailable");
console2.log("A maxRequestRedeem shares / balance",sh,juniorVault.balanceOf(A));
vm.prank(A);
uint256 rr=juniorVault.requestRedeem(sh,A,A);
_toNextEpoch(5);
_settleEpoch();
uint256 cl2=juniorVault.claimableRedeemRequest(rr,A);
require(cl2==sh,"A entry-side redemption was not fully funded");
console2.log("A claimable redeem shares",cl2);
uint256 aWalletBeforeClaim=settlementToken.balanceOf(A);
vm.prank(A);
uint256 aClaimed=juniorVault.claimRedeem(rr,cl2,A,A);
require(aClaimed>0,"A entry-side redemption assets are zero");
require(settlementToken.balanceOf(A)==aWalletBeforeClaim + aClaimed,"A entry claim transfer mismatch");
require(juniorVault.balanceOf(A)==0,"A entry shares remain after full exit");
require(juniorVault.pendingRedeemRequest(rr,A)==0,"A entry redeem remains pending");
require(juniorVault.refundableRedeemRequest(rr,A)==0,"A entry redeem is refundable");
require(juniorVault.claimableRedeemAssets(rr,A)==0,"A entry redeem assets remain claimable");
}
o.aSharesLeft=juniorVault.balanceOf(A);
o.aWalletEnd=settlementToken.balanceOf(A);
o.aEnd=o.aWalletEnd + juniorVault.convertToAssets(o.aSharesLeft);
o.bEnd=_val(B);
o.oEnd=_val(O);
o.vEndConv=juniorVault.convertToAssets(juniorVault.balanceOf(V));
console2.log("B USDG wallet after close and margin withdrawal",settlementToken.balanceOf(B));
console2.log("B clearinghouse assets after close and margin withdrawal",clearinghouse.balanceAssets(B));
console2.log("B trader claim after close",engine.traderClaimBalanceAssets(B));
console2.log("B VPI reserve after close",clearinghouse.vpiRebateReserveAssets(B));
console2.log("aggregate trader claims after B close",engine.totalTraderClaimBalanceAssets());
console2.log("Pool raw USDG balance after B close",settlementToken.balanceOf(address(pool)));
console2.log("Pool totalAssets after B close",pool.totalAssets());
o.treasuryDelta=clearinghouse.balanceAssets(engine.protocolTreasury()) - treas0;
o.dSeedJ=int256(juniorVault.convertToAssets(juniorVault.balanceOf(juniorVault.seedReceiver()))) - int256(seedJ0);
o.dSeedS=int256(seniorVault.convertToAssets(seniorVault.balanceOf(seniorVault.seedReceiver()))) - int256(seedS0);
o.dSlp=int256(seniorVault.convertToAssets(seniorVault.balanceOf(SLP))) - int256(slp0);
o.dKeeper=int256(settlementToken.balanceOf(KEEPER)) - int256(keeper0);
o.dKeeperCH=int256(clearinghouse.balanceAssets(KEEPER)) - int256(keeperCH0);
uint256 snap=vm.snapshotState();
vm.warp(_now() + 20);
require(
_commitExecBy(O,_req(CfdTypes.Side.LONG,sizeDelta,0,true),KEEPER),
"O close required for matched passive LP exit"
);
{
o.vEndAfterOClose=juniorVault.convertToAssets(juniorVault.balanceOf(V));
uint256 vCooldown=juniorVault.lastDepositTime(V) + 1 hours + 1;
if (_now()<vCooldown) vm.warp(vCooldown);
uint256 vShares=juniorVault.balanceOf(V);
uint256 vMaxShares=juniorVault.maxRequestRedeem(V);
require(vMaxShares==vShares,"passive LP full redeem unavailable after O close");
vm.prank(V);
uint256 vRequest=juniorVault.requestRedeem(vShares,V,V);
_toNextEpoch(5);
_settleEpoch();
uint256 vClaimableShares=juniorVault.claimableRedeemRequest(vRequest,V);
require(vClaimableShares==vShares,"passive LP full redemption was not funded after O close");
uint256 vClaimableAssets=juniorVault.claimableRedeemAssets(vRequest,V);
require(vClaimableAssets>0,"passive LP redemption assets are zero");
console2.log("V claimable redeem shares",vClaimableShares);
console2.log("V claimable redeem assets",vClaimableAssets);
uint256 vWalletBeforeClaim=settlementToken.balanceOf(V);
vm.prank(V);
o.vEndRealizedAfterOClose=juniorVault.claimRedeem(vRequest,vClaimableShares,V,V);
require(o.vEndRealizedAfterOClose>0,"V realized exit is zero");
require(
settlementToken.balanceOf(V)==vWalletBeforeClaim + o.vEndRealizedAfterOClose,
"V claim transfer mismatch"
);
require(juniorVault.balanceOf(V)==0,"passive LP shares remain after full exit");
require(juniorVault.pendingRedeemRequest(vRequest,V)==0,"passive LP shares remain pending");
require(juniorVault.refundableRedeemRequest(vRequest,V)==0,"passive LP shares are refundable");
require(juniorVault.claimableRedeemAssets(vRequest,V)==0,"passive LP redeem assets remain claimable");
}
vm.revertToState(snap);
}
function _report(string memory tag,Out memory o) internal pure {
console2.log("== RESULT (USDG 6-dec) ==",tag);
int256 aNet=int256(o.aEnd) - int256(o.aDep);
int256 bNet=int256(o.bEnd) - int256(o.bStart);
console2.log("A deposit",o.aDep);
console2.log("A end (wallet + residual shares)",o.aEnd);
console2.log("A USDG wallet after redemption",o.aWalletEnd);
console2.log("A shares remaining",o.aSharesLeft);
console2.log("A shares minted",o.aShares);
console2.log("R (B rebate received at open)",o.rebateR);
console2.log("A net",aNet);
console2.log("B net",bNet);
console2.log("COALITION NET (A+B)",aNet + bNet);
console2.log("V start",o.vStart);
console2.log("V end (conv, O open)",o.vEndConv);
console2.log("V delta (conv, O open)",int256(o.vEndConv) - int256(o.vStart));
console2.log("V end (conv, after O closes)",o.vEndAfterOClose);
console2.log("V delta (after O closes)",int256(o.vEndAfterOClose) - int256(o.vStart));
console2.log("V realized full-exit assets after O closes",o.vEndRealizedAfterOClose);
console2.log("V realized full-exit delta after O closes",int256(o.vEndRealizedAfterOClose) - int256(o.vStart));
console2.log("O delta",int256(o.oEnd) - int256(o.oStart));
console2.log("treasury delta (fees)",o.treasuryDelta);
console2.log("delta junior seed holder",o.dSeedJ);
console2.log("delta senior seed holder",o.dSeedS);
console2.log("delta senior LP",o.dSlp);
console2.log("delta keeper wallet",o.dKeeper);
console2.log("delta keeper clearinghouse credit",o.dKeeperCH);
int256 sum=aNet + bNet + (int256(o.vEndConv) - int256(o.vStart)) + (int256(o.oEnd) - int256(o.oStart))
+ int256(o.treasuryDelta) + o.dSeedJ + o.dSeedS + o.dSlp + o.dKeeper + o.dKeeperCH;
console2.log("sum of tracked deltas (including keeper clearinghouse credit)",sum);
if (o.conservationComplete) require(sum==0,"token conservation must reconcile including keeper credit");
else console2.log("conservation checkpoint excludes unreconciled O-only VPI; passive exit is measured after O close");
}
uint256 internal constant JR0=2_000_000e6;
function testEntryAttack_s40() public {
Out memory o=_run(1_333_333e6,true);
_report("ATTACK s=0.4",o);
}
function testEntryAttack_s60() public {
Out memory o=_run(3_000_000e6,true);
_report("ATTACK s=0.6",o);
}
function testEntryAttack_s80() public {
Out memory o=_run(8_000_000e6,true);
_report("ATTACK s=0.8",o);
}
function testEntryControl_NoB_s40() public {
Out memory o=_run(1_333_333e6,false);
_report("CONTROL-a (B idle) s=0.4",o);
}
function testEntryControl_NoB_s60() public {
Out memory o=_run(3_000_000e6,false);
_report("CONTROL-a (B idle) s=0.6",o);
}
function testEntryControl_NoB_s80() public {
Out memory o=_run(8_000_000e6,false);
_report("CONTROL-a (B idle) s=0.8",o);
}
function testEntryControl_NoDeposit() public {
Out memory o=_run(0,true);
_report("CONTROL-b (no A deposit, B trades)",o);
}
function testEntryControl_NoDeposit_NoOpposingTrade() public {
Out memory o=_run(0,false);
_report("CONTROL-c (nothing, O only)",o);
}
function _compareAssert(uint256 aDep,string memory tag,bool requireProfit) internal {
uint256 snap=vm.snapshotState();
Out memory ctl=_run(aDep,false);
vm.revertToState(snap);
Out memory atk=_run(aDep,true);
int256 aCtl=int256(ctl.aEnd) - int256(aDep);
int256 coalition=(int256(atk.aEnd) - int256(aDep)) + (int256(atk.bEnd) - int256(atk.bStart));
int256 coalitionVsCtl=coalition - aCtl;
int256 vLossVsCtl=int256(ctl.vEndConv) - int256(atk.vEndConv);
int256 vLossVsCtlAfterO=int256(ctl.vEndAfterOClose) - int256(atk.vEndAfterOClose);
int256 vRealizedLossVsCtl=int256(ctl.vEndRealizedAfterOClose) - int256(atk.vEndRealizedAfterOClose);
console2.log("##### COMPARE",tag);
console2.log("R",atk.rebateR);
console2.log("coalition net (absolute)",coalition);
console2.log("control-a A net",aCtl);
console2.log("coalition net minus control-a A net",coalitionVsCtl);
console2.log("V loss vs control-a (conv, O open)",vLossVsCtl);
console2.log("V loss vs control-a (after O closes)",vLossVsCtlAfterO);
console2.log("V realized loss vs control-a (after O closes)",vRealizedLossVsCtl);
console2.log("A gain vs control-a",(int256(atk.aEnd) - int256(ctl.aEnd)));
console2.log("B cost (net)",int256(atk.bEnd) - int256(atk.bStart));
console2.log("junior seed holder loss vs control-a",ctl.dSeedJ - atk.dSeedJ);
assertLt(atk.vEndRealizedAfterOClose,atk.vStart,"passive LP did not realize principal loss");
assertGe(ctl.vEndRealizedAfterOClose,ctl.vStart,"matched control passive LP lost principal");
assertGt(vRealizedLossVsCtl,0,"realized passive loss absent against control");
if (requireProfit) {
assertGt(coalition,0,"coalition not profitable");
} else {
assertLt(coalition,0,"s=0.4 case should remain unprofitable");
}
}
function testEntryCompare_s40() public {
_compareAssert(1_333_333e6,"s=0.4",false);
}
function testEntryCompare_s60() public {
_compareAssert(3_000_000e6,"s=0.6",true);
}
function testEntryCompare_s80() public {
_compareAssert(8_000_000e6,"s=0.8",true);
}
}
```

### Actual Foundry run (original readable 24-test source)

```bash
FOUNDRY_FFI=false forge test --root packages/perps --match-path test/perps/VpiNavTimingExtraction.t.sol -vvv
```

Exit code 0; Foundry 1.8.3 / Solc 0.8.35. Actual stdout excerpt:

```text
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 105.29s
Ran 7 tests for test/perps/VpiNavTimingExtraction.t.sol:VpiNavExitTest
[PASS] testExitCapture_Attack()
[PASS] testExitCapture_Attack_10pct()
[PASS] testExitCapture_Attack_25pct()
[PASS] testExitCapture_Attack_35pct()
[PASS] testExitCapture_Attack_KeeperExecutes()
[PASS] testExitCapture_Control()
[PASS] testExitCapture_Control_CloseFirst()
Suite result: ok. 7 passed; 0 failed; 0 skipped

Ran 11 tests for test/perps/VpiNavTimingExtraction.t.sol:VpiNavEntryTest
[PASS] testEntryAttack_s40()
[PASS] testEntryAttack_s60()
[PASS] testEntryAttack_s80()
[PASS] testEntryCompare_s40()
[PASS] testEntryCompare_s60()
[PASS] testEntryCompare_s80()
[PASS] testEntryControl_NoB_s40()
[PASS] testEntryControl_NoB_s60()
[PASS] testEntryControl_NoB_s80()
[PASS] testEntryControl_NoDeposit()
[PASS] testEntryControl_NoDeposit_NoOpposingTrade()
Suite result: ok. 11 passed; 0 failed; 0 skipped

Ran 6 tests for test/perps/VpiNavTimingExtraction.t.sol:VpiNavRepeatabilityTest
[PASS] testControl_LPChurnOnly()
[PASS] testControl_NoRedeem()
[PASS] testLargerCoalitionShare()
[PASS] testSingleCycleBaseline()
[PASS] testSixCycles_RealizedFinalExit()
[PASS] testSixCycles_ShareValue()
Suite result: ok. 6 passed; 0 failed; 0 skipped

Ran 3 test suites in 185.65ms (428.01ms CPU time): 24 tests passed, 0 failed, 0 skipped (24 total tests)
```

Representative actual console balances (6-decimal USDG atoms):

```text
A claimed assets 3003254591487
B USDG wallet after close and margin withdrawal 163712132726
B clearinghouse assets after close and margin withdrawal 0
B trader claim after close 0
B VPI reserve after close 0
sum of tracked deltas (including LP holders and recipients) 0
V USDG wallet after full redemption 998892095167
COALITION NET 1047865777
V delta (realized exit) -1104190974

A final redeem assets 3019555328655
A realized wallet balance 3019555328655
B realized wallet balance after margin withdrawal 163836352929
B external USDG capital supplied 177149084457
coalition realized delta after six cycles 6253738691
passive LP realized full-exit assets 993367630890
passive LP realized delta -6628655251
```

O-only control: no settlement call is made without a pending LP request; passive exit is measured after O closes.

