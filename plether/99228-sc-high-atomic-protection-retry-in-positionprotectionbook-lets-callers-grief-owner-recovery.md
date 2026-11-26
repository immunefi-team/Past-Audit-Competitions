# \#99228 [SC-High] Atomic Protection Retry in PositionProtectionBook Lets Callers Grief Owner Recovery

**Submitted on Nov 12th 2026 at 13:41:02 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99228
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PositionProtectionBook` permits any address to retry a failed protection close. When a close fails its retained financial limits, a caller can execute the attempt and retry in one transaction, returning the protection to `Triggered` before the owner can make a standalone amendment.

Repeating that cycle forces the owner to win a child-execution race before amending the limits. Until the owner does, a standalone amendment and ordinary close revert while the position remains open and exposed to price movement.

## Vulnerability Details

**Close limits and the owner's recovery.** By design, each position protection retains the owner's close financial limits (execution notional, gross account debit, action charges and explicit fees) and applies them to every triggered or retried close attempt (CANONICAL_ENTRYPOINTS.md:121-126). `Triggered` means one live close attempt, and `Latched` means the trigger remains active with no live attempt and may be retried permissionlessly. The recovery sequence gives the owner the interval after a failed attempt relatches and before a fresh retry to amend the retained limits. A live child keeps its original request (IPositionProtectionActions.sol:32-33). The conservative keeper procedure investigates persistent planner or constraint failures before retry and avoids continuous requeueing of the same failing attempt (DEPLOYMENT.md:420-421). When each close continues to exceed the owner's retained financial limits, amendment is an owner-controlled remedy for the repeated close failures.

`amendLatchedPositionProtectionCloseLimits` checks both conditions:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327

```solidity
        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
            revert OrderRouter__PendingOrdersExist();
        }
```

**Retry restores the blocking state.** `retryPositionProtectionClose` loads the protection without an owner check (`_activeProtection` instead of `_ownedActiveProtection`, PositionProtectionBook.sol:983-1000). It requires the same `Latched` status and empty account queue, commits a new close attempt for the protected account, and records it as the live attempt, which returns the protection to `Triggered` (PositionProtectionBook.sol:564-578):

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405

```solidity
        StoredProtection storage protection = _activeProtection(protectionId);
        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
        _requireMatchingPosition(protection);

        IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
        if (router.pendingOrderCounts(protection.account) != 0) {
            revert OrderRouter__PendingOrdersExist();
        }
```

**The execute-and-retry cycle.** Order execution is also permissionless. In one transaction, a caller executes the live close attempt through `OrderRouter.executeOrder`; the retained gross-debit limit makes it fail, and the lifecycle callback relatches the protection (PositionProtectionBook.sol:637-668) and removes the failed attempt from the account queue. Before the transaction ends, the same caller calls `retryPositionProtectionClose`, which finds `Latched` status and an empty queue and commits the next attempt. The caller's transaction leaves `Triggered` with one pending attempt, in which a standalone amendment reverts. If the owner or another keeper executes the live child first, it leaves `Latched` until a retry. The owner can amend during that interval, including by executing the child and amending in one smart-account transaction. During a frozen-market window, the Router skips the SameBlock and MevBoundary timing checks (OrderRouterExecutionSidecar.sol:445-451); a same-block batch can execute a fresh child if the remaining execution conditions hold.

**Ordinary close is rejected while the protection remains active.** The protection lock rejects the owner's ordinary open and close orders while any protection is active:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L314

Cancellation accepts only `PendingOpen` or `Armed` protection, and cancel-and-close accepts only `Armed` protection (PositionProtectionBook.sol:414-419, :345-347). Queued intents have no manual cancellation (SECURITY.md:1345). The position's PnL pledge remains reserved while the position is open. If the owner or another keeper executes the failing child before the caller, the protection relatches and the owner can amend before the next retry.

**Caller authority.** The caller needs no role. It uses only `OrderRouter.executeOrder` and `PositionProtectionBook.retryPositionProtectionClose` with valid price updates, once in each attempt's execution window.

The retained close limits and the owner amendment were added in commit eadc5c58 (Plether-Fi/plether-core#124) so that the trader's financial ceilings stay effective in triggered and retried attempts. They remain effective in every retry, and a ceiling that prevents execution is recoverable through the owner amendment after relatch. The defect reported here is introduced by that change: the amendment requires `Latched`, which the permissionless retry can consume and replace with `Triggered` atomically before the owner changes the ceilings.

## Impact Details

Severity: Medium, Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).

A caller who executes a failing stop-loss child and retries it in one transaction makes the owner's standalone amendment and ordinary close revert. Repeating the cycle forces a child to be executed before the caller's next retry, followed by the owner's amendment, or requires the owner to combine execution and amendment in a smart-account transaction. Until recovery, the position remains open, its PnL pledge stays reserved, and the owner remains exposed to price movement.

In the release deployment fixture with release parameters, the honest sequence and the attack start from the same state: a leveraged 200,000-unit LONG with a stop-loss at 1.22 and a gross account debit limit, and a gap to 1.25 at which the close attempt fails that limit (parameters in the Proof of Concept). In the honest sequence the owner amends the limits after the failed attempt and exits at once. In the attack sequence, an address with no role executes each child first and retries it once per minute, with no competing order execution.

| | Honest sequence | Attack |
|---|---|---|
| Attacker-first execute-and-retry cycling | 0 s | 86,400 s (1,440 cycles) |
| Standalone amendment and ordinary close after a child fails | owner amends and exits | revert after attacker cycles |
| PnL pledge locked at the end | 0.000000 | 19,376.000576 |
| Maximum immediate withdrawal | 29,803.596530 | 19,999.600000 |
| Additional USDG unavailable for immediate withdrawal | 0.000000 | 9,803.996530 |
| Owner balance after exiting at the unchanged price | 29,803.596530 | 29,796.932862 (6.663668 less) |
| USDG gained by the caller at the unchanged price | N/A | 0.00 |

At the end of this attacker-first sequence, 9,803.996530 USDG is additionally unavailable for immediate withdrawal. The 19,376.000576 USDG PnL pledge is the gross reserved amount, not the incremental amount denied by the attack; 19,999.600000 USDG remains withdrawable. At an unchanged price, the eventual balance difference is 6.663668 USDG from carry and other time-dependent settlement.

The following additional losses require an adverse basket movement during the lock. From the same starting state, the honest sequence exits at 1.25:

| Basket during the 24-hour lock | Owner's final USDG, honest sequence | Owner's final USDG, attack | Difference |
|---|---|---|---|
| Unchanged at 1.25 | 29,803.60 | 29,796.93 | 6.66 |
| Moves 1% to 1.2625 | 29,803.60 | 27,295.93 | 2,507.66 |
| Reaches the position's liquidation price at the start of the lock, 1.29558441; the caller then liquidates the position and receives the 120.00 USDG liquidation bounty | 29,803.60 | 20,252.10 | 9,551.50 |

The pending close attempt does not prevent liquidation. The position's equity at liquidation is 252.45 USDG, above the 240.00 USDG liquidation charge, so the liquidation leaves no bad debt.

Prerequisites:
- A protection whose close attempt exceeds its retained limits. This occurs when the price moves past the trigger by more than the owner's limits allow, for example in a gap through a stop-loss. Repeated failure requires the price to stay beyond those limits; a retrace may permit the next child to close.
- In the measured sequence, the caller executes each close attempt before the owner or another keeper and retries in the same transaction. Each attempt is executed near the end of its 60-second execution window, so for most of each window the owner or an ordinary keeper can execute it first. An owner that executes the live child and amends before the next retry ends the cycle, atomically from a smart account or with back-to-back transactions that win that race. Against an acting owner, the caller must execute each child as soon as it becomes executable, which does not require observing the owner's transaction. The test does not measure the cost or probability of maintaining attacker-first ordering against an acting owner.
- Adding margin prevents the liquidation shown above but does not by itself make a standalone amendment succeed after the next attacker cycle. Adding margin and opening an opposite 200,000-unit SHORT from a second account in the first 2 minutes reduces the difference in the drift to the liquidation price to 852.37 USDG.
- One cycle uses an average of 1,237,626 gas with warm storage in the 24-hour test and 1,543,695 gas in Forge isolated-transaction mode. The latter is not an Arbitrum One fee measurement: the PoC's `MockPyth` accepts one empty update blob and returns a zero fee, while deployed execution requires signed updates for all 6 feeds and pays transaction and data fees. The test runs 1,440 cycles at one cycle per minute. The oracle accepts a publication only after the attempt's commit time (PletherOracle.sol:610), but that condition does not guarantee a complete new signed basket at any fixed cadence. At an unchanged price the caller gains no USDG.
- No privileged role, incorrect oracle value or unusual configuration is involved. Every call in the cycle is public; the fixture supplies coherent 6-feed `MockPyth` prices. Live execution requires authenticated signed updates as described above.

## Recommendation

The owner's recovery requires a state that the permissionless retry removes within the same transaction. Allow the owner to amend the retained close limits while the protection is `Triggered`, leaving the request of the already pending close attempt unchanged, so that the next retry copies the amended limits, and keep the existing empty-queue requirement for amendments in `Latched` status. Permissionless retry then remains available, and the owner's recovery no longer depends on winning an execution race.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Protection close limits, trigger irreversibility and amendment: `packages/perps/CANONICAL_ENTRYPOINTS.md:121-126`, `:142-148`
- Permissionless retry of a latched protection: `packages/perps/SECURITY.md:547-552`
- Owner amendment after relatch and immutable live child: `packages/perps/src/interfaces/IPositionProtectionActions.sol:32-33`
- Persistent-failure retry handling: `packages/perps/DEPLOYMENT.md:420-421`
- Frozen-market execution checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L445-L451
- Release parameters: `packages/perps/DEPLOYMENT.md` (Initial release parameters) and `script/DeployPerpsArbitrumSepolia.s.sol`
- Change that added the retained close limits and the owner amendment: Plether-Fi/plether-core#124 (commit eadc5c58)

## Proof of Concept

## Proof of Concept

The tests deploy the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge through `ReleaseFixture.sol` (source at the end of this section). In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The PoC consists of 2 test files. Steps in `test_attack_retry_loop_locks_owner_24h` (`ProtectionRetryLock24h.t.sol`), all through public entry points:
1. A Junior LP provides liquidity. Alice opens a 200,000-unit LONG with 20,000 USDG margin and a stop-loss at 1.22. The protection limits gross account debit to 5,000 USDG.
2. The basket moves to 1.25. The close attempt fails the retained limit and relatches (lifecycle status 3 is Failed, terminal reason 8 is ConstraintViolation).
3. An address with no role uses a helper contract that, in one transaction, executes the live close attempt with `executeOrder` and calls `retryPositionProtectionClose` after the attempt fails.
4. The cycle repeats 1,440 times, once in each attempt's execution window, with the caller always first and no competing keeper execution. At checkpoints and after the final cycle, the test checks that standalone owner amendment, cancellation, cancel-and-close and ordinary close calls revert.
5. After 86,400 seconds, the test checks that the full position is still open, the protection is still `Triggered`, those standalone owner calls still revert and the full balance cannot be withdrawn, and logs the PnL pledge. A separate counter-check executes the live child from an owner wallet and escapes after winning one execution race.

`test_control_vs_attack_same_snapshot` runs the honest sequence and the attack from one snapshot and compares the owner's maximum immediate withdrawal and final balance. The same file contains counter-checks: the owner's amendment succeeds after an honest execution leaves the protection `Latched`, separate owner transactions sent after an atomic cycle revert, and an owner that batches execution and amendment escapes by winning one execution race. `test_execute_and_retry_cycle_gas`, run with `--isolate`, measures the gas of one cycle as its own transaction, with cold account and storage access and its intrinsic cost.

The owner-batch counter-check runs in the live-market execution state. The tests do not exercise an owner batch in a frozen-market window or establish that the remaining execution conditions always hold there.

`ProtectionRetryLockLiquidation.t.sol` uses the same setup, with a fresh address performing the cycle (1,440 cycles in the drift and hedge tests), and compares the owner's final USDG with an immediate amendment and exit taken from the same snapshot and measured at the same final time:
1. The basket moves linearly from 1.25 to the position's liquidation price, 1.29558441, over the 1,440 cycles. Two blocks (4 seconds) after the last cycle, the same address calls `executeLiquidation` for Alice. The pending close attempt ends Failed with terminal reason AccountLiquidated, and the address withdraws its liquidation bounty.
2. The basket moves linearly from 1.25 to 1.2625 (1%) over the 1,440 cycles. The caller then stops, the last attempt fails, and Alice amends and exits at 1.2625.
3. During the lock, a new protection, an ordinary open and a full withdrawal revert. `addMargin` succeeds and raises the liquidation price; liquidation at the former liquidation price then reverts with `CfdEngine__PositionIsSolvent`, and the next cycle still locks the owner out. A further test opens an opposite hedge from a second account during the lock.

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `ReleaseFixture.sol`, `ProtectionRetryLock24h.t.sol` and `ProtectionRetryLockLiquidation.t.sol` (sources at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/ProtectionRetryLock24h.t.sol --gas-limit 30000000000 -vv
forge test --isolate --match-path test/poc/ProtectionRetryLock24h.t.sol --match-test test_execute_and_retry_cycle_gas -vv
forge test --match-path test/poc/ProtectionRetryLockLiquidation.t.sol --gas-limit 30000000000 -vv
```

