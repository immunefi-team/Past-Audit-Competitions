# \#99500 [SC-Medium] A 10% mark deletes parked recap, and the flat settle pays it to junior

**Submitted on Nov 13th 2026 at 22:21:17 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99500
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When senior principal already sits on its high-water mark, `recapitalizePool` parks the new USDG in `unassignedAssets`. That balance is reserved out of LP withdrawals. `assignUnassignedAssets` is the call that can still hand it out. A short marked 10% above its entry makes terminal equity smaller than the parked balance. `planReconcile` then lowers `unassignedAssets` to whatever distributable is left, and it never writes the tag back up. The same settle wipes both principals. One hour later the basket is 1.00 again. The short is closed at that recovered price, so the marked loss is not paid to the trader and the cash stays in the pool. On that flat book the cap predicate is false: distributable is above the stored tag. The next `settleLpEpoch` still seeds distributable minus the stale tag through the revenue route. Senior is restored only up to the old high-water mark. The rest becomes junior principal. Bob's `claimRedeem` pays that principal out in USDG.

On the Arbitrum Sepolia v2.0.0 fork, Bob deposited 2,000,000 USDG and was paid 3,108,751.362680 USDG. The 1,108,751.362680 USDG above the deposit is his share of the 1,639,548.074718 USDG the 1.10 mark deleted from `unassignedAssets`. The owner can afterwards assign only the 158,360,571.925282 USDG the cap left behind.

## Vulnerability Details

`CfdEngine.recapitalizePool` pulls USDG from the engine owner into HousePool and records a recapitalization inflow. It mints no shares. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` restores senior principal only up to `seniorHighWaterMark`. Anything left is added to `unassignedAssets`. The function comment says recapitalization never accrues to junior principal. `ACCOUNTING_SPEC.md` says the same thing in the ownership section: the queued amount goes to senior restoration or to unassigned assets, and only value with no safe claimant path belongs in `unassignedAssets`.

The comparison rule is in the waterfall section. Terminal equity is compared with total principal after reserved pending inflows and `unassignedAssets` are taken out. A shortfall is charged to junior, then senior. Losses do not reduce the senior high-water mark. Surplus restores senior to the mark, then goes to junior.

`HousePoolReconcilePlanLib.planReconcile` does the cap before that comparison, and the cap only moves one way:

```solidity
plan.claimedEquityZero = plan.state.waterfall.seniorPrincipal + plan.state.waterfall.juniorPrincipal == 0;

plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;

