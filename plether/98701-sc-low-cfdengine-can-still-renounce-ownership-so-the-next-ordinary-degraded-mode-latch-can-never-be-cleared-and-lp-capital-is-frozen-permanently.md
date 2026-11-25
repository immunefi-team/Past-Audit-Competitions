# \#98701 [SC-Low] `CfdEngine` can still renounce ownership, so the next ordinary degraded-mode latch can never be cleared and LP capital is frozen permanently

**Submitted on Nov 9th 2026 at 17:00:29 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98701
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Affected contracts (in-scope commit `a82d4fef`; live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. I read the live wiring with read-only calls:
- `HousePool.ENGINE()` (immutable) and `OrderRouter.engine()` both return the `CfdEngine` below.
- The engine's `clearinghouse()` and the pool's vaults are the addresses listed.
- The engine currently has an owner, no pending owner, and `degradedMode() == false`.

The proof of concept runs in the repository's Foundry suite at the in-scope commit. It uses the release parameters, and Pyth and USDG are mocked. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | **Root cause.** Inherits `Ownable.renounceOwnership()` through `CfdTreasury is Ownable2Step` with no override. It is the only contract that can leave degraded mode (`clearDegradedMode()`, `recapitalizePool()`, both `onlyOwner`), and it latches `degradedMode` after any qualifying close or liquidation |
| `HousePool` | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | Holds LP capital. `settleLpEpoch` reverts `HousePool__DegradedMode` while the engine is latched. Its `ENGINE` is immutable. Its own `renounceOwnership()` reverts (the earlier fix) |
| `TrancheVault` (Senior) | `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d` | Senior redemptions are funded only by HousePool epoch settlement. A matured request cannot be cancelled |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | Junior redemptions and queued deposit escrow. A matured deposit cannot activate or be cancelled |
| `MarginClearinghouse` | `0xb5b851f627d6df3cf3f096e23e20a58884d7b89a` | Withdrawals by accounts with an open position go through the engine guard, which reverts `CfdEngine__DegradedMode`. Also inherits `renounceOwnership()` |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder` / `executeOrder` / `updateMarkPrice` / `settleLpEpoch` used by traders and any keeper |
| `CfdEngineAdmin` | `0xfa444c18c90f1b5f768eddbb56cd1d92710a8327` | Engine risk-config module. Also inherits `renounceOwnership()`. It has no path that clears degraded mode |

Release parameters used (`packages/perps/DEPLOYMENT.md` and the program page):
- risk: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, frozen-market 300 bps, `baseCarryBps` 500;
- execution fee 4 bps, LP epoch 1 hour, LP deposit cooldown 1 hour;
- tranche seeds of 1 USDG each, and the Senior cap of 40,000,000 USDG and 80% of the pool, set through the 72-hour timelock;
- Router: execution window 60 s, order staleness 60 s, liquidation staleness 15 s, basket confidence 10 bps, settlement window 15 s, `minOpenNotionalAssets` 1,000 USDG, `maxPendingOrders` 5, applied through the owner's 72-hour timelocked config path.

## Brief/Intro

**When the engine owner renounces, the next time degraded mode latches, nobody can ever clear it. Degraded mode blocks LP epoch settlement and position holders' withdrawals, so LP funds are frozen forever.**

`CfdEngine` still inherits OpenZeppelin's `renounceOwnership()`. An earlier audit finding on exactly this root cause led to a revert override in `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`, but not in `CfdEngine`. Yet `CfdEngine` is the only contract with a way out of degraded mode: `clearDegradedMode()` and `recapitalizePool()` are both `onlyOwner`, and nothing else ever resets the flag.

The latch itself is documented design and needs no attacker. Any trader's profitable close that leaves pool assets net of claims (`E`) below the maximum directional liability (`L`) sets it. The documented recovery is an owner who clears it once `E >= L`. With an ownerless engine, that recovery no longer exists. `HousePool.settleLpEpoch` reverts `HousePool__DegradedMode` forever, even after the balance sheet is fully restored. Every LP redemption is funded, and every queued deposit activated, only through that settlement. Matured redemptions and matured Junior deposits also cannot be cancelled. New LP deposits and new positions are refused.

The proof of concept uses release parameters and public calls only. The one privileged call is the engine owner's `renounceOwnership()`. An ordinary trader's +10% SHORT close then latches degraded mode. Solvency is then fully restored: `E` = 1,207,037.730007 USDG against `L` = 2,000.000000 USDG. After that, the keeper's LP settlement reverts on every attempt, and nobody can clear the latch. That includes the former owner, a keeper, an LP and an outsider. The LP capital held by `HousePool` (1,207,037.730007 USDG) and a queued Junior deposit (100,000.000000 USDG) stay frozen. Matured Junior and Senior redemption requests stay unfunded and uncancellable. 52 weeks later the state is identical. In the control branch, the owner clears the latch and the same requests are paid in one keeper settlement.

## Vulnerability Details

### `CfdEngine` still inherits `renounceOwnership()`

`CfdEngine` is `Ownable2Step` through `CfdTreasury`:

`CfdEngine.sol`, line 46, and `CfdTreasury.sol`, line 10:

```solidity
contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdTreasury, ReentrancyGuardTransient {
abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {
```

Neither contract overrides `renounceOwnership()`, so OpenZeppelin's `Ownable.renounceOwnership()` sets `owner()` to `address(0)` in one transaction. `CfdEngineAdmin` (`contract CfdEngineAdmin is Ownable2Step`, line 17) and `MarginClearinghouse` (`... Ownable2Step, ReentrancyGuardTransient`, line 30) are in the same position. Three other perps contracts were changed so that renouncing reverts:

`HousePool.sol` line 265, `OrderRouterAdmin.sol` line 125, `EmergencyPauseCoordinator.sol` line 102:

```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

### The only way out of degraded mode is owner-only

`degradedMode` is written in exactly two places. The latch sets it, and only the owner clears it:

`CfdEngine.sol`, lines 1351–1369 (latch, reached after every close and liquidation, lines 1320 and 1344):

```solidity
function _enterDegradedModeIfInsolvent(
    address account,
    uint256 pendingPoolPayoutAssets
) internal {
    if (degradedMode) {
        return;
    }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ...
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

`CfdEngine.sol`, lines 658–667:

```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

`recapitalizePool` (lines 632–653) is also `onlyOwner`, and by its own NatSpec it "does not clear degraded mode". No other contract writes the flag. `HousePool` binds the engine immutably (`ICfdEngineCore public immutable ENGINE;`, `HousePool.sol` line 133), so the pool cannot be pointed at a replacement engine. Once `owner() == address(0)`, the condition `_availableCashForFreshPoolPayouts() >= _maxLiability()` can hold forever while the flag stays `true`.

### Degraded mode blocks every LP exit and entry

All LP exits go through `HousePool.settleLpEpoch`, and its body rejects a latched engine before it does anything else:

`HousePool.sol`, lines 824–833 and 1864–1870:

```solidity
function _settleLpEpoch(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot,
    uint256 cutoffEpoch
) internal returns (IHousePool.LpEpochSettlementResult memory result) {
    if (seniorVault == address(0) || juniorVault == address(0)) {
        revert HousePool__ZeroAddress();
    }
    _requireWithdrawalsLive(statusSnapshot);
    ...
function _requireWithdrawalsLive(
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode();
    }
}
```

The consequences in `TrancheVault` follow from that:

- **Redemptions.** A redemption is paid only from an epoch that settlement has funded. Once the request's epoch has started, `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable` (`TrancheVault.sol` lines 893–897). The shares stay in escrow, unfunded.
- **Queued deposits.** A deposit activates only in settlement. Once its epoch has started, a Junior deposit cannot be cancelled, because `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive` (lines 851–860). Its exceptions are a rejected queue, a terminal wipe or Senior impairment, and none of these applies to a solvent pool. The USDG stays in the vault.
- **New deposits.** New deposit requests are refused, because `canAcceptTrancheDeposits` is false while degraded (`HousePool.sol` lines 1139–1145, `TrancheVault.sol` line 1319).
- **Position holders.** Withdrawals by any account with an open position revert `CfdEngine__DegradedMode` in the engine's withdraw guard (`CfdEngineSettlementSidecar.sol` lines 255–276).

### The latch itself is designed and reachable by any trader

The latch is documented design and I do not dispute it. "The orthogonal latch D moves from `Healthy` to `Degraded` when a permitted terminal close or liquidation leaves projected physical assets net of claims below the remaining raw endpoint envelope … It does not auto-clear: after genuine recapitalization or risk reduction restores the balance sheet, an explicit owner action is required to return it to `Healthy`" (`WHITEPAPER.md` lines 1095–1099). Section 2.3 adds that "the live protocol … may enter degraded mode after a terminal action" (line 290). Section 7.5 lists what it blocks and states that "the owner may clear the latch through the explicit recovery path" (lines 1322–1342). `EMERGENCY_RESPONSE_GUIDE.md` lines 45–47 say the same: "It blocks new risk, LP entry, and all LP epoch settlement. The Engine owner can call `clearDegradedMode()` once raw `E >= L`."

So a latch needs no attacker. Any trader whose profitable close leaves `E < L` sets it, for example after a sharp basket move on a book with a large opposite-side envelope. The documented way back is an owner who, once `E >= L`, calls `clearDegradedMode()`. This report is about what happens when that owner no longer exists: the designed latch becomes permanent.

### The earlier renounce finding was fixed incompletely, and renouncing the engine is a supported state

The Cecuro AI audit (October 16, 2026) reported LOW-59, "Owner can leave emergency recovery ownerless". The root cause is the same: perps contracts inherit `Ownable.renounceOwnership()` and lose their owner-only recovery paths. Its stated impact was "LP epoch settlement can become permanently unavailable after a later pause." Its Remediation reads: "Override renounceOwnership() **in every governance-bearing perps contract** and make it revert, or replace Ownable2Step with an ownership module that removes renounce entirely."

The resolution ("Addressed", fix commit `1151d01`) overrode `renounceOwnership()` in only three contracts: HousePool, OrderRouterAdmin and EmergencyPauseCoordinator. It did not cover `CfdEngine`. Yet `CfdEngine` holds the one recovery path that every LP exit depends on. The program says such a report is valid only "if it shows that the fix at commit a82d4fef… is incomplete or introduces a new bug". This report shows that the fix is incomplete, with a stronger trigger than the original finding. LOW-59 needed a later privileged pause to freeze LP settlement. Here the freeze is set by an unprivileged trader's ordinary close.

Renouncing the engine is also a documented and tested state, not a misuse. `SECURITY.md` line 128, on the treasury schedule, says: "Ownership transfer preserves the schedule; renouncing ownership does not prevent activation." The repository's `test_TreasuryProposal_ActivationDoesNotRequireAnOwner` (`test/perps/spec/admin/TreasuryProposal.t.sol` lines 340–345) calls `engine.renounceOwnership()` and asserts `engine.owner() == address(0)`. Supporting an ownerless engine is not the same as accepting a permanent LP freeze. No document says that an ownerless engine cannot leave degraded mode, or that LP capital becomes unrecoverable after the next latch. `SECURITY.md` is the only document that mentions renouncing. The program's scope rules cover this case: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol", and an impact "that arises from a logic error that a privileged address can trigger … remains valid".

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack. The release risk parameters, 1 USDG tranche seeds, the 40,000,000 USDG / 80% Senior cap and the release Router configuration are applied through the owner's 72-hour timelocked config paths. Every LP action goes through the public `TrancheVault` / `HousePool` / `OrderRouter.settleLpEpoch` entry points, and every trade through the public V3 `OrderRouter.commitOrder` / `executeOrder` with keeper execution. Marks go through `OrderRouter.updateMarkPrice`. Prices come from the suite's Pyth mock, and USDG is the suite's mock token. There is no `vm.store`, `deal`, `etch` or `mockCall` on in-scope contracts. In the attack branch, the only privileged call is `CfdEngine.renounceOwnership()`.

1. A Junior LP deposits 1,030,000 USDG and a Senior LP 300,000 USDG through the public request/settle/claim path.
2. A trader (the "holder") deposits 10,000 USDG and opens a 2,000 USDG LONG with 1,000 USDG margin. A new LP queues a 100,000 USDG Junior deposit.
3. The engine owner calls `CfdEngine.renounceOwnership()`, and `engine.owner()` becomes `address(0)`. A separate test shows that the same call reverts on `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`. It succeeds on `CfdEngine`, `CfdEngineAdmin` and `MarginClearinghouse`.
4. Ordinary trading follows. Two traders open three 432,000 USDG LONG chunks and three 432,000 USDG SHORT chunks. The basket rises 10%, and the SHORT trader closes for a 121,365.829270 USDG settlement gain. That close leaves `E < L`, and the engine latches `degradedMode`, as documented.
5. The Junior LP requests redemption of half its shares, and the Senior LP half of its shares, after the one-hour post-deposit cooldown.
6. Solvency is fully restored. The basket returns to 1.00 and the large LONG closes. Only the holder's 2,000 USDG LONG remains, so `E` = 1,207,037.730007 USDG and `L` = 2,000.000000 USDG. The time then advances until every request's epoch has started.
7. Nobody can clear the latch. `clearDegradedMode()`, `recapitalizePool()`, `acceptOwnership()` and `transferOwnership()` revert with `OwnableUnauthorizedAccount` for the former owner, a keeper, an LP and an outsider. `pendingOwner()` is `address(0)`.
8. LP funds are frozen. A keeper's `OrderRouter.settleLpEpoch` with a fresh signed tick reverts `HousePool__DegradedMode`.
   - **Redemptions:** the matured Junior and Senior requests have `claimableRedeemRequest == 0`, and `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable`.
   - **Queued deposit:** it has `claimableDepositRequest == 0`, and `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive`.
   - **Holder:** withdrawing 5,000 of its 8,999.800000 USDG free settlement reverts `CfdEngine__DegradedMode`.
   - **Balances:** `HousePool` holds 1,207,037.730007 USDG, and the Junior vault holds the 100,000.000000 USDG deposit escrow.
9. Forever. The time advances 52 weeks and the mark is refreshed. `E` and `L` are unchanged and `degradedMode` is still `true`. Every call in steps 7 and 8 reverts the same way. The USDG in `HousePool` and in the Junior vault, the stored tranche principals and the holder's free settlement are asserted unchanged.
10. No position remains. The holder closes its position and withdraws 9,998.154518 USDG. This is the only way out for a trader, and it means leaving the market. With no open position at all, a direct `HousePool.settleLpEpoch(0, 0)` still reverts `HousePool__DegradedMode`. Neither public mark updates nor a new open order clear the latch. `HousePool` still holds 1,207,037.775489 USDG of LP capital.
11. Control: the same sequence without step 3. The owner calls `clearDegradedMode()` once `E >= L`, and one keeper settlement funds both matured redemptions in full and activates the queued deposit. The Junior LP receives 453,515.180680 USDG and the Senior LP 150,002.743911 USDG. The depositor's activated shares are worth 100,000.000990 USDG. The holder withdraws 5,000.000000 USDG while keeping its position open.

## Impact Details

**Critical — Permanent freezing of funds.**

Every LP exit and entry in Plether Perps goes through `HousePool.settleLpEpoch`. After the engine owner has renounced, the first degraded-mode latch blocks that settlement for good. The latch can come from any trader's ordinary profitable close or from a liquidation. The block cannot be lifted by recapitalization, by solvency recovery, by time, or by any public call. In the proof of concept:

- **LP capital: 1,207,037.730007 USDG in `HousePool`.** By the vaults' own views, that is 300,006.489393 USDG Senior and 907,031.240614 USDG Junior. No share can ever be redeemed. Over the 52 weeks, the views move a Senior coupon of 23,934.328002 USDG from Junior to Senior on paper, but no USDG can be paid to either.
- **Matured redemption requests.** The Junior request is worth 453,515.180680 USDG and the Senior request 150,002.743911 USDG. Both are part of the capital above. They are unfunded and cannot be cancelled.
- **Queued deposit escrow: 100,000.000000 USDG** in the Junior vault. It can never activate or be cancelled.
- **Position holders.** The holder's free settlement (8,999.800000 USDG) cannot be withdrawn while its position is open. A trader can recover funds only by closing, as the holder did (9,998.154518 USDG), so I do not count trader funds as permanently frozen. The market is also permanently close-only, because new opens are refused.

Total permanently frozen in the proof of concept: 1,307,037.730007 USDG, made up of LP capital plus queued deposit escrow.

Preconditions, stated plainly:
- **The engine owner renounces.** This is a documented, tested and still-callable action of the trusted owner, not a compromise or misuse (see the last Vulnerability Details subsection). On the live deployment the engine currently has an owner and `degradedMode()` is `false`.
- **A latch follows at some later time.** It needs no attacker, because it is the protocol's designed response to an ordinary close that leaves `E < L`.

The remaining owners (`HousePool`, `OrderRouterAdmin`) cannot help. `HousePool`'s owner powers do not settle epochs in degraded mode or move LP backing ("these powers do not permit withdrawal of existing claimant backing", `SECURITY.md` line 245). By code reading, and not exercised in the proof of concept, the one partial exception is a queued **Senior** deposit: it becomes cancellable if the `HousePool` owner pauses LP settlement (`TrancheVault.sol` line 853). Junior deposits, all redemptions and all LP capital have no such path.

## Recommendation

1. Finish the earlier remediation. Override `renounceOwnership()` to revert in `CfdTreasury`, so that `CfdEngine` inherits the override, and also in `CfdEngineAdmin` and `MarginClearinghouse`, as was already done in `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`:

```solidity
/// @notice Ownership cannot be renounced: degraded-mode recovery and recapitalization are owner-only.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

2. Independently of ownership, do not make the only exit from a solvency-gated latch depend on a live owner. The clear condition is fully on-chain (`_availableCashForFreshPoolPayouts() >= _maxLiability()`). The latch can therefore be cleared permissionlessly once that condition holds, optionally after a minimum delay, or at least permissionlessly when `owner() == address(0)`:

```solidity
function clearDegradedMode() external {
    if (msg.sender != owner() && owner() != address(0)) revert OwnableUnauthorizedAccount(msg.sender);
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
    emit DegradedModeCleared();
}
```

3. Add a regression test that renounces the engine and asserts that the call reverts. If option 2 is chosen, add a test that a latched, ownerless engine with `E >= L` can be cleared and that LP settlement then completes.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/RenouncedEngineDegradedFreeze.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract RenouncedEngineDegradedFreezePoC -vv
```

The file contains three tests:
- `test_IncompleteFix_EngineStillRenounces`: `renounceOwnership()` reverts on `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`, and succeeds on `CfdEngine`, `CfdEngineAdmin` and `MarginClearinghouse` (`owner() == address(0)`);
- `test_RenouncedEngine_LatchFreezesLpFundsForever`: the renounce, the ordinary latch, restored solvency, the "nobody can clear" checks, every frozen path, and the same checks 52 weeks later and with no open position;
- `test_Control_OwnerClearsAndLpExitsComplete`: the same sequence without the renounce; the owner clears and every exit completes.

The suite's Router is the repository's test harness, which only adds a legacy scalar `commitOrder` overload. The proof uses the production V3 `commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch` entry points.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice The renounce fix covers HousePool, OrderRouterAdmin and EmergencyPauseCoordinator but not CfdEngine, the
///         only contract that can clear degraded mode. After the engine owner renounces, the next ordinary latch
///         (an unprivileged trader's profitable close that leaves E < L, the documented mechanism) can never be
///         cleared: LP epoch settlement reverts forever, so LP capital, matured redemptions and queued deposit escrow
///         are frozen permanently, even after solvency is fully restored.
///
///         Public calls only: LP capital through TrancheVault / HousePool / OrderRouter.settleLpEpoch, trades through
///         OrderRouter.commitOrder / executeOrder, marks through OrderRouter.updateMarkPrice. The only privileged call
///         in the attack branch is CfdEngine.renounceOwnership(). Pyth (MockPyth) and USDG (mock token) are the only
///         mocks; no vm.store / deal / etch / mockCall on in-scope contracts. Release risk, Router and Senior-capacity
///         parameters, 1 USDG tranche seeds.
contract RenouncedEngineDegradedFreezePoC is BasePerpTest {

    address internal constant LONG_ACCOUNT = address(0xA770C1);
    address internal constant SHORT_ACCOUNT = address(0xA770C2);
    address internal constant HOLDER = address(0xA770C3); // trader who keeps a small position open throughout
    address internal constant SENIOR_LP = address(0x5E1105);
    address internal constant JUNIOR_LP = address(0x1D1105);
    address internal constant DEPOSITOR = address(0x1D3001); // Junior deposit queued before the latch
    address internal constant KEEPER = address(0xE0EC);
    address internal constant OUTSIDER = address(0x0B5E);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant UP_PRICE = 110_000_000; // +10%
    uint256 internal constant CHUNK_SIZE = 432_000e18;
    uint256 internal constant CHUNKS_PER_LEG = 3;
    uint256 internal constant LONG_CHUNK_MARGIN = 50_000e6;
    uint256 internal constant SHORT_CHUNK_MARGIN = 20_000e6;
    uint256 internal constant JUNIOR_DEPOSIT = 1_030_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 300_000e6;
    uint256 internal constant QUEUED_DEPOSIT = 100_000e6;
    uint256 internal constant HOLDER_FUNDS = 10_000e6;
    uint256 internal constant HOLDER_SIZE = 2_000e18;
    uint256 internal constant HOLDER_MARGIN = 1_000e6;
    uint256 internal constant HOLDER_WITHDRAW = 5_000e6;

    uint256 internal orderNonce;

    struct Requests {
        uint256 juniorRedeem;
        uint256 juniorShares;
        uint256 seniorRedeem;
        uint256 seniorShares;
        uint256 deposit;
    }

    struct Frozen {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 juniorQueuedValue;
        uint256 seniorQueuedValue;
        uint256 depositEscrow;
        uint256 holderFree;
        uint256 poolUsdg;
        uint256 vaultUsdg;
    }

    // ------------------------------------------------------------------ configuration (release values)

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // all LP capital enters through the public request/settle/claim path
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig(); // 72 h timelock
        _setSeniorCapacity(40_000_000e6, 8_000); // 72 h timelock
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live market
        _refreshMark(ENTRY_PRICE);

        uint256 jr = _requestDeposit(juniorVault, JUNIOR_LP, JUNIOR_DEPOSIT);
        _warpTo(juniorVault.depositEpochStart(jr));
        _refreshMark(ENTRY_PRICE);
        pool.settleLpEpoch(0, 0); // no open positions: permissionless
        _claimDeposit(juniorVault, JUNIOR_LP, jr);
        uint256 sr = _requestDeposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT);
        _warpTo(seniorVault.depositEpochStart(sr));
        _refreshMark(ENTRY_PRICE);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, sr);

        _fundTrader(LONG_ACCOUNT, CHUNKS_PER_LEG * LONG_CHUNK_MARGIN + 20_000e6);
        _fundTrader(SHORT_ACCOUNT, CHUNKS_PER_LEG * SHORT_CHUNK_MARGIN + 20_000e6);
        _fundTrader(HOLDER, HOLDER_FUNDS);
        _refreshMark(ENTRY_PRICE);
    }

    // ------------------------------------------------------------------ tests

    /// @notice 1. The renounce fix is incomplete: three contracts revert, the engine (and two other Ownable2Step perps
    ///         contracts) still renounce.
    function test_IncompleteFix_EngineStillRenounces() external {
        EmergencyPauseCoordinator coordinator =
            new EmergencyPauseCoordinator(address(routerAdmin), address(pool), address(this));
        bytes memory invalidOwner = abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0));
        vm.expectRevert(invalidOwner);
        pool.renounceOwnership();
        vm.expectRevert(invalidOwner);
        routerAdmin.renounceOwnership();
        vm.expectRevert(invalidOwner);
        coordinator.renounceOwnership();
        assertEq(pool.owner(), address(this));
        assertEq(routerAdmin.owner(), address(this));
        assertEq(coordinator.owner(), address(this));
        emit log("HousePool / OrderRouterAdmin / EmergencyPauseCoordinator: renounceOwnership reverts");

        engine.renounceOwnership();
        engineAdmin.renounceOwnership();
        clearinghouse.renounceOwnership();
        assertEq(engine.owner(), address(0), "engine still owned");
        assertEq(engineAdmin.owner(), address(0), "engine admin still owned");
        assertEq(clearinghouse.owner(), address(0), "clearinghouse still owned");
        emit log("CfdEngine / CfdEngineAdmin / MarginClearinghouse: renounceOwnership succeeds, owner() == 0");
    }

    /// @notice 2-5. Engine renounced; an ordinary trader's profitable close latches degraded mode; nobody can clear
    ///         it; LP funds stay frozen after solvency is restored and 52 weeks pass.
    function test_RenouncedEngine_LatchFreezesLpFundsForever() external {
        Requests memory q = _scenario(true);

        // 3. Nobody can clear the latch, and no ownership path remains.
        _assertNobodyCanClear();

        // 4. LP funds frozen (solvency already restored: E >= L).
        Frozen memory a = _assertFrozen(q, "AFTER RESTORED SOLVENCY");

        // 5. Forever: 52 weeks later, still solvent, still latched, every path still blocked, amounts unchanged.
        vm.warp(vm.getBlockTimestamp() + 52 weeks);
        _refreshMark(ENTRY_PRICE);
        _assertSolvent("52 WEEKS LATER");
        _assertNobodyCanClear();
        Frozen memory b = _assertFrozen(q, "52 WEEKS LATER");
        assertEq(b.seniorPrincipal, a.seniorPrincipal, "senior principal moved");
        assertEq(b.juniorPrincipal, a.juniorPrincipal, "junior principal moved");
        assertEq(b.poolUsdg, a.poolUsdg, "pool USDG moved");
        assertEq(b.holderFree, a.holderFree, "holder free settlement moved");
        assertEq(b.depositEscrow, a.depositEscrow, "deposit escrow moved");
        assertEq(b.vaultUsdg, a.vaultUsdg, "vault escrow USDG moved");

        // The position holder can only leave by closing; LP settlement still reverts with no positions at all.
        _closeLeg(HOLDER, CfdTypes.Side.LONG, ENTRY_PRICE);
        uint256 holderFree = clearinghouse.getAccountAssetBuckets(HOLDER).freeSettlementAssets;
        vm.prank(HOLDER);
        clearinghouse.withdrawMargin(holderFree);
        emit log_named_decimal_uint("holder withdrew after closing its position (USDG)", holderFree, 6);
        assertEq(_maxLiability(), 0, "positions remain");
        vm.prank(KEEPER);
        vm.expectRevert(bytes4(keccak256("HousePool__DegradedMode()")));
        pool.settleLpEpoch(0, 0);
        assertTrue(engine.degradedMode(), "latch cleared");
        emit log("no open positions, E >= L, 52 weeks: HousePool.settleLpEpoch still reverts HousePool__DegradedMode");
        emit log_named_decimal_uint("LP USDG still in HousePool", settlementToken.balanceOf(address(pool)), 6);

        // Public actions available to anyone leave the latch set.
        _permissionlessActionsDoNotClear();
    }

    /// @notice 6. Control: the same sequence without the renounce. The owner clears once E >= L and every LP exit,
    ///         the queued deposit and the holder's withdrawal complete.
    function test_Control_OwnerClearsAndLpExitsComplete() external {
        Requests memory q = _scenario(false);
        engine.clearDegradedMode(); // documented owner recovery once raw E >= L
        assertFalse(engine.degradedMode(), "not cleared");

        _nextBlock();
        vm.prank(KEEPER);
        router.settleLpEpoch(_liveTick(ENTRY_PRICE));

        uint256 jClaim = juniorVault.claimableRedeemRequest(q.juniorRedeem, JUNIOR_LP);
        uint256 sClaim = seniorVault.claimableRedeemRequest(q.seniorRedeem, SENIOR_LP);
        assertEq(jClaim, q.juniorShares, "junior redemption not fully funded");
        assertEq(sClaim, q.seniorShares, "senior redemption not fully funded");
        uint256 jBefore = settlementToken.balanceOf(JUNIOR_LP);
        uint256 sBefore = settlementToken.balanceOf(SENIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(q.juniorRedeem, jClaim, JUNIOR_LP, JUNIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(q.seniorRedeem, sClaim, SENIOR_LP, SENIOR_LP);
        _claimDeposit(juniorVault, DEPOSITOR, q.deposit);
        uint256 hBefore = settlementToken.balanceOf(HOLDER);
        vm.prank(HOLDER);
        clearinghouse.withdrawMargin(HOLDER_WITHDRAW);

        emit log_named_decimal_uint("CONTROL: Junior LP USDG received", settlementToken.balanceOf(JUNIOR_LP) - jBefore, 6);
        emit log_named_decimal_uint("CONTROL: Senior LP USDG received", settlementToken.balanceOf(SENIOR_LP) - sBefore, 6);
        emit log_named_decimal_uint(
            "CONTROL: queued deposit activated (Junior shares value)",
            juniorVault.convertToAssets(juniorVault.balanceOf(DEPOSITOR)),
            6
        );
        emit log_named_decimal_uint(
            "CONTROL: holder withdrew with its position open (USDG)", settlementToken.balanceOf(HOLDER) - hBefore, 6
        );
    }

    // ------------------------------------------------------------------ scenario

    function _scenario(
        bool renounce
    ) internal returns (Requests memory q) {
        // A trader holds a small LONG; a new LP queues a Junior deposit (both before anything happens).
        _open(HOLDER, CfdTypes.Side.LONG, HOLDER_SIZE, HOLDER_MARGIN);
        q.deposit = _requestDeposit(juniorVault, DEPOSITOR, QUEUED_DEPOSIT);

        if (renounce) {
            engine.renounceOwnership(); // the only privileged call in the attack branch
            assertEq(engine.owner(), address(0), "engine still owned");
            emit log("engine owner renounced ownership");
        }

        // 2. Ordinary trading: a hedge, then the SHORT side closes in profit after a +10% move. E < L latches.
        for (uint256 i; i < CHUNKS_PER_LEG; ++i) {
            _open(LONG_ACCOUNT, CfdTypes.Side.LONG, CHUNK_SIZE, LONG_CHUNK_MARGIN);
            _open(SHORT_ACCOUNT, CfdTypes.Side.SHORT, CHUNK_SIZE, SHORT_CHUNK_MARGIN);
        }
        assertFalse(engine.degradedMode(), "hedge latched");
        _refreshMark(UP_PRICE);
        uint256 shortBefore = clearinghouse.balanceAssets(SHORT_ACCOUNT);
        _closeLeg(SHORT_ACCOUNT, CfdTypes.Side.SHORT, UP_PRICE);
        assertTrue(engine.degradedMode(), "winning close did not latch");
        emit log_named_decimal_uint(
            "degraded mode latched by an ordinary winning SHORT close; trader's settlement gain (USDG)",
            clearinghouse.balanceAssets(SHORT_ACCOUNT) - shortBefore,
            6
        );

        // LPs ask to leave (after their one-hour post-deposit cooldown).
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        q.juniorShares = juniorVault.balanceOf(JUNIOR_LP) / 2;
        vm.prank(JUNIOR_LP);
        q.juniorRedeem = juniorVault.requestRedeem(q.juniorShares, JUNIOR_LP, JUNIOR_LP);
        q.seniorShares = seniorVault.balanceOf(SENIOR_LP) / 2;
        vm.prank(SENIOR_LP);
        q.seniorRedeem = seniorVault.requestRedeem(q.seniorShares, SENIOR_LP, SENIOR_LP);

        // Solvency fully restored: the basket returns to entry and the large LONG closes. Only the holder's small
        // position remains.
        _refreshMark(ENTRY_PRICE);
        _closeLeg(LONG_ACCOUNT, CfdTypes.Side.LONG, ENTRY_PRICE);
        _assertSolvent("RESTORED");

        // Every request has matured.
        uint256 matured = pool.lpEpochStart(q.juniorRedeem);
        if (pool.lpEpochStart(q.seniorRedeem) > matured) matured = pool.lpEpochStart(q.seniorRedeem);
        if (juniorVault.depositEpochStart(q.deposit) > matured) matured = juniorVault.depositEpochStart(q.deposit);
        _warpTo(matured + 10);
        _refreshMark(ENTRY_PRICE);
    }

    function _assertSolvent(
        string memory tag
    ) internal {
        uint256 e = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 l = _maxLiability();
        assertGe(e, l, "E < L");
        emit log(tag);
        emit log_named_decimal_uint("  E = pool assets net of trader claims", e, 6);
        emit log_named_decimal_uint("  L = max directional liability", l, 6);
    }

    function _assertNobodyCanClear() internal {
        address[4] memory callers = [address(this), KEEPER, JUNIOR_LP, OUTSIDER]; // address(this) = former owner
        for (uint256 i; i < callers.length; ++i) {
            vm.prank(callers[i]);
            vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, callers[i]));
            engine.clearDegradedMode();
            vm.prank(callers[i]);
            vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, callers[i]));
            engine.recapitalizePool(1e6);
            vm.prank(callers[i]);
            vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, callers[i]));
            engine.acceptOwnership();
            vm.prank(callers[i]);
            vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, callers[i]));
            engine.transferOwnership(callers[i]);
        }
        assertEq(engine.owner(), address(0), "owner restored");
        assertEq(engine.pendingOwner(), address(0), "pending owner exists");
        assertTrue(engine.degradedMode(), "latch cleared");
        emit log("clearDegradedMode / recapitalizePool / acceptOwnership / transferOwnership revert for every caller");
    }

    function _assertFrozen(
        Requests memory q,
        string memory tag
    ) internal returns (Frozen memory f) {
        // LP epoch settlement: any keeper, through the Router with a fresh signed tick.
        _nextBlock();
        bytes[] memory tick = _liveTick(ENTRY_PRICE);
        vm.prank(KEEPER);
        vm.expectRevert(bytes4(keccak256("HousePool__DegradedMode()")));
        router.settleLpEpoch(tick);

        // Matured redemptions are unfunded and cannot be cancelled.
        assertEq(juniorVault.claimableRedeemRequest(q.juniorRedeem, JUNIOR_LP), 0, "junior redemption funded");
        assertEq(seniorVault.claimableRedeemRequest(q.seniorRedeem, SENIOR_LP), 0, "senior redemption funded");
        vm.prank(JUNIOR_LP);
        vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")));
        juniorVault.cancelRedeemRequest(q.juniorRedeem, JUNIOR_LP, JUNIOR_LP);
        vm.prank(SENIOR_LP);
        vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")));
        seniorVault.cancelRedeemRequest(q.seniorRedeem, SENIOR_LP, SENIOR_LP);

        // The queued deposit cannot activate and cannot be cancelled.
        assertEq(juniorVault.claimableDepositRequest(q.deposit, DEPOSITOR), 0, "deposit activated");
        vm.prank(DEPOSITOR);
        vm.expectRevert(bytes4(keccak256("TrancheVault__DepositEpochAlreadyActive()")));
        juniorVault.cancelPendingDeposit(q.deposit);

        // A position holder cannot withdraw free settlement.
        f.holderFree = clearinghouse.getAccountAssetBuckets(HOLDER).freeSettlementAssets;
        assertGt(f.holderFree, HOLDER_WITHDRAW, "holder free settlement too small");
        vm.prank(HOLDER);
        vm.expectRevert(bytes4(keccak256("CfdEngine__DegradedMode()")));
        clearinghouse.withdrawMargin(HOLDER_WITHDRAW);

        f.seniorPrincipal = pool.seniorPrincipal();
        f.juniorPrincipal = pool.juniorPrincipal();
        f.seniorQueuedValue = seniorVault.convertToAssets(q.seniorShares);
        f.juniorQueuedValue = juniorVault.convertToAssets(q.juniorShares);
        f.depositEscrow = juniorVault.pendingDepositEscrowAssets();
        f.poolUsdg = settlementToken.balanceOf(address(pool));
        f.vaultUsdg = settlementToken.balanceOf(address(juniorVault));
        emit log(tag);
        emit log("  LP epoch settlement reverts HousePool__DegradedMode; redemptions unfunded, not cancellable;");
        emit log("  queued deposit not activated, not cancellable; holder withdrawal reverts CfdEngine__DegradedMode");
        emit log_named_decimal_uint("  Senior LP principal (no exit path)", f.seniorPrincipal, 6);
        emit log_named_decimal_uint("  Junior LP principal (no exit path)", f.juniorPrincipal, 6);
        emit log_named_decimal_uint("  LP principal total", f.seniorPrincipal + f.juniorPrincipal, 6);
        emit log_named_decimal_uint("  matured Junior redemption (share value)", f.juniorQueuedValue, 6);
        emit log_named_decimal_uint("  matured Senior redemption (share value)", f.seniorQueuedValue, 6);
        emit log_named_decimal_uint("  queued Junior deposit escrow", f.depositEscrow, 6);
        emit log_named_decimal_uint("  position holder free settlement (blocked while the position is open)", f.holderFree, 6);
        emit log_named_decimal_uint("  Senior tranche value (all Senior shares)", seniorVault.totalAssets(), 6);
        emit log_named_decimal_uint("  Junior tranche value (all Junior shares)", juniorVault.totalAssets(), 6);
        emit log_named_decimal_uint("  USDG held by HousePool", f.poolUsdg, 6);
        emit log_named_decimal_uint("  USDG held by the Junior vault", f.vaultUsdg, 6);
    }

    /// @dev Mark refreshes, a settlement attempt and a new risk-increasing order by anyone: none clears the latch.
    function _permissionlessActionsDoNotClear() internal {
        _refreshMark(ENTRY_PRICE);
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open-after-latch", OUTSIDER, ++orderNonce));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = CHUNK_SIZE / 100;
        request.marginDelta = 1_000e6;
        request.targetPrice = 1;
        request.bounds = _bounds();
        _fundTrader(OUTSIDER, 2_000e6);
        vm.prank(OUTSIDER);
        try router.commitOrder(request) returns (uint64 orderId) {
            OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, ENTRY_PRICE);
            assertTrue(result.status != OrderV3Types.LifecycleStatus.Executed, "open executed while degraded");
        } catch {}
        assertTrue(engine.degradedMode(), "latch cleared by a public action");
        emit log("public mark updates, settlement attempts and new orders leave degradedMode == true");
    }

    // ------------------------------------------------------------------ LP helpers (public path)

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

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) vm.warp(t);
    }

    // ------------------------------------------------------------------ Router helpers (public V3 path, Pyth mocked)

    function _open(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, ENTRY_PRICE);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        _refreshMark(ENTRY_PRICE);
    }

    function _closeLeg(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = _positionSize(account);
        request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, price);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close failed");
        assertEq(_positionSize(account), 0, "leg not closed");
    }

    /// @dev Live v2.0.0 Router configuration, applied through the owner's 72 h timelocked config path.
    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
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
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = router.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        bytes[] memory tick = _ticks(price, publishTime, commitTime);
        vm.prank(keeper);
        result = router.executeOrder(orderId, tick);
    }

    function _liveTick(
        uint256 price
    ) internal view returns (bytes[] memory) {
        return _ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime());
    }

    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _nextBlock();
        router.updateMarkPrice(_liveTick(price));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}