Excerpt from the output of the first command for `test_attack_retry_loop_locks_owner_24h` and `test_control_vs_attack_same_snapshot` (the release deployment script's 56 log lines after `Logs:` omitted):

```
[PASS] test_attack_retry_loop_locks_owner_24h() (gas: 1482171116)
Logs:
  cycle 0: child status / terminalReason 3 8
  attack: attacker-first sequence elapsed seconds 86400
  attack: cycles / avg gas / max gas 1440 1237626 1243394
  attack: alice clearinghouse balance before/after 24 h 39615600576 39615600576
  attack: alice pnl pledge (locked)  19376000576
  attacker wallet / keeper USDG gained 0 0
```

```
[PASS] test_control_vs_attack_same_snapshot() (gas: 1494968995)
Logs:
  same-snapshot: locked settlement / total locked / free 39615600576 19616000576 19999600000
  same-snapshot: locked maximum withdraw / immediate control exit 19999600000 29803596530
  same-snapshot: immediate control exit minus locked maximum withdraw 9803996530
  same-snapshot: alice balance at latch 39615600576
  same-snapshot: control exit balance / exit time 29803596530 1709535652
  same-snapshot: attack exit balance / lock seconds 29796932862 86400
  same-snapshot: extra cost to alice from the lock (carry etc.), atoms 6663668
  same-snapshot: attacker USDG gained 0
```

Excerpt from the output of the second command (deployment lines omitted):

```
[PASS] test_execute_and_retry_cycle_gas() (gas: 21866172)
Logs:
  execute-and-retry cycle gas: cycles / min / max 8 1543695 1543695
```

Excerpt from the output of the third command for the drift to the liquidation price, the 1% drift and the hedge (deployment lines omitted):

```
[PASS] test_drift_to_liquidation_while_protection_close_pending() (gas: 1490742357)
Logs:
  position margin (PnL pledge) / trader claim / entry cost atoms 19376000576 0 240000000000
  liquidation price: closed form without pending carry / canonical lens predicate 129558442 129558441
  liquidation price after 24 h (canonical lens predicate) 129555112
  Alice before liquidation: settlement / PnL pledge / liquidation reserve 39615600576 19376000576 240000000
  Alice before liquidation: free settlement 19999600000
  liquidation preview: equity / liquidation charge / keeper bounty 252451978 240000000 120000000
  liquidation preview: LP liquidation fee / settlement retained / immediate payout 120000000 20252051978 0
  immediate-exit control: exit time / balance 1709535652 29803596530
  locked sequence: cycles / total gas / liquidation gas 1440 1782032499 682742
  locked sequence: liquidation time / price 1709622040 129558441
  Alice final USDG: immediate-exit control / locked then liquidated 29803596530 20252095978
  Alice realized loss from the lock 9551500552
  HousePool totalAssets: control / locked 1010102003470 1019533504022
  retry caller USDG before / after (keeper bounty) 0 120000000
```

```
[PASS] test_mild_drift_realized_loss() (gas: 1485904682)
Logs:
  mild drift: start / end basket price 125000000 126250000
  mild drift: cycles / total gas / lock seconds 1440 1782030045 86409
  Alice final USDG: immediate-exit control / locked then exit 29803596530 27295932862
  Alice realized loss from the lock 2507663668
  HousePool totalAssets: control / locked 1010102003470 1012609667138
```

```
[PASS] test_owner_hedge_bounds_loss() (gas: 1491058097)
Logs:
  hedge: Alice LONG account / hedge account final USDG 20053878173 13897344415
  hedge: combined net of hedge deposit / immediate-exit control 28951222588 29803596530
  hedge: residual loss against control 852373942
```

### Source files

#### `ReleaseFixture.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Deploys the release contracts with script/DeployPerpsArbitrumSepolia.s.sol inside Forge.
// Pyth, USDG and ArbSys are replaced by local test components; in-scope contracts are unmodified.

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test, console2} from "forge-std/Test.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract ReleaseFixtureArbSys {

  function arbBlockNumber() external view returns (uint256) {
    return block.number;
  }

}