if (plan.claimedEquityZero) {
    uint256 seededDistributableToClaims = snapshot.distributable > plan.state.unassignedAssets
        ? snapshot.distributable - plan.state.unassignedAssets
        : 0;
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    return plan;
}
```

The comment names those two operations. It caps `unassignedAssets` when the stored tag is above distributable, and it seeds distributable minus the stored tag when principal is zero. It does not say to keep the lower number after distributable recovers.

`claimedEquityZero` is taken from principal after the coupon, before the cap. On the stress settle the principals are still there, so the seed branch is skipped. `buildReconcileSnapshot` adds a negative `terminalLpPriceDeltaAssets` to liabilities, and distributable is physical assets minus those liabilities. The short is 574,000 lots, entered at 1.00000000. At 1.10000000 the terminal delta is -5,740,000.000000 USDG. That loss is larger than the 4,000,002.000007 USDG of principal, so distributable falls through the parked balance and the cap stores the lower tag. The write-off is the stress loss minus principal minus the open-fee cash that had just arrived:

```text
5,740,000.000000 - 4,000,002.000007 - 100,449.925275 = 1,639,548.074718
```

Distributable left for claims is then 0. `absorbLoss` takes the shortfall from junior and then senior and leaves the high-water mark where it is. Both principals print 0. Pool USDG does not move. Bob's redeem in that epoch is refunded. The engine cap is still 2.00000000. This print is 1.10.

One hour later `router.updateMarkPrice` publishes basket 1.00000000. The short is still open. The call does not settle the pool. Terminal delta at that mark is 0. `unassignedAssets` stays 158,360,571.925282. Both principals stay 0. The cash balance does not move. The close is the next second, through `commitOrder` and `executeOrder`, at the same 1.00000000 basket. A short profits when the exit value is at least the entry cost, so this fill has no price PnL. The trader's wallet does not gain. The pool drops 77,370.482720 USDG of round-trip fees. The tag stays capped.

After that close the book is flat. Pending revenue is 57.919296 USDG and trader claims are 0, so distributable is 164,023,143.523266 USDG. The stored tag is 158,360,571.925282 USDG. The cap predicate `unassigned > distributable` is false. The gap between them is 5,662,571.597984 USDG, which still contains the 1,639,548.074718 USDG that was quarantined recap. `HousePool._reconcile` writes `unassignedAssets = plan.state.unassignedAssets` and has no path that restores the deleted amount when distributable recovers. The cap expression is a minimum against whatever number was stored last time.

The recovery settle is flat, so `OrderRouter.settleLpEpoch` is permissionless. Principal is 0, so `claimedEquityZero` is true. The cap leaves the stored tag where it is, because the tag is already below distributable. The seed is accounted cash minus that tag, 5,662,629.517280 USDG. `applyRevenueIntent` is called with claimed equity disabled. Principal is 0, so the early return that would park the amount back into unassigned does not fire. Senior supply is nonzero, so the gap `seniorHighWaterMark - 0` is filled first. Senior comes back to 1,001,005.585252 USDG, the high-water mark `absorbLoss` kept. Junior supply is nonzero, so the remainder is added to `juniorPrincipal`.

`HousePool._settleLpEpoch` reconciles before it prices the junior phase. The phase uses `juniorPrincipal`. The redemption phase transfers settlement token into the vault. `claimRedeem` pays Bob. `getFreeAssets` before the stress settle was 4,000,002.000007, equal to the two principals, so the parked recap was reserved. After the seed the same cash is principal, and the redeem pays it.

`assignUnassignedAssets` reads the stored `unassignedAssets` after its own reconcile. That read returns the capped balance. The 1,639,548.074718 USDG is no longer in the balance the owner can assign.

The 5,740,000 USDG terminal delta is the mark on an open short, and the close at the recovered 1.00 print does not pay it out. The USDG Bob receives above his deposit was the recap the cap took off `unassignedAssets`. F-98667 is a different root: a replaced recap mark pays booked senior principal to junior while senior is impaired. Here senior starts equal to its high-water mark, the recap is parked, and the paying balance is the amount the cap deleted. F-97677 is the flat-short pledge. This short is filled at 1.00000000, closed at 1.00000000, and the trader wallet gain on the close is 0.

`commitOrder`, `executeOrder`, `updateMarkPrice`, and `settleLpEpoch` are external router functions. After `recapitalizePool`, the owner is not called again. The assign call in the test is reverted. It shows the balance the owner can still see.

## Impact Details

Impact: Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield. Critical.

The recap is HousePool USDG. Before the stress settle it sits in `unassignedAssets`, and `getFreeAssets` does not include it. After the recovery redeem it is in Bob's wallet. The owner funded it through the documented `recapitalizePool`. The owner does not pick a share price and does not sign the open, the close, or either settle.

Settlement token is the deployed Sepolia USDG, 6 decimals. The fork is Arbitrum Sepolia v2.0.0 at block 324500000. Perps is not on mainnet, so the figures are the measured transfer on this position. The fork already had senior and junior principal. Bob's 2,000,000 USDG is a deposit into that junior vault, and his redeem is his share of the junior principal the seed creates. The rest of the junior seed stays as `juniorPrincipal` for the shares that do not exit.

| Account | Deposited (USDG) | Received (USDG) | Difference (USDG) |
|---|---:|---:|---:|
| Bob, junior | 2,000,000.000000 | 3,108,751.362680 | +1,108,751.362680 |
| Trader wallet on the close | margin stays in the clearinghouse | 0 | 0 |
| Pool on the round trip | | | -77,370.482720 |

The short is 574,000 lots, notional 57,400,000 USDG, margin 28,720,000 USDG plus a 1,000 USDG bounty buffer in the clearinghouse. The stress print is 1.10000000, ten percent above the 1.00000000 entry, and under the 2.00000000 engine cap. Price PnL on the round trip is 0. The 77,370.482720 USDG is the fee cash that leaves the pool. One person can be Bob and the trader. That person's net, after the whole fee, is 1,031,380.879960 USDG. If another account holds the short, Bob's excess is the full 1,108,751.362680 USDG.

The tag that disappears is larger than Bob's excess, because other junior shares remain. The identities from the run:

- Parked unassigned 160,000,120.000000. Capped unassigned 158,360,571.925282. Written off 1,639,548.074718. The last two sum to the parked balance.
- Stress settle cash is 164,100,571.925282 before and after. Both principals are 0. Bob's USDG stays 0 and his shares are returned.
- The recovered `updateMarkPrice` is at timestamp 1793592001. The stress mark was 1793588401. The gap is 3,600 seconds. Mark 100,000,000. Terminal delta 0. Unassigned and both principals are unchanged, and the pool balance does not move.
- Close one second later, at the same 1.00 basket. Terminal delta 0, trader wallet gain 0, unassigned unchanged, principals still 0.
- Flat distributable is 164,023,143.523266. The cap predicate does not bind. Distributable minus the stored tag is 5,662,571.597984, which is the wiped principal plus the net fee cash minus 57.919296 of pending revenue, plus the 1,639,548.074718 write-off.
- Seeded principal, senior left plus junior left plus Bob's payment, is 5,662,629.517280. Accounted cash before the redeem minus the capped tag is the same number.
- That seed is the pre-stress principal of 4,000,002.000007 plus an increase of 1,662,627.517273. The increase is the write-off 1,639,548.074718, plus the 100,449.925275 of cash the open added, minus the 77,370.482720 the close removed.
- Junior principal rises by 1,662,618.375753 from its pre-stress value. That is the write-off plus 23,070.301035. Senior is restored to 1,001,005.585252, which is 9.141520 above its pre-stress principal, the high-water mark `absorbLoss` kept. Those two leftovers sum to the 23,079.442555 of fee cash that stayed in the pool.
- USDG `totalSupply` does not change on the recovery settle. The pool balance falls by exactly Bob's 3,108,751.362680.
- `assignUnassignedAssets` mints against 158,360,571.925282. After the snapshot revert, Bob's wallet and the pool balance are the paid figures.

## References

- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol`: `planReconcile` (line 47). Zero-principal flag at line 62, before the cap. Cap at lines 73-74. Seed at lines 76-81.
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`: `applyClaimantRecapitalizationIntent` (line 111) parks the remainder in `unassignedAssets`. `applyRevenueIntent` (line 158) restores senior up to the high-water mark and adds the rest to `juniorPrincipal` when principal is 0.
- `packages/perps/src/libraries/HousePoolAccountingLib.sol`: `buildReconcileSnapshot` (line 60). A negative terminal delta increases liabilities.
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `absorbLoss` (line 167). The high-water mark is left unchanged.
- `packages/perps/src/HousePool.sol`: `_reconcile` (line 1562) stores the capped unassigned at line 1585. `_settleLpEpoch` (line 824) reconciles before redemption pricing. `assignUnassignedAssets` (line 660) assigns the stored balance. `getFreeAssets` (line 1235).
- `packages/perps/src/CfdEngine.sol`: `recapitalizePool` (line 632).
- `packages/perps/src/OrderRouter.sol`: `commitOrder` (line 85), `executeOrder` (line 240), `updateMarkPrice` (line 429), `settleLpEpoch` (line 479).
- `packages/perps/ACCOUNTING_SPEC.md`: line 315, value with no claimant path sits in `unassignedAssets`. Lines 336-338, terminal equity excluding unassigned is compared with principal, and losses do not reduce the senior high-water mark. Lines 642-650, recap allocates to senior restoration or unassigned, and only value with no safe claimant path belongs there.
- Source pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. `packages/perps/src` at that commit matches the Arbitrum Sepolia v2.0.0 deployment.

## Proof of Concept

## Proof of Concept

Run from `audit-comp-plether/packages/perps` (`foundry.toml`: solc 0.8.35, via-IR, optimizer runs 200). Optional env `ARB_SEPOLIA_RPC_URL`. If it is unset, the test forks `https://arbitrum-sepolia.gateway.tenderly.co` at block 324500000.