```

</details>

### Observed balances:

| Measurement (USDG) | Renounced engine, after solvency restored | Renounced engine, 52 weeks later | Control: owner clears |
| --- | ---: | ---: | ---: |
| `E` = pool assets net of trader claims | 1,207,037.730007 | 1,207,037.730007 | 1,207,037.730007 |
| `L` = maximum directional liability | 2,000.000000 | 2,000.000000 | 2,000.000000 |
| `degradedMode` | `true`, nobody can clear | `true`, nobody can clear | cleared by the owner |
| Keeper LP settlement (`OrderRouter.settleLpEpoch`) | reverts `HousePool__DegradedMode` | reverts `HousePool__DegradedMode` | succeeds |
| LP capital held by `HousePool` | 1,207,037.730007 frozen | 1,207,037.730007 frozen | redeemable |
| Senior tranche value (all Senior shares) | 300,006.489393 | 323,940.817395 | redeemable |
| Junior tranche value (all Junior shares) | 907,031.240614 | 883,096.912612 | redeemable |
| Matured Junior redemption (half the Junior LP's shares) | 453,515.180680 unfunded, not cancellable | 441,548.028280 unfunded, not cancellable | paid 453,515.180680 |
| Matured Senior redemption (half the Senior LP's shares) | 150,002.743911 unfunded, not cancellable | 161,969.867959 unfunded, not cancellable | paid 150,002.743911 |
| Queued Junior deposit escrow | 100,000.000000, not activated, not cancellable | 100,000.000000, not activated, not cancellable | activated (shares worth 100,000.000990) |
| Holder free settlement, position open | 8,999.800000, withdrawal reverts `CfdEngine__DegradedMode` | 8,999.800000, withdrawal reverts | withdrew 5,000.000000 |
| **Permanently frozen (LP capital + deposit escrow)** | **1,307,037.730007** | **1,307,037.730007** | **0** |

After the 52 weeks, the holder closes its position and withdraws 9,998.154518 USDG. With no open position left, `HousePool.settleLpEpoch(0, 0)` still reverts `HousePool__DegradedMode`, and `HousePool` still holds 1,207,037.775489 USDG. The trader that latched the engine realized a 121,365.829270 USDG settlement gain on its close.