abstract contract ReleaseFixture is Test {

  address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
  uint256 internal constant DEPLOYER_KEY = 0xA11CE;
  uint256 internal constant MONDAY_MORNING = 1_709_532_000; // Mon 2024-03-04 06:00 UTC (live market)
  uint256 internal constant BASKET_1_20 = 120_000_000; // normalized basket mark 1.20 (8 decimals)
  uint8 internal constant MODE_LIVE = 1;
  uint8 internal constant MODE_FAD = 2;
  uint8 internal constant MODE_FROZEN = 4;
  uint8 internal constant MODE_ANY = 7;

  DeployPerpsArbitrumSepolia.DeployedContracts internal d;
  address internal owner;
  address internal keeper = makeAddr("keeper");
  uint256 internal clientNonce;

  function setUp() public virtual {
    vm.chainId(421_614);
    vm.warp(MONDAY_MORNING);
    ReleaseFixtureArbSys arbSys = new ReleaseFixtureArbSys();
    vm.etch(address(100), address(arbSys).code);
    MockPyth pyth = new MockPyth();
    vm.etch(RELEASE_PYTH, address(pyth).code);
    vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
    d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
    owner = vm.addr(DEPLOYER_KEY);

    // Permanent 1 USDG seeds per tranche (release values), then activation.
    d.settlementToken.mint(owner, 2e6);
    vm.startPrank(owner);
    d.settlementToken.approve(address(d.housePool), 2e6);
    d.housePool.initializeSeedPosition(false, 1e6, owner);
    d.housePool.initializeSeedPosition(true, 1e6, owner);
    d.housePool.activateTrading();
    vm.stopPrank();

    _setBasketPrices(BASKET_1_20);
    d.router.updateMarkPrice(_updateData());
  }


  function _oracle() internal view returns (PletherOracle) {
    return PletherOracle(d.pletherOracle);
  }

  /// @dev Sets all six feeds so the normalized basket equals `basketPrice`, published at the current block time.
  function _setBasketPrices(
    uint256 basketPrice
  ) internal {
    _setBasketPricesAt(basketPrice, vm.getBlockTimestamp());
  }

  function _setBasketPricesAt(
    uint256 basketPrice,
    uint256 publishTime
  ) internal {
    for (uint256 i; i < 6; ++i) {
      _setFeed(i, basketPrice, publishTime);
    }
  }

  /// @dev Sets one component so that it alone would give normalized basket `basketPrice`, at `publishTime`.
  function _setFeed(
    uint256 i,
    uint256 basketPrice,
    uint256 publishTime
  ) internal {
    PletherOracle oracle = _oracle();
    uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
    uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
    MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
  }

  /// @dev Same as _setFeed with an explicit confidence (in feed units).
  function _setFeedWithConf(
    uint256 i,
    uint256 basketPrice,
    uint64 conf,
    uint256 publishTime
  ) internal {
    PletherOracle oracle = _oracle();
    uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
    uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
    MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), conf, int32(-8), publishTime);
  }

  /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
  function _updateData() internal pure returns (bytes[] memory data) {
    data = new bytes[](1);
    data[0] = "";
  }

  function _refreshMark(
    uint256 basketPrice
  ) internal {
    _setBasketPrices(basketPrice);
    vm.prank(keeper);
    d.router.updateMarkPrice(_updateData());
  }


  function _warp(
    uint256 dt
  ) internal {
    vm.warp(vm.getBlockTimestamp() + dt);
    vm.roll(vm.getBlockNumber() + 1 + dt / 4);
  }

  function _warpTo(
    uint256 t
  ) internal {
    uint256 nowTs = vm.getBlockTimestamp();
    require(t >= nowTs, "warp back");
    vm.roll(vm.getBlockNumber() + 1 + (t - nowTs) / 4);
    vm.warp(t);
  }


  function _mint(
    address to,
    uint256 amount
  ) internal {
    d.settlementToken.mint(to, amount);
  }

  function _vault(
    bool senior
  ) internal view returns (TrancheVault) {
    return senior ? d.seniorVault : d.juniorVault;
  }

  function _requestDeposit(
    bool senior,
    address lp,
    uint256 assets
  ) internal returns (uint256 requestId) {
    TrancheVault v = _vault(senior);
    _mint(lp, assets);
    vm.startPrank(lp);
    d.settlementToken.approve(address(v), assets);
    requestId = v.requestDeposit(assets, lp, lp);
    vm.stopPrank();
  }

  /// @dev Warps to the request's epoch start (if not past) and settles through the Router's atomic oracle route.
  function _settleEpoch(
    uint256 requestId,
    uint256 basketPrice
  ) internal {
    uint256 start = d.housePool.lpEpochStart(requestId);
    if (vm.getBlockTimestamp() < start) {
      _warpTo(start);
    }
    _setBasketPrices(basketPrice);
    vm.prank(keeper);
    d.router.settleLpEpoch(_updateData());
  }

  function _claimDeposit(
    bool senior,
    address lp,
    uint256 requestId
  ) internal returns (uint256 shares) {
    TrancheVault v = _vault(senior);
    uint256 claimable = v.claimableDepositRequest(requestId, lp);
    vm.prank(lp);
    shares = v.claimDeposit(requestId, claimable, lp, lp);
  }

  /// @dev Request, settle at `basketPrice`, claim. Returns shares.
  function _lpDeposit(
    bool senior,
    address lp,
    uint256 assets,
    uint256 basketPrice
  ) internal returns (uint256 shares) {
    uint256 requestId = _requestDeposit(senior, lp, assets);
    _settleEpoch(requestId, basketPrice);
    shares = _claimDeposit(senior, lp, requestId);
  }

  function _requestRedeem(
    bool senior,
    address lp,
    uint256 shares
  ) internal returns (uint256 requestId) {
    vm.prank(lp);
    requestId = _vault(senior).requestRedeem(shares, lp, lp);
  }

  function _claimRedeem(
    bool senior,
    address lp,
    uint256 requestId
  ) internal returns (uint256 assets) {
    TrancheVault v = _vault(senior);
    uint256 shares = v.claimableRedeemRequest(requestId, lp);
    vm.prank(lp);
    assets = v.claimRedeem(requestId, shares, lp, lp);
  }


  function _fundTrader(
    address trader,
    uint256 assets
  ) internal {
    _mint(trader, assets);
    vm.startPrank(trader);
    d.settlementToken.approve(address(d.clearinghouse), assets);
    d.clearinghouse.depositMargin(assets);
    vm.stopPrank();
  }

  /// @dev Permissive bounds: no practical ceilings (zero means zero, so ceilings use max values).
  function _bounds(
    uint8 modes
  ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
    b.submitBy = uint64(vm.getBlockTimestamp() + 60);
    b.executionWindowSeconds = 60;
    b.allowedExecutionModes = modes;
    b.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
    b.maxExecutionBountyAssets = type(uint256).max;
    b.maxExecutionNotionalAssets = type(uint256).max;
    b.maxGrossAccountDebitAssets = type(uint256).max;
    b.maxActionChargeAssets = type(uint256).max;
    b.maxExplicitFeesAssets = type(uint256).max;
    b.maxPostPositionSize = type(uint256).max;
    b.minPostSettlementBalanceAssets = 0;
    b.minPostPositionEquityAssets = 0;
    b.maxPostLeverageBps = type(uint32).max;
  }

  function _nextClientId() internal returns (bytes32) {
    return bytes32(uint256(keccak256(abi.encode("poc", ++clientNonce))));
  }

  /// @param size synthetic size in 1e18 units; must be a multiple of 100e18 (positionSizeQuantum 1e20).
  function _openReq(
    CfdTypes.Side side,
    uint256 size,
    uint256 margin,
    uint256 targetPrice
  ) internal returns (OrderV3Types.OrderRequest memory r) {
    r.clientOrderId = _nextClientId();
    r.side = side;
    r.sizeDelta = size;
    r.marginDelta = margin;
    r.targetPrice = targetPrice;
    r.isClose = false;
    r.bounds = _bounds(MODE_LIVE | MODE_FAD);
  }

  function _closeReq(
    CfdTypes.Side side,
    uint256 size,
    uint256 targetPrice
  ) internal returns (OrderV3Types.OrderRequest memory r) {
    r.clientOrderId = _nextClientId();
    r.side = side;
    r.sizeDelta = size;
    r.marginDelta = 0;
    r.targetPrice = targetPrice;
    r.isClose = true;
    r.bounds = _bounds(MODE_ANY);
  }

  function _commit(
    address trader,
    OrderV3Types.OrderRequest memory r
  ) internal returns (uint64 orderId) {
    vm.prank(trader);
    orderId = d.router.commitOrder(r);
  }

  /// @dev Advances 7 s / 2 blocks (past the commit tick), publishes `basketPrice`, executes as `keeper`.
  function _execute(
    uint64 orderId,
    uint256 basketPrice
  ) internal returns (OrderV3Types.ExecutionResult memory result) {
    vm.warp(vm.getBlockTimestamp() + 7);
    vm.roll(vm.getBlockNumber() + 2);
    _setBasketPrices(basketPrice);
    vm.prank(keeper);
    result = d.router.executeOrder(orderId, _updateData());
  }

  function _openPosition(
    address trader,
    CfdTypes.Side side,
    uint256 size,
    uint256 margin,
    uint256 basketPrice
  ) internal returns (OrderV3Types.ExecutionResult memory result) {
    // Slippage targets: a LONG gains when the basket falls, so it opens at a price >= target and closes at <= target.
    uint256 target = side == CfdTypes.Side.LONG ? 1 : basketPrice * 2;
    uint64 id = _commit(trader, _openReq(side, size, margin, target));
    result = _execute(id, basketPrice);
  }

  function _closePosition(
    address trader,
    CfdTypes.Side side,
    uint256 size,
    uint256 basketPrice
  ) internal returns (OrderV3Types.ExecutionResult memory result) {
    uint256 target = side == CfdTypes.Side.LONG ? basketPrice * 2 : 1;
    uint64 id = _commit(trader, _closeReq(side, size, target));
    result = _execute(id, basketPrice);
  }

  function _requireExecuted(
    OrderV3Types.ExecutionResult memory r,
    string memory label
  ) internal pure {
    if (r.status != OrderV3Types.LifecycleStatus.Executed) {
      console2.log("order not executed:", label);
      console2.log("  status", uint256(uint8(r.status)));
      console2.log("  terminalReason", uint256(uint8(r.terminalReason)));
      console2.log("  pendingReason", uint256(uint8(r.pendingReason)));
      revert(label);
    }
  }

  function _liquidate(
    address account,
    uint256 basketPrice
  ) internal {
    _setBasketPrices(basketPrice);
    vm.prank(keeper);
    d.router.executeLiquidation(account, _updateData());
  }


  function _poolAssets() internal view returns (uint256) {
    return d.housePool.totalAssets();
  }

  function _usdgOf(
    address a
  ) internal view returns (uint256) {
    return d.settlementToken.balanceOf(a);
  }

  function _logState(
    string memory label
  ) internal view {
    console2.log("==", label);
    console2.log("  pool totalAssets      ", d.housePool.totalAssets());
    console2.log("  pool raw USDG         ", _usdgOf(address(d.housePool)));
    console2.log("  seniorPrincipal       ", d.housePool.seniorPrincipal());
    console2.log("  juniorPrincipal       ", d.housePool.juniorPrincipal());
    console2.log("  clearinghouse USDG    ", _usdgOf(address(d.clearinghouse)));
    console2.log("  seniorVault USDG      ", _usdgOf(address(d.seniorVault)));
    console2.log("  juniorVault USDG      ", _usdgOf(address(d.juniorVault)));
    console2.log("  total trader claims   ", d.engine.totalTraderClaimBalanceAssets());
    console2.log("  engine mark           ", d.engine.lastMarkPrice());
    console2.log("  degradedMode          ", d.engine.degradedMode());
  }

  /// @dev Sum of USDG held by every in-scope custody contract (Clearinghouse, HousePool, both vaults, Engine).
  function _custodyUsdg() internal view returns (uint256) {
    return _usdgOf(address(d.clearinghouse)) + _usdgOf(address(d.housePool)) + _usdgOf(address(d.seniorVault))
      + _usdgOf(address(d.juniorVault)) + _usdgOf(address(d.engine));
  }

}
```

#### `ProtectionRetryLock24h.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {ReleaseFixture} from "./ReleaseFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/Test.sol";