```bash
cd audit-comp-plether/packages/perps && forge test --match-contract F22451CleanFork --threads 1 -vv
```

The recorded run added `--offline` after that fork block was cached. Suite result: ok. 1 passed, 0 failed, 334.54ms.

Chain id 421614. HousePool `0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e`, codehash `0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957`. OrderRouter `0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444`. USDG `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D`, the deployment's mintable token. No in-scope file is edited. No protocol `vm.mockCall` or `vm.store`.

Pyth in the test is `MockPyth` from `packages/shared/test-support`, etched over the oracle's Pyth address `0x0B73614636C855Bf23F342F307FB981A3e47f42B`. A signed Pyth update cannot be produced offline, and the program forbids calling live Pyth. The six feed ids, weights, base prices, and inversions are read from the deployed oracle. The test stops unless those weights sum to 1e18 and the preview basket is exactly 1.00000000 and 1.10000000. Confidence is 0. The engine mark after `updateMarkPrice` and after `settleLpEpoch` is the basket the payload asked for. Foundry has no ArbSys precompile, so `ArbitrumForkSupport.install` etches the repo's `ForkArbSys` stub at address 100. That stub is not a protocol contract.

Opens and closes go through `commitOrder` and `executeOrder`. Both returned status 2, pending reason 0, terminal reason 1. The stress epoch and the recovery epoch go through `settleLpEpoch`. The 1.00 print an hour after the stress mark goes through `updateMarkPrice` while the short is still open, and it does not settle the pool. The owner calls `recapitalizePool` once, for 40 times the principal then outstanding. Bob's payout is `claimRedeem`.

The fill warps inside `_executeFill`, immediately before `executeOrder`. via-IR caches `block.timestamp` across `vm.warp` in the same function, and the oracle reads the EVM clock. The historical publish time is `commitTime + 1`, with `prevPublishTime` equal to the commit time. Live mode is bit 1. The close fill is one second after the recovered live mark, and both prices are 1.00000000.

Steps:

1. Warp to Sunday 2026-11-01 22:30:00 UTC and publish basket 1.00000000 through `router.updateMarkPrice`. The book is flat, unassigned is 0, and the order queue is empty.
2. Bob `requestDeposit`s 2,000,000 USDG. A dust account deposits 1 USDG. Each deposit is settled with `router.settleLpEpoch` and claimed.
3. The engine owner calls `recapitalizePool` for 40 times the principal then outstanding. The dust account redeems. That settle parks the recap. Dust is paid 0.999993 USDG. Unassigned is 160,000,120.000000. Free cash is 4,000,002.000007, equal to principal.
4. The trader deposits margin and opens a 574,000 lot short through `commitOrder` and `executeOrder` at basket 1.00000000. Entry price is 100,000,000. Notional is 57,400,000 USDG. Unassigned does not change. Terminal delta is 0.
5. After the deposit cooldown, Bob `requestRedeem`s. At the next epoch, `router.settleLpEpoch` publishes basket 1.10000000. Terminal delta is -5,740,000.000000. The pool balance does not move. Both principals are 0. Unassigned is 158,360,571.925282. Written off 1,639,548.074718. Bob's redeem pays 0 and the shares come back.
6. One hour later, `router.updateMarkPrice` publishes basket 1.00000000. Stress mark time 1793588401, recovered mark time 1793592001. The short is still open. Terminal delta is 0. Unassigned and both principals are unchanged. The pool balance does not move.
7. The trader closes the whole short through `commitOrder` and `executeOrder` at that same 1.00000000 basket, one second later. Size is 0. Terminal delta is 0. Trader wallet gain is 0. The pool drops 77,370.482720 USDG. Distributable is 164,023,143.523266 and the cap predicate does not bind. Unassigned and both principals stay where the stress settle left them.
8. Bob `requestRedeem`s again. The next `router.settleLpEpoch` is at basket 1.00000000. `claimRedeem` pays 3,108,751.362680 USDG. The pool balance falls by that amount. Total supply is unchanged. Unassigned is still 158,360,571.925282.
9. The test snapshots, calls `assignUnassignedAssets(false, assignee)`, and reverts the snapshot. Assignable is 158,360,571.925282. Bob's wallet and the pool balance after the revert are the figures from step 8.

```text
dustPaid 999993
unassignedBefore 160000120000000
freeBefore 4000002000007
principalBefore 4000002000007
shortLots 574000
expectedStressLoss 5740000000000
executeStatus 2
executePending 0
executeTerminal 1
deltaAtEntry 0
deltaAtStress -5740000000000
unassignedCapped 158360571925282
writtenOff 1639548074718
recoveredMarkTime 1793592001
stressMarkTime 1793588401
deltaAtRecoveredMark 0
executeStatus 2
executePending 0
executeTerminal 1
deltaFlat 0
distributableFlat 164023143523266
capPredicateBinds 0
poolDropOnClose 77370482720
traderWalletGain 0
bobPaid 3108751362680
principalSeeded 5662629517280
expectedSeed 5662629517280
juniorIncrease 1662618375753
principalIncrease 1662627517273
bobExcess 1108751362680
assignableAfterCap 158360571925282
```