interface IRouterExec {

  function executeOrder(
    uint64 orderId,
    bytes[] calldata pythUpdateData
  ) external payable returns (OrderV3Types.ExecutionResult memory result);

}

contract RetryLocker {

  IRouterExec internal immutable ROUTER;
  PositionProtectionBook internal immutable BOOK;

  constructor(
    address router,
    PositionProtectionBook book
  ) {
    ROUTER = IRouterExec(router);
    BOOK = book;
  }

  function cycle(
    uint64 child,
    uint64 protectionId,
    bytes[] calldata data
  ) external returns (uint64 nextChild, uint8 childStatus, uint8 childReason) {
    OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder(child, data);
    childStatus = uint8(r.status);
    childReason = uint8(r.terminalReason);
    if (BOOK.getPositionProtection(protectionId).status != PositionProtectionTypes.PositionProtectionStatus.Latched)
    {
      return (child, childStatus, uint8(r.pendingReason));
    }
    nextChild = BOOK.retryPositionProtectionClose(protectionId);
  }

}

contract ProtectionRetryLock24hTest is ReleaseFixture {

  address internal juniorLp = makeAddr("juniorLp");
  address internal alice = makeAddr("alice");
  address internal attackerEoa = makeAddr("attacker");
  RetryLocker internal locker;

  uint256 internal constant SIZE = 200_000e18; // 200,000 basket units, notional ~240,000 USDG at 1.20
  uint256 internal constant MARGIN = 20_000e6;
  uint256 internal constant SL = 122_000_000; // LONG stop-loss: mark >= 1.22
  uint256 internal constant GAP = 125_000_000; // trigger/fill price after a gap through the stop
  uint256 internal constant MAX_LOSS = 5_000e6; // stop-limit ceiling scales with the position

  function setUp() public override {
    super.setUp();
    _lpDeposit(false, juniorLp, 1_000_000e6, BASKET_1_20);
    _fundTrader(alice, 40_000e6);
    locker = new RetryLocker(address(d.router), _book());
  }

  function _book() internal view returns (PositionProtectionBook) {
    return PositionProtectionBook(d.positionProtectionBook);
  }

  function _limits(
    uint256 maxDebit
  ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
    l.maxExecutionNotionalAssets = type(uint256).max;
    l.maxGrossAccountDebitAssets = maxDebit;
    l.maxActionChargeAssets = type(uint256).max;
    l.maxExplicitFeesAssets = type(uint256).max;
  }

  function _status(
    uint64 pid
  ) internal view returns (uint256) {
    return uint256(uint8(_book().getPositionProtection(pid).status));
  }

  function _size(
    address a
  ) internal view returns (uint256 size) {
    (size,,,,,,) = d.engine.positions(a);
  }

  function _maximumWithdrawable(
    address account
  ) internal returns (uint256 maximum) {
    uint256 upper = d.clearinghouse.balanceAssets(account);
    while (maximum < upper) {
      uint256 mid = maximum + (upper - maximum + 1) / 2;
      uint256 snap = vm.snapshotState();
      vm.prank(account);
      (bool ok,) = address(d.clearinghouse).call(abi.encodeCall(d.clearinghouse.withdrawMargin, (mid)));
      vm.revertToState(snap);
      if (ok) maximum = mid;
      else upper = mid - 1;
    }
    uint256 exactSnap = vm.snapshotState();
    uint256 beforeWallet = _usdgOf(account);
    vm.prank(account);
    d.clearinghouse.withdrawMargin(maximum);
    assertEq(_usdgOf(account) - beforeWallet, maximum, "exact maximum transferred");
    vm.revertToState(exactSnap);
    if (maximum < d.clearinghouse.balanceAssets(account)) {
      vm.prank(account);
      (bool over,) = address(d.clearinghouse)
        .call(abi.encodeCall(d.clearinghouse.withdrawMargin, (maximum + 1)));
      assertFalse(over, "maximum plus one must fail");
    }
  }

  function _sel(
    bytes memory ret
  ) internal pure returns (bytes4 s) {
    if (ret.length >= 4) {
      assembly ("memory-safe") {
        s := mload(add(ret, 32))
      }
    }
  }

  function _standaloneOwnerActionsRevert(
    uint64 pid
  ) internal returns (bool) {
    OrderV3Types.OrderRequest memory closeReq = _closeReq(CfdTypes.Side.LONG, SIZE, 2e8);
    OrderV3Types.OrderRequest memory closeReq2 = _closeReq(CfdTypes.Side.LONG, SIZE, 2e8);
    bool ok;
    bytes memory ret;
    vm.prank(alice);
    (ok, ret) = address(_book())
      .call(abi.encodeCall(PositionProtectionBook.amendLatchedPositionProtectionCloseLimits, (pid, _limits(type(uint256).max))));
    bytes4 amendSel = _sel(ret);
    if (ok || (amendSel != IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector && amendSel != IOrderRouterErrors.OrderRouter__PendingOrdersExist.selector)) { console2.log("amend not blocked as expected; ok=", ok); console2.logBytes(ret); return false; }
    vm.prank(alice);
    (ok, ret) = address(d.router).call(abi.encodeCall(d.router.commitOrder, (closeReq)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionActive.selector) { console2.log("close commit result ok=", ok); console2.logBytes(ret); return false; }
    vm.prank(alice);
    (ok, ret) = address(_book()).call(abi.encodeCall(PositionProtectionBook.cancelPositionProtection, (pid)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector) { console2.log("cancel ok=", ok); console2.logBytes(ret); return false; }
    vm.prank(alice);
    (ok, ret) = address(_book())
      .call(abi.encodeCall(PositionProtectionBook.cancelPositionProtectionAndCommitClose, (pid, closeReq2)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector) { console2.log("cancel+close ok=", ok); console2.logBytes(ret); return false; }
    return true;
  }

  function _latchedAfterGap() internal returns (uint64 pid, uint64 child) {
    PositionProtectionTypes.PositionProtectionParams memory p;
    p.takeProfitTriggerPrice = 0;
    p.stopLossTriggerPrice = SL;
    p.closeLimits = _limits(MAX_LOSS);
    OrderV3Types.OrderRequest memory r = _openReq(CfdTypes.Side.LONG, SIZE, MARGIN, 1);
    vm.prank(alice);
    (uint64 parentId, uint64 protectionId) = _book().commitOpenOrderWithProtection(r, p);
    pid = protectionId;
    _requireExecuted(_execute(parentId, BASKET_1_20), "parent open");
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Armed), "Armed");
    _warp(1);
    _refreshMark(BASKET_1_20);

    _warp(30);
    _setBasketPrices(GAP);
    vm.prank(keeper);
    child = _book().triggerPositionProtection(pid, _updateData());
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
  }

  function test_control_owner_amends_and_exits() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    uint256 aliceBefore = d.clearinghouse.balanceAssets(alice);
    OrderV3Types.ExecutionResult memory r = _execute(child, GAP);
    console2.log("child status / terminalReason", uint256(uint8(r.status)), uint256(uint8(r.terminalReason)));
    assertEq(uint256(uint8(r.status)), uint256(OrderV3Types.LifecycleStatus.Failed), "child failed");
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Latched), "Latched");
    vm.prank(alice);
    _book().amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
    vm.prank(keeper);
    uint64 retry = _book().retryPositionProtectionClose(pid);
    _requireExecuted(_execute(retry, GAP), "retry close");
    assertEq(_size(alice), 0, "closed");
    uint256 aliceAfter = d.clearinghouse.balanceAssets(alice);
    console2.log("control: alice clearinghouse balance before/after exit", aliceBefore, aliceAfter);
    vm.prank(alice);
    d.clearinghouse.withdrawMargin(aliceAfter);
    console2.log("control: alice withdrew (USDG)", _usdgOf(alice));
    assertEq(_usdgOf(alice), aliceAfter, "owner exits and withdraws");
  }

  function test_attack_retry_loop_locks_owner_24h() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    uint256 aliceBefore = d.clearinghouse.balanceAssets(alice);
    uint256 start = vm.getBlockTimestamp();
    uint256 cycles = 24 * 60;
    uint256 gasTotal;
    uint256 gasMax;
    for (uint256 i; i < cycles; ++i) {
      vm.warp(vm.getBlockTimestamp() + 2);
      vm.roll(vm.getBlockNumber() + 1);
      _setBasketPrices(GAP);
      vm.warp(vm.getBlockTimestamp() + 56);
      vm.roll(vm.getBlockNumber() + 1);
      uint256 g0 = gasleft();
      vm.prank(attackerEoa);
      (uint64 next, uint8 st, uint8 reason) = locker.cycle(child, pid, _updateData());
      uint256 used = g0 - gasleft();
      gasTotal += used;
      if (used > gasMax) gasMax = used;
      if (i == 0) console2.log("cycle 0: child status / terminalReason", uint256(st), uint256(reason));
      if (st != uint8(OrderV3Types.LifecycleStatus.Failed)) {
        console2.log("cycle stopped at i / status / pendingReason", i, uint256(st), uint256(reason));
        console2.log("  t, child, engine mark time", vm.getBlockTimestamp(), uint256(child), d.engine.lastMarkTime());
        revert("child did not fail");
      }
      child = next;
      vm.roll(vm.getBlockNumber() + 1);
      vm.warp(vm.getBlockTimestamp() + 2);
      if (i % 120 == 0 || i == cycles - 1) {
        assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert");
      }
    }
    uint256 elapsed = vm.getBlockTimestamp() - start;
    console2.log("attack: attacker-first sequence elapsed seconds", elapsed);
    console2.log("attack: cycles / avg gas / max gas", cycles, gasTotal / cycles, gasMax);
    assertGe(elapsed, 24 hours, ">= 24 h");
    assertEq(_size(alice), SIZE, "position still open");
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered), "live child");
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert at the end");
    uint256 bal = d.clearinghouse.balanceAssets(alice);
    console2.log("attack: alice clearinghouse balance before/after 24 h", aliceBefore, bal);
    vm.prank(alice);
    (bool ok,) = address(d.clearinghouse).call(abi.encodeCall(d.clearinghouse.withdrawMargin, (bal)));
    assertFalse(ok, "full balance not withdrawable while position is open");
    console2.log("attack: alice pnl pledge (locked) ", d.clearinghouse.pnlPledgeAssets(alice));
    console2.log("attacker wallet / keeper USDG gained", _usdgOf(attackerEoa), d.clearinghouse.balanceAssets(address(locker)));
  }

  function test_control_vs_attack_same_snapshot() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    uint256 balStart = d.clearinghouse.balanceAssets(alice);
    uint256 custody0 = _custodyUsdg();
    uint256 snap = vm.snapshotState();

    _execute(child, GAP);
    vm.prank(alice);
    _book().amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
    vm.prank(keeper);
    uint64 r0 = _book().retryPositionProtectionClose(pid);
    _requireExecuted(_execute(r0, GAP), "control exit");
    uint256 baseEnd = d.clearinghouse.balanceAssets(alice);
    uint256 baseExitTime = vm.getBlockTimestamp();

    vm.revertToState(snap);
    uint256 t0 = vm.getBlockTimestamp();
    for (uint256 i; i < 24 * 60; ++i) {
      vm.warp(vm.getBlockTimestamp() + 2);
      vm.roll(vm.getBlockNumber() + 1);
      _setBasketPrices(GAP);
      vm.warp(vm.getBlockTimestamp() + 56);
      vm.roll(vm.getBlockNumber() + 1);
      vm.prank(attackerEoa);
      (child,,) = locker.cycle(child, pid, _updateData());
      vm.roll(vm.getBlockNumber() + 1);
      vm.warp(vm.getBlockTimestamp() + 2);
    }
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert at 24 h");
    uint256 lockSeconds = vm.getBlockTimestamp() - t0;
    IMarginClearinghouse.PnlIsolationBuckets memory buckets = d.clearinghouse.getPnlIsolationBuckets(alice);
    uint256 lockedMaxWithdraw = _maximumWithdrawable(alice);
    console2.log("same-snapshot: locked settlement / total locked / free", buckets.settlementBalanceAssets,
      buckets.totalLockedAssets, buckets.freeSettlementAssets);
    console2.log("same-snapshot: locked maximum withdraw / immediate control exit", lockedMaxWithdraw, baseEnd);
    assertLe(lockedMaxWithdraw, buckets.freeSettlementAssets, "withdraw cannot exceed free settlement");
    assertGt(baseEnd, lockedMaxWithdraw, "exit increases immediately accessible settlement");
    console2.log("same-snapshot: immediate control exit minus locked maximum withdraw", baseEnd - lockedMaxWithdraw);
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(GAP);
    vm.prank(keeper);
    d.router.executeOrder(child, _updateData());
    vm.prank(alice);
    _book().amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
    vm.prank(keeper);
    uint64 r1 = _book().retryPositionProtectionClose(pid);
    _requireExecuted(_execute(r1, GAP), "post-lock exit");
    uint256 attackEnd = d.clearinghouse.balanceAssets(alice);

    console2.log("same-snapshot: alice balance at latch", balStart);
    console2.log("same-snapshot: control exit balance / exit time", baseEnd, baseExitTime);
    console2.log("same-snapshot: attack exit balance / lock seconds", attackEnd, lockSeconds);
    console2.log("same-snapshot: extra cost to alice from the lock (carry etc.), atoms", baseEnd - attackEnd);
    console2.log("same-snapshot: attacker USDG gained", _usdgOf(attackerEoa) + d.clearinghouse.balanceAssets(address(locker)));
    assertGe(lockSeconds, 24 hours, "lock >= 24 h");
    assertGe(baseEnd, attackEnd, "lock never helps the owner at an unchanged price");
    assertEq(_custodyUsdg(), custody0, "custody conserved");
  }

  function test_attack_owner_separate_txs_lose_race() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(GAP);
    vm.prank(attackerEoa);
    (uint64 next,,) = locker.cycle(child, pid, _updateData());
    vm.roll(vm.getBlockNumber() + 1);
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert after one cycle");
    assertTrue(next != child, "fresh child appended");
    assertEq(d.router.pendingOrderCounts(alice), 1, "one pending child");
  }

  function test_counter_wallet_owner_escapes_by_winning_one_race() public {
    BatchingOwner w = new BatchingOwner(address(d.router), _book(), address(d.clearinghouse), IERC20Like(address(d.settlementToken)));
    _mint(address(w), 40_000e6);
    w.deposit(40_000e6);
    PositionProtectionTypes.PositionProtectionParams memory p;
    p.stopLossTriggerPrice = SL;
    p.closeLimits = _limits(MAX_LOSS);
    OrderV3Types.OrderRequest memory r = _openReq(CfdTypes.Side.LONG, SIZE, MARGIN, 1);
    (uint64 parentId, uint64 pid) = w.openProtected(r, p);
    _requireExecuted(_execute(parentId, BASKET_1_20), "wallet parent open");
    _warp(1);
    _refreshMark(BASKET_1_20);
    _warp(30);
    _setBasketPrices(GAP);
    vm.prank(keeper);
    uint64 child = _book().triggerPositionProtection(pid, _updateData());
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(GAP);
    vm.prank(attackerEoa);
    (uint64 next,,) = locker.cycle(child, pid, _updateData());
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(GAP);
    uint64 retry = w.escape(next, pid, _updateData(), _limits(type(uint256).max));
    _requireExecuted(_execute(retry, GAP), "wallet retry close");
    assertEq(_size(address(w)), 0, "wallet owner escaped");
    console2.log("counter: wallet owner balance after escape", d.clearinghouse.balanceAssets(address(w)));
  }

  function test_execute_and_retry_cycle_gas() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    uint256 cycles = 8;
    uint256 gasMin = type(uint256).max;
    uint256 gasMax;
    for (uint256 i; i < cycles; ++i) {
      vm.warp(vm.getBlockTimestamp() + 2);
      vm.roll(vm.getBlockNumber() + 1);
      _setBasketPrices(GAP);
      vm.warp(vm.getBlockTimestamp() + 56);
      vm.roll(vm.getBlockNumber() + 1);
      vm.prank(attackerEoa);
      (uint64 next, uint8 st,) = locker.cycle(child, pid, _updateData());
      uint256 used = vm.lastCallGas().gasTotalUsed;
      assertEq(uint256(st), uint256(OrderV3Types.LifecycleStatus.Failed), "child failed");
      assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered), "live child");
      if (used < gasMin) gasMin = used;
      if (used > gasMax) gasMax = used;
      child = next;
      vm.roll(vm.getBlockNumber() + 1);
      vm.warp(vm.getBlockTimestamp() + 2);
    }
    console2.log("execute-and-retry cycle gas: cycles / min / max", cycles, gasMin, gasMax);
    assertGt(gasMin, 0, "nonzero cycle gas");
    assertGe(gasMin, 1_000_000, "cycle gas below sanity range");
    assertLe(gasMax, 2_000_000, "cycle gas above sanity range");
  }

}

contract BatchingOwner {

  IRouterExec internal immutable ROUTER;
  PositionProtectionBook internal immutable BOOK;
  address internal immutable CH;
  IERC20Like internal immutable TOKEN;

  constructor(
    address router,
    PositionProtectionBook book,
    address clearinghouse,
    IERC20Like token
  ) {
    ROUTER = IRouterExec(router);
    BOOK = book;
    CH = clearinghouse;
    TOKEN = token;
  }

  function deposit(
    uint256 amount
  ) external {
    TOKEN.approve(CH, amount);
    IClearinghouseLike(CH).depositMargin(amount);
  }

  function openProtected(
    OrderV3Types.OrderRequest calldata r,
    PositionProtectionTypes.PositionProtectionParams calldata p
  ) external returns (uint64, uint64) {
    return BOOK.commitOpenOrderWithProtection(r, p);
  }

  function escape(
    uint64 child,
    uint64 pid,
    bytes[] calldata data,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
  ) external returns (uint64 retry) {
    ROUTER.executeOrder(child, data);
    BOOK.amendLatchedPositionProtectionCloseLimits(pid, limits);
    retry = BOOK.retryPositionProtectionClose(pid);
  }

}

interface IERC20Like {

  function approve(
    address spender,
    uint256 amount
  ) external returns (bool);

}