Full test, `audit-comp-plether/packages/perps/test/perps/fork/F22451CleanFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";
import {Test} from "forge-std/Test.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Clean fork PoC for the unassigned-cap seed.
/// @dev Local Arbitrum Sepolia fork only. Production path is `OrderRouter.commitOrder`,
///      `OrderRouter.executeOrder`, `OrderRouter.updateMarkPrice`, and `OrderRouter.settleLpEpoch`.
///      The stress print is 1.10, not the cap. The recovered 1.00 print is a later live mark.
///      MockPyth replaces only the out-of-scope Pyth endpoint. USDG is the deployed mintable
///      Sepolia token. No in-scope source edit, no protocol `vm.etch`, `vm.mockCall`, or
///      `vm.store`. The owner calls `recapitalizePool` once, which is the documented park,
///      and does not act again.
///      Run from `packages/perps`:
///        forge test --match-contract F22451CleanFork --threads 1 -vv
contract F22451CleanForkTest is Test {

    uint256 constant FORK_BLOCK = 324_500_000;
    /// @dev Sunday 2026-11-01 22:30:00 UTC, after the 17:00 New York FX open.
    uint256 constant LIVE_TS = 1_793_572_200;
    uint256 constant JUNIOR_DEPOSIT = 2_000_000e6;
    uint256 constant DUST_DEPOSIT = 1e6;
    uint256 constant ENTRY = 1e8;
    /// @dev 1.10 basket. Ten percent above entry, under the 2.00 cap.
    uint256 constant STRESS = 110_000_000;
    uint256 constant CAP = 2e8;
    /// @dev 10% of a 35% short clears principal once parked recap is about 28x principal. 40x leaves fee room.
    uint256 constant RECAP_MULTIPLE = 40;
    uint256 constant FEEDS = 6;
    uint256 constant SCALE = 1e10;
    uint256 constant LIVE_MODE = 1;
    uint256 constant MARK_ENTRY = 1;
    uint256 constant MARK_STRESS = 2;

    address constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address constant SENIOR = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address constant ROUTER_ADDR = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    bytes32 constant POOL_CODEHASH = 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;

    IUsdG internal token;
    IPool internal pool;
    IVault internal junior;
    IVault internal senior;
    IEngine internal engine;
    IRouter internal router;
    IOracle internal oracle;
    address internal pyth;

    address internal bob;
    address internal dust;
    address internal trader;
    address internal assignee;

    bytes32[] internal feedIds;
    uint256[] internal weights;
    uint256[] internal bases;
    bool[] internal inverted;
    int64[] internal rawAtEntry;
    int64[] internal rawAtStress;

    function setUp() public {
        string memory rpc = vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co"));
        vm.createSelectFork(rpc, FORK_BLOCK);
        ArbitrumForkSupport.install();
        assertEq(block.chainid, 421_614, "fork must be Arbitrum Sepolia");
        assertEq(POOL.codehash, POOL_CODEHASH, "fork is not the v2.0.0 HousePool");

        token = IUsdG(TOKEN);
        pool = IPool(POOL);
        junior = IVault(JUNIOR);
        senior = IVault(SENIOR);
        engine = IEngine(ENGINE);
        router = IRouter(engine.orderRouter());
        assertEq(address(router), ROUTER_ADDR, "engine router is not the v2.0.0 router");
        oracle = IOracle(router.pletherOracle());
        pyth = oracle.pyth();
        assertEq(pyth, RELEASE_PYTH, "oracle Pyth endpoint moved");

        bob = makeAddr("bob");
        dust = makeAddr("dust");
        trader = makeAddr("trader");
        assignee = makeAddr("assignee");

        _loadBasket();
        _installPythMock();
    }

    function test_Fork_CleanPathSeedsCappedUnassignedIntoJuniorRedeem() public {
        _warpUntilOracleLive();
        _refresh(MARK_ENTRY);
        assertEq(engine.lastMarkPrice(), ENTRY, "live mark is not 1.00");
        assertTrue(!engine.isOracleFrozen(), "oracle frozen");
        assertTrue(!engine.isFadWindow(), "FAD window");
        _log("pin");

        (uint256 longOi,,,) = engine.sides(0);
        (uint256 shortOi,,,) = engine.sides(1);
        require(longOi == 0 && shortOi == 0, "fork book is not flat");
        require(engine.CAP_PRICE() == CAP, "cap is not 2.00");
        require(STRESS > ENTRY && STRESS < CAP, "stress print is not between entry and the cap");
        require(pool.unassignedAssets() == 0, "pin already has unassigned cash");
        require(router.nextExecuteId() == router.nextCommitId(), "fork order queue is not empty");

        // Step 1. Bob deposits junior through the vault. The dust redeem is the matured
        // epoch work that commits the recap reconcile. It is priced before the cap.
        _deposit(junior, bob, JUNIOR_DEPOSIT);
        _deposit(junior, dust, DUST_DEPOSIT);
        uint256 bobShares = junior.balanceOf(bob);
        require(bobShares > 0, "junior deposit minted no shares");
        _log("deposited");

        uint256 principalBeforeRecap = pool.seniorPrincipal() + pool.juniorPrincipal();
        require(pool.seniorPrincipal() > 0 && pool.seniorPrincipal() == pool.seniorHighWaterMark(), "senior is impaired");
        require(pool.juniorPrincipal() > JUNIOR_DEPOSIT, "junior principal missed the deposit");
        _mintAndRecap(principalBeforeRecap * RECAP_MULTIPLE);
        uint256 dustPaid = _redeemAndCollect(junior, dust);
        _log("parked");
        console2.log("dustPaid", dustPaid);
        require(dustPaid > 0 && dustPaid < 2e6, "dust redeem was not the 1 USDG park settle");

        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 principalBefore = seniorBefore + juniorBefore;
        uint256 unassignedBefore = pool.unassignedAssets();
        uint256 cashBefore = token.balanceOf(POOL);
        uint256 freeBefore = pool.getFreeAssets();
        console2.log("unassignedBefore", unassignedBefore);
        console2.log("freeBefore", freeBefore);
        console2.log("principalBefore", principalBefore);
        require(pool.pendingRecapitalizationAssets() == 0, "recap still pending");
        require(unassignedBefore > principalBefore, "recap was not parked as unassigned");
        require(freeBefore + unassignedBefore <= cashBefore, "free cash already includes unassigned");

        // Step 2. Permissionless short open through commitOrder + executeOrder.
        // The cap mark has to cross principal and stay inside pool cash. The trader is not paid.
        uint256 depth = pool.totalAssets();
        uint256 lots = depth * 35 / 100 / ENTRY;
        require(lots > 0, "no short room");
        uint256 expectedLoss = lots * (STRESS - ENTRY);
        console2.log("shortLots", lots);
        console2.log("expectedStressLoss", expectedLoss);
        require(expectedLoss > principalBefore, "cap mark cannot cross principal");
        require(expectedLoss < cashBefore, "cap mark would consume more than the cash");

        uint256 notional = lots * ENTRY;
        _open(trader, lots, notional / 2 + 20_000e6);
        _log("opened");
        (uint256 openSize,, uint256 entryPrice,,,,) = engine.positions(trader);
        require(openSize == lots * 1e20, "open size mismatch");
        require(entryPrice == ENTRY, "open fill is not 1.00");
        int256 deltaAtEntry = _terminalDelta();
        console2.log("deltaAtEntry");
        console2.logInt(deltaAtEntry);
        require(pool.unassignedAssets() == unassignedBefore, "open spent the quarantined balance");

        // Step 3. Queue Bob, settle the 1.10 basket through the router, and refund the redeem.
        // The stress settle moves no USDG. It writes unassigned down and zeros both principals.
        uint256 bobBeforeCap = token.balanceOf(bob);
        uint256 capRequest = _queueRedeem(junior, bob);
        uint256 capMaturity = pool.lpEpochStart(capRequest);
        if (block.timestamp <= capMaturity) vm.warp(capMaturity + 1);
        _syncClock();
        uint256 cashAtCapBefore = token.balanceOf(POOL);
        _settle(MARK_STRESS);
        int256 deltaAtCap = _terminalDelta();
        console2.log("deltaAtStress");
        console2.logInt(deltaAtCap);
        require(engine.lastMarkPrice() == STRESS, "stress settle did not mark at 1.10");
        require(deltaAtCap < 0, "cap mark is not a trader gain");
        require(uint256(-deltaAtCap) > principalBefore, "terminal loss does not cross principal");
        uint256 capRefund = junior.refundableRedeemRequest(capRequest, bob);
        require(capRefund > 0, "cap settle did not refund the redeem");
        require(junior.claimableRedeemRequest(capRequest, bob) == 0, "cap settle funded the redeem");
        vm.prank(bob);
        junior.claimRedeemRefund(capRequest, bob, bob);
        require(token.balanceOf(bob) == bobBeforeCap, "cap settle paid bob");
        _log("capped");

        uint256 unassignedCapped = pool.unassignedAssets();
        uint256 writtenOff = unassignedBefore - unassignedCapped;
        uint256 cashAtCap = token.balanceOf(POOL);
        console2.log("unassignedCapped", unassignedCapped);
        console2.log("writtenOff", writtenOff);
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "cap settle left principal");
        require(writtenOff > 1, "cap did not write off more than 1 raw");
        require(unassignedCapped < unassignedBefore, "unassigned was not capped");
        require(cashAtCap == cashAtCapBefore, "cap settle moved USDG");
        require(junior.balanceOf(bob) == bobShares, "cap settle burned bob");

        // Step 4. An hour later, publish 1.00 through the live mark path. The short stays open.
        // This print does not settle the pool, so the capped tag and the zero principals stay.
        uint64 stressTime = engine.lastMarkTime();
        _advanceLive(uint256(stressTime) + 1 hours);
        _refresh(MARK_ENTRY);
        require(engine.lastMarkPrice() == ENTRY, "recovery print is not 1.00");
        require(uint256(engine.lastMarkTime()) >= uint256(stressTime) + 1 hours, "recovery print is not an hour later");
        require(pool.unassignedAssets() == unassignedCapped, "live print restored unassigned");
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "live print restored principal");
        int256 deltaRecovered = _terminalDelta();
        console2.log("deltaAtRecoveredMark");
        console2.logInt(deltaRecovered);
        console2.log("recoveredMarkTime", uint256(engine.lastMarkTime()));
        console2.log("stressMarkTime", uint256(stressTime));
        require(deltaRecovered == 0, "recovered mark still books the stress loss");
        _log("recovered-mark");

        // Step 5. Close at that same 1.00 basket. The fill is the next second, same price.
        // Price PnL is zero. The stress loss does not leave the pool. Unassigned stays capped.
        uint256 traderBeforeClose = token.balanceOf(trader);
        uint256 poolBeforeClose = token.balanceOf(POOL);
        (uint256 liveSize,,,,,,) = engine.positions(trader);
        _close(trader, liveSize);
        _log("flat");
        (uint256 sizeAfter,,,,,,) = engine.positions(trader);
        require(sizeAfter == 0, "close left a position");
        (, uint256 shortAfter,,) = engine.sides(1);
        (, uint256 longAfter,,) = engine.sides(0);
        require(shortAfter == 0 && longAfter == 0, "book is not flat");
        require(pool.seniorPrincipal() == 0 && pool.juniorPrincipal() == 0, "close restored principal");
        require(pool.unassignedAssets() == unassignedCapped, "close restored unassigned");
        require(!engine.degradedMode(), "flat close latched degraded mode");
        int256 deltaFlat = _terminalDelta();
        console2.log("deltaFlat");
        console2.logInt(deltaFlat);
        require(deltaFlat == 0, "flat book still has a terminal delta");
        uint256 pendingFlat = pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets();
        uint256 claimsFlat = engine.totalTraderClaimBalanceAssets();
        uint256 accountedFlat = pool.totalAssets();
        uint256 distributableFlat = accountedFlat > claimsFlat ? accountedFlat - claimsFlat : 0;
        if (pendingFlat < distributableFlat) distributableFlat -= pendingFlat;
        console2.log("distributableFlat", distributableFlat);
        console2.log("capPredicateBinds", pool.unassignedAssets() > distributableFlat ? 1 : 0);
        require(pool.unassignedAssets() < distributableFlat, "cap predicate still binds on the flat book");
        require(distributableFlat - pool.unassignedAssets() > writtenOff, "flat cash does not still contain the written-off recap");
        uint256 poolDropOnClose = poolBeforeClose - token.balanceOf(POOL);
        console2.log("poolDropOnClose", poolDropOnClose);
        console2.log("traderWalletGain", token.balanceOf(trader) - traderBeforeClose);
        require(poolDropOnClose < writtenOff / 10, "close paid out the capped loss");

        uint256 cashBeforeRecovery = token.balanceOf(POOL);
        uint256 accountedBeforeRecovery = pool.totalAssets();
        uint256 supplyAtRecovery = token.totalSupply();
        console2.log("cashBeforeRecovery", cashBeforeRecovery);
        console2.log("accountedBeforeRecovery", accountedBeforeRecovery);

        // Step 6. The next fresh router settle seeds distributable minus the capped tag.
        // Bob's redeem is priced after that seed and pays the excess in USDG.
        uint256 bobBefore = token.balanceOf(bob);
        uint256 bobPaid = _redeemAndCollect(junior, bob);
        _log("recovered");
        console2.log("bobPaid", bobPaid);
        console2.log("unassignedAfter", pool.unassignedAssets());
        console2.log("seniorAfter", pool.seniorPrincipal());
        console2.log("juniorAfter", pool.juniorPrincipal());

        uint256 principalLeft = pool.seniorPrincipal() + pool.juniorPrincipal();
        uint256 principalSeeded = principalLeft + bobPaid;
        uint256 seniorAfter = pool.seniorPrincipal();
        uint256 juniorSeeded = pool.juniorPrincipal() + bobPaid;
        console2.log("principalSeeded", principalSeeded);
        require(pool.unassignedAssets() == unassignedCapped, "recovery restored unassigned");
        require(bobPaid > JUNIOR_DEPOSIT + 1_000e6, "bob was not paid more than his deposit plus 1000 USDG");
        require(token.balanceOf(bob) - bobBefore == bobPaid, "bob wallet is not the redeem");
        require(token.balanceOf(POOL) + bobPaid == cashBeforeRecovery, "pool drop is not bob's redeem");
        require(token.totalSupply() == supplyAtRecovery, "USDG was minted during the recovery settle");

        uint256 expectedSeed = accountedBeforeRecovery - unassignedCapped;
        console2.log("expectedSeed", expectedSeed);
        require(principalSeeded == expectedSeed, "seed is not cash minus capped unassigned");

        uint256 juniorIncrease = juniorSeeded - juniorBefore;
        console2.log("juniorIncrease", juniorIncrease);
        console2.log("seniorAfter", seniorAfter);
        uint256 principalIncrease = principalSeeded - principalBefore;
        console2.log("principalIncrease", principalIncrease);
        require(principalIncrease > writtenOff / 2, "seed did not keep most of the written-off recap");
        require(juniorIncrease > writtenOff / 2, "junior did not receive the written-off recap");
        uint256 bobExcess = bobPaid - JUNIOR_DEPOSIT;
        console2.log("bobExcess", bobExcess);
        require(bobExcess > 1_000e6, "bob captured less than 1000 USDG above his deposit");

        // The owner can assign only the balance that survived the cap. The snapshot is restored,
        // so the paid balances above are the balances that stand.
        uint256 snap = vm.snapshotState();
        uint256 assignable = pool.unassignedAssets();
        vm.prank(pool.owner());
        pool.assignUnassignedAssets(false, assignee);
        uint256 minted = junior.balanceOf(assignee);
        console2.log("assignableAfterCap", assignable);
        console2.log("assigneeShares", minted);
        require(pool.unassignedAssets() == 0, "assign left a balance");
        require(assignable == unassignedCapped, "assign saw a restored balance");
        require(assignable + writtenOff == unassignedBefore, "assignable plus written-off is not the old balance");
        require(minted > 0, "assign minted nothing");
        require(vm.revertToState(snap), "assign snapshot revert failed");
        require(pool.unassignedAssets() == unassignedCapped, "snapshot restore lost the capped balance");
        require(junior.balanceOf(assignee) == 0, "snapshot restore kept assignee shares");
        require(token.balanceOf(bob) - bobBefore == bobPaid, "snapshot restore changed bob");
        require(token.balanceOf(POOL) + bobPaid == cashBeforeRecovery, "snapshot restore changed the pool");
    }

    function _deposit(IVault vault, address account, uint256 amount) internal {
        token.mint(account, amount);
        vm.startPrank(account);
        token.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, account, account);
        vm.stopPrank();
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        _settle(MARK_ENTRY);
        uint256 claimable = vault.claimableDepositRequest(requestId, account);
        require(claimable == amount, "deposit was not fully funded");
        vm.prank(account);
        vault.claimDeposit(requestId, claimable, account, account);
    }

    function _mintAndRecap(uint256 amount) internal {
        address engineOwner = engine.owner();
        token.mint(engineOwner, amount);
        vm.startPrank(engineOwner);
        token.approve(ENGINE, amount);
        engine.recapitalizePool(amount);
        vm.stopPrank();
    }

    function _open(address account, uint256 lots, uint256 margin) internal {
        uint256 bountyBuffer = 1_000e6;
        token.mint(account, margin + bountyBuffer);
        vm.startPrank(account);
        token.approve(CLEARINGHOUSE, type(uint256).max);
        IClearinghouse(CLEARINGHOUSE).deposit(account, margin + bountyBuffer);
        vm.stopPrank();
        _commitAndExecute(account, false, lots * 1e20, margin, ENTRY, MARK_ENTRY);
    }

    function _close(address account, uint256 size) internal {
        _commitAndExecute(account, true, size, 0, ENTRY, MARK_ENTRY);
    }

    function _commitAndExecute(
        address account,
        bool isClose,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        uint256 multiple
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        require(window > 0 && window <= type(uint32).max, "execution window");
        IRouter.OrderRequest memory request = IRouter.OrderRequest({
            clientOrderId: keccak256(abi.encode("F22451", account, isClose, sizeDelta, block.timestamp)),
            side: 1,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: IRouter.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: uint8(LIVE_MODE),
                expectedConfigHash: IBook(router.lifecycleBook()).currentExecutionConfigHash(),
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
        require(bytes8(request.clientOrderId) != bytes8(uint64(0x504c455448455221)), "protocol client id");
        uint64 commitTime = uint64(block.timestamp);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        require(orderId != 0, "commit returned no id");
        vm.roll(block.number + 1);
        this._executeFill(orderId, multiple, commitTime);
    }

    /// @dev Warp inside this call, immediately before `executeOrder`. The oracle reads the EVM clock.
    function _executeFill(uint64 orderId, uint256 multiple, uint64 commitTime) external {
        require(msg.sender == address(this), "fill");
        vm.warp(uint256(commitTime) + 1);
        IRouter.ExecutionResult memory result =
            router.executeOrder(orderId, _payload(multiple, uint256(commitTime) + 1, commitTime));
        console2.log("executeStatus", uint256(result.status));
        console2.log("executePending", uint256(result.pendingReason));
        console2.log("executeTerminal", uint256(result.terminalReason));
        require(uint256(result.status) == 2, "executeOrder did not execute");
    }

    function _queueRedeem(IVault vault, address account) internal returns (uint256 requestId) {
        uint256 unlock = vault.lastDepositTime(account) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        _refresh(MARK_ENTRY);
        uint256 shares = vault.maxRequestRedeem(account);
        require(shares > 0, "nothing to queue");
        vm.prank(account);
        requestId = vault.requestRedeem(shares, account, account);
    }

    function _redeemAndCollect(IVault vault, address account) internal returns (uint256 assetsOut) {
        uint256 unlock = vault.lastDepositTime(account) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        _refresh(MARK_ENTRY);
        uint256 shares = vault.maxRequestRedeem(account);
        require(shares > 0, "nothing to redeem");
        vm.prank(account);
        uint256 requestId = vault.requestRedeem(shares, account, account);
        uint256 beforeBal = token.balanceOf(account);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        _settle(MARK_ENTRY);
        uint256 refundShares = vault.refundableRedeemRequest(requestId, account);
        require(refundShares == 0, "redeem was refunded");
        uint256 claimable = vault.claimableRedeemRequest(requestId, account);
        require(claimable != 0, "redeem funded nothing");
        vm.prank(account);
        vault.claimRedeem(requestId, claimable, account, account);
        assetsOut = token.balanceOf(account) - beforeBal;
    }

    function _settle(uint256 multiple) internal {
        _syncClock();
        router.settleLpEpoch(_payload(multiple, block.timestamp, 0));
    }

    function _refresh(uint256 multiple) internal {
        _syncClock();
        router.updateMarkPrice(_payload(multiple, block.timestamp, 0));
    }

    function _syncClock() internal {
        if (block.timestamp <= uint256(engine.lastMarkTime())) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
    }

    function _warpUntilOracleLive() internal {
        uint256 t = LIVE_TS;
        if (t <= uint256(engine.lastMarkTime())) t = uint256(engine.lastMarkTime()) + 1;
        vm.warp(t);
        for (uint256 i; i < 12; ++i) {
            if (!engine.isOracleFrozen()) break;
            vm.warp(block.timestamp + 3 days);
        }
        require(!engine.isOracleFrozen(), "oracle stayed frozen");
        for (uint256 i; i < 8; ++i) {
            if (!engine.isFadWindow()) break;
            vm.warp(block.timestamp + 15 minutes);
        }
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "session is not live");
    }

    /// @dev Move forward to `target`, then walk out of a frozen or FAD window if the landing time is inside one.
    function _advanceLive(uint256 target) internal {
        require(target >= block.timestamp, "live clock only moves forward");
        if (block.timestamp < target) vm.warp(target);
        for (uint256 i; i < 12; ++i) {
            if (!engine.isOracleFrozen()) break;
            vm.warp(block.timestamp + 3 days);
        }
        require(!engine.isOracleFrozen(), "oracle stayed frozen");
        for (uint256 i; i < 8; ++i) {
            if (!engine.isFadWindow()) break;
            vm.warp(block.timestamp + 15 minutes);
        }
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "session is not live");
    }

    function _payload(uint256 which, uint256 publishTime, uint256 prevPublishTime)
        internal
        view
        returns (bytes[] memory data)
    {
        require(which == MARK_ENTRY || which == MARK_STRESS, "unknown mark");
        int64[] memory raws = which == MARK_STRESS ? rawAtStress : rawAtEntry;
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](FEEDS);
        for (uint256 i; i < FEEDS; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: raws[i],
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _loadBasket() internal {
        feedIds = new bytes32[](FEEDS);
        weights = new uint256[](FEEDS);
        bases = new uint256[](FEEDS);
        inverted = new bool[](FEEDS);
        rawAtEntry = new int64[](FEEDS);
        rawAtStress = new int64[](FEEDS);
        uint256 weightSum;
        for (uint256 i; i < FEEDS; ++i) {
            feedIds[i] = oracle.pythFeedIds(i);
            weights[i] = oracle.quantities(i);
            bases[i] = oracle.basePrices(i);
            inverted[i] = oracle.inversions(i);
            weightSum += weights[i];
            require(feedIds[i] != bytes32(0) && bases[i] != 0 && weights[i] != 0, "bad basket component");
        }
        (bool extra,) = address(oracle).staticcall(abi.encodeWithSignature("pythFeedIds(uint256)", FEEDS));
        require(!extra, "basket has more than six feeds");
        require(weightSum == 1e18, "basket weights do not sum to 1e18");
        for (uint256 i; i < FEEDS; ++i) {
            rawAtEntry[i] = _componentRaw(bases[i], weights[i], ENTRY, inverted[i]);
            rawAtStress[i] = _componentRaw(bases[i], weights[i], STRESS, inverted[i]);
        }
        require(_preview(rawAtEntry) == ENTRY, "entry basket is not 1.00");
        require(_preview(rawAtStress) == STRESS, "stress basket is not 1.10");
    }

    function _installPythMock() internal {
        vm.etch(pyth, address(new MockPyth()).code);
        // The etched mock keeps the previous Pyth storage. Zero only this mock's scalar
        // slots so its fee and counters start clean. Protocol storage is not written.
        for (uint256 slot = 3; slot <= 10; ++slot) {
            vm.store(pyth, bytes32(slot), bytes32(0));
        }
        for (uint256 i; i < FEEDS; ++i) {
            MockPyth(pyth).setPrice(feedIds[i], int64(1), uint64(0), int32(-8), 0);
        }
        require(MockPyth(pyth).getUpdateFee(new bytes[](1)) == 0, "mock fee was not cleared");
    }

    function _preview(int64[] memory raws) internal view returns (uint256 price) {
        for (uint256 i; i < FEEDS; ++i) {
            uint256 norm = inverted[i] ? _invert(uint256(uint64(raws[i]))) : uint256(uint64(raws[i]));
            price += (norm * weights[i]) / (bases[i] * SCALE);
        }
    }

    function _componentRaw(uint256 basePrice, uint256 quantity, uint256 basketPrice, bool inverse)
        internal
        pure
        returns (int64)
    {
        uint256 targetNorm = basePrice * basketPrice / ENTRY;
        require(targetNorm * ENTRY == basePrice * basketPrice, "component target truncates");
        uint256 ideal = (targetNorm * quantity) / (basePrice * SCALE);
        if (!inverse) {
            require(_weighted(targetNorm, basePrice, quantity) == ideal, "direct component missed its weight");
            return int64(uint64(targetNorm));
        }
        uint256 guess = (10 ** 34) / (targetNorm * 1e18);
        if (guess == 0) guess = 1;
        for (uint256 distance; distance <= 250_000; ++distance) {
            if (guess > distance) {
                uint256 lower = guess - distance;
                uint256 lowerNorm = _invert(lower);
                if (lowerNorm == targetNorm || _weighted(lowerNorm, basePrice, quantity) == ideal) {
                    return int64(uint64(lower));
                }
            }
            uint256 upper = guess + distance;
            uint256 upperNorm = _invert(upper);
            if (upperNorm == targetNorm || _weighted(upperNorm, basePrice, quantity) == ideal) {
                return int64(uint64(upper));
            }
        }
        revert("no inverse raw keeps the basket weight");
    }

    function _weighted(uint256 norm, uint256 basePrice, uint256 quantity) internal pure returns (uint256) {
        return (norm * quantity) / (basePrice * SCALE);
    }

    function _invert(uint256 price) internal pure returns (uint256) {
        uint256 scaledPrecision = 10 ** 34;
        uint256 scaledInverse = (scaledPrecision + (price / 2)) / price;
        return scaledInverse / 1e18;
    }

    function _terminalDelta() internal view returns (int256) {
        IEngine.Nav memory nav = engine.terminalNavSnapshot();
        return nav.terminalLpPriceDeltaAssets;
    }

    function _log(string memory tag) internal view {
        console2.log(tag);
        console2.log("  ts", block.timestamp);
        console2.log("  mark", engine.lastMarkPrice());
        console2.log("  markTime", uint256(engine.lastMarkTime()));
        console2.log("  cash", token.balanceOf(POOL));
        console2.log("  accounted", pool.totalAssets());
        console2.log("  seniorP", pool.seniorPrincipal());
        console2.log("  juniorP", pool.juniorPrincipal());
        console2.log("  hwm", pool.seniorHighWaterMark());
        console2.log("  unassigned", pool.unassignedAssets());
        console2.log("  pendRecap", pool.pendingRecapitalizationAssets());
        console2.log("  pendRev", pool.pendingTradingRevenueAssets());
        console2.log("  claims", engine.totalTraderClaimBalanceAssets());
        console2.log("  free", pool.getFreeAssets());
        console2.log("  bobShares", junior.balanceOf(bob));
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
    function owner() external view returns (address);
    function juniorPrincipal() external view returns (uint256);
    function seniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function pendingTradingRevenueAssets() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function getFreeAssets() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external pure returns (uint256);
    function assignUnassignedAssets(bool toSenior, address receiver) external;
}

interface IVault {
    function requestDeposit(uint256 assets, address controller, address owner) external returns (uint256 requestId);
    function claimableDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimDeposit(uint256 requestId, uint256 assets, address receiver, address controller)
        external
        returns (uint256 shares);
    function balanceOf(address account) external view returns (uint256);
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
    struct Nav {
        uint32 markPrice;
        uint64 markTime;
        int256 terminalLpPriceDeltaAssets;
        uint256 totalTraderClaimsAssets;
        uint256 maxDirectionalLiabilityAssets;
        uint64 bookVersion;
        bool hasOpenPositions;
        bool degradedMode;
    }

    function owner() external view returns (address);
    function orderRouter() external view returns (address);
    function degradedMode() external view returns (bool);
    function CAP_PRICE() external view returns (uint256);
    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function isOracleFrozen() external view returns (bool);
    function isFadWindow() external view returns (bool);
    function totalTraderClaimBalanceAssets() external view returns (uint256);
    function recapitalizePool(uint256 amountAssets) external;
    function terminalNavSnapshot() external view returns (Nav memory);
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

interface IOracle {
    function pyth() external view returns (address);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function quantities(uint256 index) external view returns (uint256);
    function basePrices(uint256 index) external view returns (uint256);
    function inversions(uint256 index) external view returns (bool);
}

interface IBook {
    function currentExecutionConfigHash() external view returns (bytes32);
}

interface IRouter {
    struct ExecutionBounds {
        uint64 submitBy;
        uint32 executionWindowSeconds;
        uint8 allowedExecutionModes;
        bytes32 expectedConfigHash;
        uint256 maxExecutionBountyAssets;
        uint256 maxExecutionNotionalAssets;
        uint256 maxGrossAccountDebitAssets;
        uint256 maxActionChargeAssets;
        uint256 maxExplicitFeesAssets;
        uint256 maxPostPositionSize;
        uint256 minPostSettlementBalanceAssets;
        uint256 minPostPositionEquityAssets;
        uint32 maxPostLeverageBps;
    }

    struct OrderRequest {
        bytes32 clientOrderId;
        uint8 side;
        uint256 sizeDelta;
        uint256 marginDelta;
        uint256 targetPrice;
        bool isClose;
        ExecutionBounds bounds;
    }

    struct ExecutionResult {
        uint64 orderId;
        uint8 status;
        uint8 terminalReason;
        uint8 pendingReason;
        bytes32 receiptHash;
    }

    function pletherOracle() external view returns (address);
    function lifecycleBook() external view returns (address);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function nextCommitId() external view returns (uint64);
    function nextExecuteId() external view returns (uint64);
    function commitOrder(OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (ExecutionResult memory result);
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
}

```