interface IClearinghouseLike {

  function depositMargin(
    uint256 amount
  ) external;

}
```

#### `ProtectionRetryLockLiquidation.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {ReleaseFixture} from "./ReleaseFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/Test.sol";

interface IRouterExecuteOrder {

  function executeOrder(
    uint64 orderId,
    bytes[] calldata pythUpdateData
  ) external payable returns (OrderV3Types.ExecutionResult memory result);

}

contract ExecuteAndRetryCaller {

  IRouterExecuteOrder internal immutable ROUTER;
  PositionProtectionBook internal immutable BOOK;

  constructor(
    address router,
    PositionProtectionBook book
  ) {
    ROUTER = IRouterExecuteOrder(router);
    BOOK = book;
  }

  function cycle(
    uint64 child,
    uint64 protectionId,
    bytes[] calldata data
  ) external returns (uint64 nextChild, uint8 childStatus, uint8 childReason) {
    OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder(child, data);
    childStatus = uint8(r.status);
    childReason = uint8(r.terminalReason);
    if (BOOK.getPositionProtection(protectionId).status != PositionProtectionTypes.PositionProtectionStatus.Latched)
    {
      return (child, childStatus, uint8(r.pendingReason));
    }
    nextChild = BOOK.retryPositionProtectionClose(protectionId);
  }

}

contract ProtectionRetryLockLiquidationTest is ReleaseFixture {

  address internal juniorLp = makeAddr("juniorLp");
  address internal alice = makeAddr("alice");
  address internal aliceHedge = makeAddr("aliceHedge");
  address internal retryCaller = makeAddr("retryCaller");
  ExecuteAndRetryCaller internal locker;

  uint256 internal constant SIZE = 200_000e18;
  uint256 internal constant LOTS = 2000; // SIZE / positionSizeQuantum (1e20)
  uint256 internal constant MARGIN = 20_000e6;
  uint256 internal constant SL = 122_000_000;
  uint256 internal constant GAP = 125_000_000;
  uint256 internal constant MAX_LOSS = 5_000e6;
  uint256 internal constant CYCLES = 1440; // one cycle per 60 s for 24 h
  uint256 internal constant MILD_END = 126_250_000; // 1.2625 = 1.25 * 1.01

  struct Outcome {
    uint256 settlement; // clearinghouse settlement balance
    uint256 traderClaim; // engine trader claim
    uint256 wallet; // USDG withdrawn to the wallet
    uint256 poolAssets; // HousePool totalAssets
    uint256 time;
  }

  function setUp() public override {
    super.setUp();
    _lpDeposit(false, juniorLp, 1_000_000e6, BASKET_1_20);
    _fundTrader(alice, 40_000e6);
    locker = new ExecuteAndRetryCaller(address(d.router), _book());
  }


  function _book() internal view returns (PositionProtectionBook) {
    return PositionProtectionBook(d.positionProtectionBook);
  }

  function _limits(
    uint256 maxDebit
  ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
    l.maxExecutionNotionalAssets = type(uint256).max;
    l.maxGrossAccountDebitAssets = maxDebit;
    l.maxActionChargeAssets = type(uint256).max;
    l.maxExplicitFeesAssets = type(uint256).max;
  }

  function _status(
    uint64 pid
  ) internal view returns (uint256) {
    return uint256(uint8(_book().getPositionProtection(pid).status));
  }

  function _size(
    address a
  ) internal view returns (uint256 size) {
    (size,,,,,,) = d.engine.positions(a);
  }

  function _sel(
    bytes memory ret
  ) internal pure returns (bytes4 s) {
    if (ret.length >= 4) {
      assembly ("memory-safe") {
        s := mload(add(ret, 32))
      }
    }
  }

  function _standaloneOwnerActionsRevert(
    uint64 pid
  ) internal returns (bool) {
    OrderV3Types.OrderRequest memory closeReq = _closeReq(CfdTypes.Side.LONG, SIZE, 2e8);
    OrderV3Types.OrderRequest memory closeReq2 = _closeReq(CfdTypes.Side.LONG, SIZE, 2e8);
    bool ok;
    bytes memory ret;
    vm.prank(alice);
    (ok, ret) = address(_book())
      .call(abi.encodeCall(PositionProtectionBook.amendLatchedPositionProtectionCloseLimits, (pid, _limits(type(uint256).max))));
    bytes4 amendSel = _sel(ret);
    if (ok || (amendSel != IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector && amendSel != IOrderRouterErrors.OrderRouter__PendingOrdersExist.selector)) { return false; }
    vm.prank(alice);
    (ok, ret) = address(d.router).call(abi.encodeCall(d.router.commitOrder, (closeReq)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionActive.selector) { return false; }
    vm.prank(alice);
    (ok, ret) = address(_book()).call(abi.encodeCall(PositionProtectionBook.cancelPositionProtection, (pid)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector) { return false; }
    vm.prank(alice);
    (ok, ret) = address(_book())
      .call(abi.encodeCall(PositionProtectionBook.cancelPositionProtectionAndCommitClose, (pid, closeReq2)));
    if (ok || _sel(ret) != IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector) { return false; }
    return true;
  }

  function _latchedAfterGap() internal returns (uint64 pid, uint64 child) {
    PositionProtectionTypes.PositionProtectionParams memory p;
    p.stopLossTriggerPrice = SL;
    p.closeLimits = _limits(MAX_LOSS);
    OrderV3Types.OrderRequest memory r = _openReq(CfdTypes.Side.LONG, SIZE, MARGIN, 1);
    vm.prank(alice);
    (uint64 parentId, uint64 protectionId) = _book().commitOpenOrderWithProtection(r, p);
    pid = protectionId;
    _requireExecuted(_execute(parentId, BASKET_1_20), "parent open");
    _warp(1);
    _refreshMark(BASKET_1_20);
    _warp(30);
    _setBasketPrices(GAP);
    vm.prank(keeper);
    child = _book().triggerPositionProtection(pid, _updateData());
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
  }


  function _liquidationPriceClosedForm(
    address a
  ) internal view returns (uint256 price, uint256 collateralPlusEntry) {
    (uint256 size, uint256 margin,,,,,) = d.engine.positions(a);
    assertEq(size / 1e20, LOTS, "lots");
    (,, uint256 maintMarginBps,,,,,,,) = d.engine.riskParams();
    assertEq(maintMarginBps, 10, "release maintenance margin 0.1%");
    collateralPlusEntry = margin + d.engine.traderClaimBalanceAssets(a) + d.engine.positionEntryCostAssetAtoms(a);
    uint256 den = LOTS * (10_000 + maintMarginBps) / 10_000;
    price = (collateralPlusEntry + den - 1) / den;
  }

  function _liquidationPriceLens(
    address a
  ) internal view returns (uint256 lo) {
    uint256 depth = d.housePool.totalAssets();
    lo = GAP;
    uint256 hi = 2e8;
    require(!d.engineLens.isLiquidatableAt(a, lo, depth), "already liquidatable at 1.25");
    require(d.engineLens.isLiquidatableAt(a, hi, depth), "not liquidatable at cap");
    while (hi - lo > 1) {
      uint256 mid = (lo + hi) / 2;
      if (d.engineLens.isLiquidatableAt(a, mid, depth)) hi = mid;
      else lo = mid;
    }
    lo = hi;
  }

  function _pathPrice(
    uint256 endPrice,
    uint256 i
  ) internal pure returns (uint256) {
    return GAP + (endPrice - GAP) * (i + 1) / CYCLES;
  }

  function _cycleAt(
    uint64 child,
    uint64 pid,
    uint256 price
  ) internal returns (uint64 next, uint256 gasUsed) {
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(price);
    vm.warp(vm.getBlockTimestamp() + 56);
    vm.roll(vm.getBlockNumber() + 1);
    uint256 g0 = gasleft();
    vm.prank(retryCaller);
    (uint64 n, uint8 st, uint8 reason) = locker.cycle(child, pid, _updateData());
    gasUsed = g0 - gasleft();
    if (st != uint8(OrderV3Types.LifecycleStatus.Failed)) {
      console2.log("protection close not failed: status / reason / price", uint256(st), uint256(reason), price);
      revert("protection close did not fail");
    }
    next = n;
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(vm.getBlockTimestamp() + 2);
  }

  function _lockedDrift(
    uint64 child,
    uint64 pid,
    uint256 endPrice
  ) internal returns (uint64 lastChild, uint256 gasTotal) {
    for (uint256 i; i < CYCLES; ++i) {
      uint256 used;
      (child, used) = _cycleAt(child, pid, _pathPrice(endPrice, i));
      gasTotal += used;
      if (i % 120 == 0 || i == CYCLES - 1) {
        assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert");
      }
    }
    lastChild = child;
  }

  function _controlExit(
    uint64 pid,
    uint64 child,
    uint256 finalTime,
    uint256 finalPrice
  ) internal returns (Outcome memory o, uint256 exitTime, uint256 exitBalance) {
    OrderV3Types.ExecutionResult memory r = _execute(child, GAP);
    assertEq(uint256(uint8(r.status)), uint256(OrderV3Types.LifecycleStatus.Failed), "control: first close fails");
    vm.prank(alice);
    _book().amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
    vm.prank(keeper);
    uint64 r0 = _book().retryPositionProtectionClose(pid);
    _requireExecuted(_execute(r0, GAP), "control exit");
    assertEq(_size(alice), 0, "control: flat");
    exitTime = vm.getBlockTimestamp();
    exitBalance = d.clearinghouse.balanceAssets(alice);
    _warpTo(finalTime);
    _refreshMark(finalPrice);
    o = _finalOutcome(alice);
  }

  function _finalOutcome(
    address a
  ) internal returns (Outcome memory o) {
    o.settlement = d.clearinghouse.balanceAssets(a);
    o.traderClaim = d.engine.traderClaimBalanceAssets(a);
    o.poolAssets = d.housePool.totalAssets();
    o.time = vm.getBlockTimestamp();
    uint256 w0 = _usdgOf(a);
    if (o.settlement != 0) {
      vm.prank(a);
      d.clearinghouse.withdrawMargin(o.settlement);
    }
    o.wallet = _usdgOf(a) - w0;
    assertEq(o.wallet, o.settlement, "full settlement withdrawable");
  }

  function _ownerExitAfterLock(
    uint64 pid,
    uint64 child,
    uint256 price
  ) internal {
    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(price);
    vm.prank(keeper);
    d.router.executeOrder(child, _updateData());
    vm.prank(alice);
    _book().amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
    vm.prank(keeper);
    uint64 r1 = _book().retryPositionProtectionClose(pid);
    _requireExecuted(_execute(r1, price), "exit after lock");
    assertEq(_size(alice), 0, "flat after lock");
  }



  function test_drift_to_liquidation_while_protection_close_pending() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    (uint256 pLiqClosed,) = _liquidationPriceClosedForm(alice);
    uint256 pLiq = _liquidationPriceLens(alice);
    (, uint256 posMargin,,,,,) = d.engine.positions(alice);
    console2.log("position margin (PnL pledge) / trader claim / entry cost atoms", posMargin,
      d.engine.traderClaimBalanceAssets(alice), d.engine.positionEntryCostAssetAtoms(alice));
    console2.log("liquidation price: closed form without pending carry / canonical lens predicate", pLiqClosed, pLiq);
    assertGe(pLiqClosed, pLiq, "pending carry can only lower the threshold");
    assertLe(pLiqClosed - pLiq, 1, "closed form within one price unit");
    uint256 t0 = vm.getBlockTimestamp();
    uint256 finalTime = t0 + CYCLES * 60 + 2;
    uint256 custody0 = _custodyUsdg();
    uint256 snap = vm.snapshotState();

    uint256 callerWallet0 = _usdgOf(retryCaller);
    uint256 callerSettlement0 = d.clearinghouse.balanceAssets(retryCaller);
    (uint64 lastChild, uint256 gasTotal) = _lockedDrift(child, pid, pLiq);
    assertEq(_size(alice), SIZE, "position open after 24 h");
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
    assertEq(d.router.pendingOrderCounts(alice), 1, "one pending protection close");
    assertEq(uint256(d.lifecycleBook.lifecycleStatus(lastChild)), uint256(OrderV3Types.LifecycleStatus.Pending), "live close");
    uint256 pLiqEnd = _liquidationPriceLens(alice);
    console2.log("liquidation price after 24 h (canonical lens predicate)", pLiqEnd);
    assertLe(pLiqEnd, pLiq + 1, "threshold stable within one unit");
    IMarginClearinghouse.PnlIsolationBuckets memory b = d.clearinghouse.getPnlIsolationBuckets(alice);
    console2.log("Alice before liquidation: settlement / PnL pledge / liquidation reserve", b.settlementBalanceAssets,
      b.pnlPledgeAssets, b.liquidationReserveAssets);
    console2.log("Alice before liquidation: free settlement", b.freeSettlementAssets);

    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    assertEq(vm.getBlockTimestamp(), finalTime, "final timestamp");
    uint256 liqPrice = pLiqEnd > pLiq ? pLiqEnd : pLiq;
    _setBasketPrices(liqPrice);
    ICfdEngineTypes.LiquidationPreview memory pv =
      d.engineLens.simulateLiquidation(alice, liqPrice, d.housePool.totalAssets());
    assertTrue(pv.liquidatable, "liquidatable");
    assertGt(pv.equityAssets, 0, "equity positive at liquidation: no bad debt");
    assertFalse(pv.triggersDegradedMode, "no degraded mode");
    console2.log("liquidation preview: equity / liquidation charge / keeper bounty", uint256(pv.equityAssets),
      pv.liquidationChargeAssets, pv.keeperBountyAssets);
    console2.log("liquidation preview: LP liquidation fee / settlement retained / immediate payout",
      pv.lpLiquidationFeeAssets, pv.settlementRetainedAssets, pv.immediatePayoutAssets);
    uint256 g0 = gasleft();
    vm.prank(retryCaller);
    d.router.executeLiquidation(alice, _updateData());
    uint256 liqGas = g0 - gasleft();
    assertEq(_size(alice), 0, "liquidated");
    assertEq(_status(pid), uint256(PositionProtectionTypes.PositionProtectionStatus.Liquidated), "protection Liquidated");
    assertEq(d.router.pendingOrderCounts(alice), 0, "queue cleared");
    OrderV3Types.TerminalOutcome memory to = d.lifecycleBook.terminalOutcome(lastChild);
    assertEq(uint256(to.status), uint256(OrderV3Types.LifecycleStatus.Failed), "pending close failed");
    assertEq(uint256(to.reason), uint256(OrderV3Types.TerminalReason.AccountLiquidated), "AccountLiquidated");
    assertFalse(d.engine.degradedMode(), "engine not degraded");
    assertEq(_custodyUsdg(), custody0, "custody conserved");

    uint256 bounty = d.clearinghouse.balanceAssets(retryCaller) - callerSettlement0;
    assertEq(bounty, pv.keeperBountyAssets, "bounty equals preview");
    vm.prank(retryCaller);
    d.clearinghouse.withdrawMargin(bounty);
    uint256 callerWallet1 = _usdgOf(retryCaller);
    assertEq(callerWallet1 - callerWallet0, bounty, "retry caller withdrew the keeper bounty");

    Outcome memory a = _finalOutcome(alice);
    assertEq(a.time, finalTime, "locked: final timestamp");

    vm.revertToState(snap);
    (Outcome memory c, uint256 cExitTime, uint256 cExitBal) = _controlExit(pid, child, finalTime, liqPrice);
    assertEq(c.settlement, cExitBal, "control balance unchanged after exit");
    assertEq(c.time, a.time, "same final timestamp in both sequences");

    console2.log("immediate-exit control: exit time / balance", cExitTime, cExitBal);
    console2.log("locked sequence: cycles / total gas / liquidation gas", CYCLES, gasTotal, liqGas);
    console2.log("locked sequence: liquidation time / price", a.time, liqPrice);
    console2.log("Alice final USDG: immediate-exit control / locked then liquidated", c.wallet, a.wallet);
    console2.log("Alice realized loss from the lock", c.wallet - a.wallet);
    console2.log("HousePool totalAssets: control / locked", c.poolAssets, a.poolAssets);
    console2.log("retry caller USDG before / after (keeper bounty)", callerWallet0, callerWallet1);
    assertEq(a.traderClaim, 0, "no trader claim");
    assertEq(c.traderClaim, 0, "no trader claim control");
    assertGt(c.wallet, a.wallet, "lock increases realized loss");
    assertEq(c.wallet - a.wallet, (a.poolAssets - c.poolAssets) + bounty, "loss = LP gain + keeper bounty");
    assertEq(pLiqClosed, 129_558_442, "closed-form threshold");
    assertEq(pLiq, 129_558_441, "lens threshold at the latch");
    assertEq(pLiqEnd, 129_555_112, "lens threshold after 24 h");
    assertEq(uint256(pv.equityAssets), 252_451_978, "equity at liquidation");
    assertEq(pv.liquidationChargeAssets, 240_000_000, "liquidation charge = liquidation reserve");
    assertEq(bounty, 120_000_000, "keeper bounty 120 USDG");
    assertEq(c.wallet, 29_803_596_530, "control final");
    assertEq(a.wallet, 20_252_095_978, "liquidated final");
    assertEq(c.wallet - a.wallet, 9_551_500_552, "realized loss");
    assertEq(a.poolAssets - c.poolAssets, 9_431_500_552, "LP gain");
  }


  function test_mild_drift_realized_loss() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    uint256 t0 = vm.getBlockTimestamp();
    uint256 snap = vm.snapshotState();

    (uint64 lastChild, uint256 gasTotal) = _lockedDrift(child, pid, MILD_END);
    _ownerExitAfterLock(pid, lastChild, MILD_END);
    Outcome memory a = _finalOutcome(alice);
    uint256 finalTime = a.time;

    vm.revertToState(snap);
    (Outcome memory c,,) = _controlExit(pid, child, finalTime, MILD_END);

    console2.log("mild drift: start / end basket price", GAP, MILD_END);
    console2.log("mild drift: cycles / total gas / lock seconds", CYCLES, gasTotal, finalTime - t0);
    console2.log("Alice final USDG: immediate-exit control / locked then exit", c.wallet, a.wallet);
    console2.log("Alice realized loss from the lock", c.wallet - a.wallet);
    console2.log("HousePool totalAssets: control / locked", c.poolAssets, a.poolAssets);
    assertEq(c.time, a.time, "same final timestamp");
    assertGt(c.wallet, a.wallet, "lock increases realized loss");
    assertGe(c.wallet - a.wallet, 2_500e6, "loss at least the price move");
    assertEq(c.wallet - a.wallet, a.poolAssets - c.poolAssets, "loss = LP gain");
    assertEq(c.wallet - a.wallet, 2_507_663_668, "realized loss");
    assertEq(finalTime - t0, 86_409, "lock seconds");
  }


  function test_owner_actions_during_lock() public {
    (uint64 pid, uint64 child) = _latchedAfterGap();
    (child,) = _cycleAt(child, pid, GAP);
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert after one cycle");

    bool ok;
    bytes memory ret;
    PositionProtectionTypes.PositionProtectionParams memory p;
    p.stopLossTriggerPrice = 130_000_000;
    p.closeLimits = _limits(type(uint256).max);
    vm.prank(alice);
    (ok, ret) = address(_book()).call(abi.encodeCall(PositionProtectionBook.replacePositionProtection, (pid, p)));
    assertFalse(ok, "replace reverts");
    assertEq(_sel(ret), IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, "replace: ProtectionNotArmed");
    vm.prank(alice);
    (ok, ret) = address(_book()).call(abi.encodeCall(PositionProtectionBook.createPositionProtection, (p)));
    assertFalse(ok, "create reverts");
    assertEq(_sel(ret), IOrderRouterErrors.OrderRouter__ProtectionAlreadyActive.selector, "create: ProtectionAlreadyActive");
    console2.log("owner: createPositionProtection revert selector");
    console2.logBytes4(_sel(ret));
    OrderV3Types.OrderRequest memory inc = _openReq(CfdTypes.Side.LONG, 100e18 * 10, 1_000e6, 1);
    vm.prank(alice);
    (ok, ret) = address(d.router).call(abi.encodeCall(d.router.commitOrder, (inc)));
    assertFalse(ok, "ordinary open reverts");
    assertEq(_sel(ret), IOrderRouterErrors.OrderRouter__ProtectionActive.selector, "open: ProtectionActive");
    uint256 bal = d.clearinghouse.balanceAssets(alice);
    vm.prank(alice);
    (ok,) = address(d.clearinghouse).call(abi.encodeCall(d.clearinghouse.withdrawMargin, (bal)));
    assertFalse(ok, "full withdrawal reverts");

    (uint256 pLiqBefore,) = _liquidationPriceClosedForm(alice);
    IMarginClearinghouse.PnlIsolationBuckets memory b = d.clearinghouse.getPnlIsolationBuckets(alice);
    uint256 add = b.freeSettlementAssets - 100e6;
    vm.prank(alice);
    d.engine.addMargin(alice, add);
    (uint256 pLiqAfter,) = _liquidationPriceClosedForm(alice);
    assertEq(_liquidationPriceLens(alice), pLiqAfter, "lens agrees after add margin");
    console2.log("owner: addMargin amount / liquidation price before / after", add, pLiqBefore, pLiqAfter);
    assertGt(pLiqAfter, pLiqBefore, "add margin raises the liquidation price");
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert after add margin");

    (child,) = _cycleAt(child, pid, GAP);
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert after the next cycle");

    vm.warp(vm.getBlockTimestamp() + 2);
    vm.roll(vm.getBlockNumber() + 1);
    _setBasketPrices(pLiqBefore);
    vm.prank(retryCaller);
    (ok, ret) = address(d.router).call(abi.encodeCall(d.router.executeLiquidation, (alice, _updateData())));
    assertFalse(ok, "liquidation at the former price reverts");
    assertEq(_sel(ret), ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector, "PositionIsSolvent");
    console2.log("liquidation at the former liquidation price: revert selector");
    console2.logBytes4(_sel(ret));
    assertEq(_size(alice), SIZE, "position remains");
    assertEq(add, 19_899_600_000, "added margin");
    assertEq(pLiqBefore, 129_558_442, "threshold before add margin (closed form)");
    assertEq(pLiqAfter, 139_498_299, "threshold after add margin (closed form)");
  }


  function test_owner_hedge_bounds_loss() public {
    (uint64 pid, uint64 child0) = _latchedAfterGap();
    (uint256 pLiq,) = _liquidationPriceClosedForm(alice);
    uint256 snap = vm.snapshotState();
    uint64 child = child0;

    uint256 hedgeDeposit = 5_000e6;
    _fundTrader(aliceHedge, hedgeDeposit);
    (uint64 c1,) = _cycleAt(child, pid, _pathPrice(pLiq, 0));
    IMarginClearinghouse.PnlIsolationBuckets memory b = d.clearinghouse.getPnlIsolationBuckets(alice);
    vm.prank(alice);
    d.engine.addMargin(alice, b.freeSettlementAssets - 100e6);
    uint64 hedgeId = _commit(aliceHedge, _openReq(CfdTypes.Side.SHORT, SIZE, 2_000e6, 4e8));
    (uint64 c2,) = _cycleAt(c1, pid, _pathPrice(pLiq, 1));
    vm.prank(keeper);
    OrderV3Types.ExecutionResult memory hr = d.router.executeOrder(hedgeId, _updateData());
    _requireExecuted(hr, "hedge open");
    child = c2;
    for (uint256 i = 2; i < CYCLES; ++i) {
      (child,) = _cycleAt(child, pid, _pathPrice(pLiq, i));
    }
    assertEq(_size(alice), SIZE, "LONG still open");
    assertTrue(_standaloneOwnerActionsRevert(pid), "standalone owner actions revert at 24 h");
    _ownerExitAfterLock(pid, child, pLiq);
    _requireExecuted(_closePosition(aliceHedge, CfdTypes.Side.SHORT, SIZE, pLiq), "hedge close");
    Outcome memory a = _finalOutcome(alice);
    Outcome memory h = _finalOutcome(aliceHedge);
    uint256 finalTime = vm.getBlockTimestamp();

    vm.revertToState(snap);
    (Outcome memory c,,) = _controlExit(pid, child0, finalTime, pLiq);

    uint256 combined = a.wallet + h.wallet - hedgeDeposit;
    console2.log("hedge: Alice LONG account / hedge account final USDG", a.wallet, h.wallet);
    console2.log("hedge: combined net of hedge deposit / immediate-exit control", combined, c.wallet);
    console2.log("hedge: residual loss against control", c.wallet > combined ? c.wallet - combined : 0);
    assertEq(c.time, finalTime, "same final timestamp");
    assertEq(a.wallet, 20_053_878_173, "hedged LONG account final");
    assertEq(h.wallet, 13_897_344_415, "hedge account final");
    assertEq(c.wallet - combined, 852_373_942, "residual loss with hedge and added margin");
  }

}
```

