# 89847 sc medium unclearable claim flag in rescuewithdrawfromblocklisted causes permanent freezing of staker withdrawals

**Submitted on Aug 21st 2026 at 13:45:28 UTC by @Emmy for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89847
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` permanently marks the rescued account as having claimed the affected withdrawal period, and no function anywhere in the codebase ever clears that mark. The mark is keyed by `(period, account)` rather than by the amount that was actually rescued, so it does not merely neutralise the balance that was moved: it disqualifies the account from that period's withdrawal bucket for the lifetime of the contract. If the account is later removed from the blocklist and registers a new withdrawal while the same period is still current, those assets become permanently unclaimable. In production this permanently freezes staker collateral. The tokens remain in the vault, the vault continues to report them as owed to the user through `withdrawalsOf`, and no role, rescue control, or subsequent operator action can release them. The frozen amount is also excluded from `totalAssets()` forever, which suppresses share value for every remaining staker and understates the collateral that backs underwriting capacity.

## Vulnerability Details

### Background: how withdrawals settle

`FirelightVault` uses a two-phase, period-based withdrawal model. Calling `withdraw()` or `redeem()` does not transfer assets. It burns the caller's shares immediately and registers a claim against a future period bucket, computed in `_requestWithdraw` (`FirelightVault.sol:994`):

```solidity
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(
    assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
...
_update(owner, address(0), shares);
```

Two properties of this function matter for the vulnerability. First, every withdrawal made during the same period lands in the **same bucket**, because `period` is always `currentPeriod() + 1`, and the credit uses `+=`, so repeated requests accumulate. Second, the credit is applied to `withdrawSharesOf[period][receiver]` with no reference to any claim state, so a fresh balance can be written into a bucket the account is already barred from.

The user later collects with `claimWithdraw(period)` once the period has elapsed (`FirelightVault.sol:788`):

```solidity
function claimWithdraw(
    uint256 period
) external whenNotPaused notBlocklisted(_msgSender()) nonReentrant returns (uint256 assets) {
    if (period >= currentPeriod()) revert InvalidPeriod();

    address sender = _msgSender();
    if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // line 794

    assets = _convertToAssetsTotals(
        withdrawSharesOf[period][sender],
        withdrawShares[period],
        withdrawAssets[period],
        Math.Rounding.Floor
    );

    if (assets == 0) revert NoWithdrawalAmount(period);

    pendingWithdrawAssets -= assets;
    isWithdrawClaimed[period][sender] = true;                                     // line 806

    IERC20(asset()).safeTransfer(sender, assets);

    emit CompleteWithdraw(sender, assets, period);
}
```

The check at line 794 is an unconditional veto. It runs **before** any amount is computed, so once the flag is set the function can never pay out for that period regardless of what the account is owed.

### The defect

`rescueWithdrawFromBlocklisted` (`FirelightVault.sol:845`) moves a blocklisted account's pending withdrawal to a safe destination:

```solidity
uint256[] memory rescuedShares = new uint256[](len);
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);  // line 864
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);    // line 865

    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;                                            // line 869
    isWithdrawClaimed[periods[i]][from] = true;                                        // line 870

    rescuedShares[i] = _withdrawOf;
}
```

Line 870 is the defect, for four compounding reasons.

**1. The flag is never cleared.** `isWithdrawClaimed` is declared in `FirelightVaultStorage.sol:71` and written in exactly two places in the entire in-scope tree, line 806 in `claimWithdraw` and line 870 here. It is set to `false` nowhere. `removeFromBlocklist` (`FirelightVault.sol:560`) reverses the blocklist designation and touches nothing else:

```solidity
function removeFromBlocklist(address account) external onlyRole(BLOCKLIST_ROLE) onlyBlocklisted(account) {
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

An account restored to good standing therefore carries the claim flag for every period it was ever rescued from, permanently.

**2. The flag is redundant.** Line 869 already sets `withdrawSharesOf[periods[i]][from] = 0` in the same iteration. That alone makes a double claim impossible: with a zero share balance, `claimWithdraw` computes `assets == 0` and reverts `NoWithdrawalAmount(period)`. Line 870 adds no protection that line 869 does not already provide, and it is the sole cause of the permanent lock. This is demonstrated directly by the negative control below, where deleting line 870 leaves the double-claim guard fully intact while restoring the user's access to their own funds.

**3. The flag is keyed by period, not by amount.** The rescue moves a specific balance, but the flag disqualifies the account from the entire bucket. Since `_requestWithdraw` credits `withdrawSharesOf[period][receiver] += sharesWithdraw` for every withdrawal made during the same period, any balance registered after the rescue lands in a bucket the account can never claim. The vault records the debt and then refuses to honour it: after the second withdrawal, `withdrawalsOf(bucket, victim)` returns the full owed amount while `claimWithdraw` reverts.

**4. The flag blocks its own remediation.** The natural recovery would be a second rescue moving the stranded balance to a fresh address. That path is gated at line 864 on the very flag line 870 set, so it reverts `AlreadyClaimedPeriod`. The escape hatch is locked by its own key. `rescueSharesFromBlocklisted` does not help either, because the shares backing the stranded balance were already burned by `_requestWithdraw` at request time.

### Sequence that triggers it

Every step below is a supported operation performed correctly. No role acts outside its documented purpose, and the step that destroys the value is the user's own ordinary call.

1. A staker calls `withdraw()`. The request registers in bucket `p = currentPeriod() + 1`.
2. `BLOCKLIST_ROLE` calls `addToBlocklist(staker)` following a compliance signal.
3. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(staker, custody, [p])`, which is precisely what the function exists to do. Line 870 sets `isWithdrawClaimed[p][staker] = true`.
4. The compliance review clears the staker. `BLOCKLIST_ROLE` calls `removeFromBlocklist(staker)`, the protocol's first-class mechanism for reversing a designation. The claim flag survives.
5. The staker, now unblocked and with no on-chain indication of a problem, calls `withdraw()` again. The period has not rolled over, so the credit lands in the same bucket `p`.
6. Once `p` has elapsed, `claimWithdraw(p)` reverts `AlreadyClaimedPeriod(p)` and will do so forever.

The window for step 5 is a full period, which the contract enforces to be at least one day (`SMALLEST_PERIOD_DURATION`) and which the project's own fixtures set to one week.

A second reachable variant exists because `_requestWithdraw` credits the **receiver**, not the owner. Once the account is un-blocklisted, any third party calling `withdraw(assets, receiver = affectedAccount, owner = self)` also strands those assets in the same bucket.

### Why this is not covered by the published known issues

The known issues concern the permissioned MVP trust model, operator-authored cover orders, the first-loss buffer's USD peg and its unescrowed wallet, collateral drawdown, uncapped aggregate claims, permissioned assessment and payout, and off-chain distribution. None of them concerns the blocklist or rescue machinery. The 0xMacro A-1 report of 29 July 2026 covered `FirelightVault.sol` and raised M-1 (leverage), M-2 (batch settle atomicity), L-1 (zero-share deposit), L-2 (concentration cap), L-3 (grace bound), L-4 (CAR floor), G-1 (payout period scan) and I-1 through I-11. None of them is this defect, and none of its accepted-risk entries covers it.

This is also distinct from the historical fix in commit `a2def47`, "Fix rescuing withdrawal to a user that already claimed them", which introduced the `to`-side guard now at line 865. The `from`-side write at line 870 was left untouched by that change.

### Why this is an enforcement failure rather than intended behaviour

No privileged role misbehaves at any point. `RESCUER_ROLE` performs a rescue, which is its exact documented purpose. `BLOCKLIST_ROLE` reverses a designation using a function that exists solely for that purpose. The value is destroyed by the affected user's own unprivileged `withdraw()` call. The constraint that fails is the double-claim guard, which over-constrains and freezes funds that no role intended to freeze and no role can release.

The project's own test suite confirms this behaviour was never intended. `test/rescue.js` contains twenty rescue cases covering role gating, zero addresses, empty amounts, already-claimed periods, successful rescues, and the rescuer claiming rescued balances. `removeFromBlocklist` does not appear anywhere in that file, so the un-blocklist path following a rescue is untested rather than deliberate.

## Impact Details

**In-scope impact: permanent freezing of funds.**

**Direct loss to the affected staker.** In the proof of concept, 40,000 fXRP is permanently unclaimable. That is **26.66% of the affected user's 150,000 fXRP position** and **0.79% of the 5.05M fXRP vault**. The loss scales with the size of the post-rescue withdrawal, so a user who requests their full remaining balance loses all of it.

**The freeze is permanent.** All three recovery paths were attempted in the test and all three fail:

* `claimWithdraw(period)` reverts `AlreadyClaimedPeriod` at line 794.
* A second `rescueWithdrawFromBlocklisted` reverts `AlreadyClaimedPeriod` at line 864, because the rescue path reads the flag it previously set.
* `rescueSharesFromBlocklisted` succeeds against the user's residual share balance and restores nothing, because the shares backing the stranded amount were burned at request time.

No role in the system can release the funds. Recovery requires a contract upgrade.

**Repeatable across accounts.** The proof of concept runs the sequence against five independent accounts in a single bucket and strands 200,000 fXRP, which is **3.77% of vault TVL**. The defect recurs for every account that passes through a rescue followed by an un-blocklisting, and the totals accumulate permanently.

**Collateral damage to every other staker.** The stranded assets remain physically in the vault but stay counted in `pendingWithdrawAssets` forever. Because `totalAssets()` is defined at `FirelightVault.sol:427` as:

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() - pendingWithdrawAssets;
}
```

the frozen amount is excluded from share value indefinitely. The proof of concept shows the vault holding 4,990,000 fXRP while reporting `totalAssets()` of 4,890,000 fXRP against `pendingWithdrawAssets` of 100,000 fXRP.

**Knock-on effect on underwriting capacity.** The permanently suppressed total propagates into `totalAssetsAt()` (`FirelightVault.sol:456`), which is the exposure snapshot consumed by `CoverOrderAllocator._computeAvailableCapacity`. The protocol therefore permanently under-reports the collateral it actually holds and under-sells cover capacity against it.

**Precision on permanence.** Bucket `p` remains slashable by `payout()` for capture period `p - 1` while `currentPeriod` is in `{p - 1, p}`, so during that window the assets are still usable as payable collateral. Any residue surviving that window is permanently stuck, which is the ordinary outcome since no incident is required for this defect to occur.

**Severity rationale.** The impact maps to permanent freezing of funds, with every recovery path closed and a loss that is a material proportion of the affected position. It is rated Medium rather than High or Critical because the per-instance loss is bounded to one account's post-rescue withdrawal within a single period, it produces no solvency break or capacity over-sell, and the triggering sequence requires a rescue followed by an un-blocklisting, which is a supported but uncommon operational flow. It is rated above Low because the freeze is permanent and verified, the proportion is well beyond dust scale, and the party who loses is an unprivileged user who does nothing wrong while every operator acts correctly.

## References

* `contracts/core/FirelightVault.sol:870`, the unclearable write inside `rescueWithdrawFromBlocklisted`
* `contracts/core/FirelightVault.sol:864-865`, the rescue preconditions that read the same flag
* `contracts/core/FirelightVault.sol:869`, the share zeroing that already prevents a double claim
* `contracts/core/FirelightVault.sol:788-812`, `claimWithdraw` and its unconditional flag check at 794
* `contracts/core/FirelightVault.sol:560-563`, `removeFromBlocklist`
* `contracts/core/FirelightVault.sol:994-1027`, `_requestWithdraw` and its `+=` bucket credit
* `contracts/core/FirelightVault.sol:427-429`, `totalAssets`
* `contracts/core/FirelightVault.sol:456-458`, `totalAssetsAt`
* `contracts/core/FirelightVaultStorage.sol:71`, the `isWithdrawClaimed` declaration
* `test/rescue.js`, the project's rescue test suite, which never exercises `removeFromBlocklist`
* Commit `a2def47`, "Fix rescuing withdrawal to a user that already claimed them", which addressed the `to` side only

## Proof of Concept

## Proof of Concept

The test exercises the real `FirelightVault` implementation behind a real `ERC1967Proxy`. No Firelight logic is mocked. The only substituted contract is the external ERC-20 collateral, a plain six-decimal OpenZeppelin `ERC20` standing in for FXRP. There is no `vm.store` anywhere: the frozen state is produced entirely by real calls. Trusted roles appear only in setup performing their documented duties, and the call that destroys the value is the affected user's own unprivileged `withdraw()`.

The suite contains three tests. The main test proves the freeze and closes every recovery path. The second proves repeatability across five independent accounts. The third is a negative control that runs the identical sequence against the same contract with line 870 deleted, proving the finding targets that single line.

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

# 3. Negative control: the in-scope contract with line 870 deleted and nothing else changed
mkdir -p test-foundry/fixed
sed -e '/isWithdrawClaimed\[periods\[i\]\]\[from\] = true;/d' \
    -e 's|import {FirelightVaultStorage} from "./FirelightVaultStorage.sol";|import {FirelightVaultStorage} from "../../contracts/core/FirelightVaultStorage.sol";|' \
    -e 's|import {Checkpoints} from "./lib/Checkpoints.sol";|import {Checkpoints} from "../../contracts/core/lib/Checkpoints.sol";|' \
    -e 's|^contract FirelightVault is|contract FirelightVaultFixed is|' \
    contracts/core/FirelightVault.sol > test-foundry/fixed/FirelightVaultFixed.sol
```

The generated control differs from the in-scope file by exactly one line:

```
869a870
>             isWithdrawClaimed[periods[i]][from] = true;
```

### Test file

Save as `repo/test-foundry/RescuePermanentLock.t.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {FirelightVaultFixed} from "./fixed/FirelightVaultFixed.sol";

/// The vault's underlying collateral. This is the EXTERNAL asset (FXRP stands in as a plain
/// 6-decimal ERC20), no Firelight logic is mocked anywhere in this file.
contract FXRP is ERC20 {
    constructor() ERC20("FAsset XRP", "fXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

/**
 * PoC for FirelightVault.sol:870
 *
 * `rescueWithdrawFromBlocklisted` writes `isWithdrawClaimed[period][from] = true` and NOTHING in
 * the in-scope tree ever clears it. Any balance credited to that account for that period bucket
 * afterwards is unclaimable forever.
 *
 * INVARIANT UNDER TEST (INV-1 SOLVENCY, withdrawal-claim form):
 *   whatever `withdrawalsOf(period, account)` reports as owed MUST be claimable by that account
 *   once the period has elapsed.
 *
 * CALLER MODEL, this is the finding's load-bearing joint, so it is made explicit:
 *   - Trusted roles appear ONLY in setup, performing their honest, documented duties:
 *       BLOCKLIST_ROLE flags an account, RESCUER_ROLE moves its pending withdrawal to custody,
 *       BLOCKLIST_ROLE clears the flag when the review exonerates the account.
 *     No role acts maliciously, and no role's harmful choice is required.
 *   - The step that destroys the value is the VICTIM'S OWN ordinary, unprivileged `withdraw()`.
 *   There is no profiting attacker here: this is the "a constraint fails to bind" shape, not the
 *   "attacker steals" shape. The constraint that fails is the double-claim guard, which
 *   over-constrains and permanently freezes funds.
 */
contract RescuePermanentLockPoC is Test {
    uint256 constant ONE = 1e6;              // 6-decimal asset
    uint48  constant PERIOD = 7 days;
    uint256 constant DEPOSIT_LIMIT = 50_000_000 * ONE;

    // realistic vault: ~5.05M fXRP TVL across three stakers
    uint256 constant STAKER1_DEP = 3_000_000 * ONE;
    uint256 constant STAKER2_DEP = 1_900_000 * ONE;
    uint256 constant VICTIM_DEP  =   150_000 * ONE;   // leaves a residual share balance so
                                                     // recovery-path 3 is testable on its merits
    uint256 constant FIRST_WD    =    60_000 * ONE;
    uint256 constant SECOND_WD   =    40_000 * ONE;   // <-- the amount that gets stranded

    FXRP asset;
    FirelightVault vault;

    address admin       = makeAddr("admin");
    address blocklister = makeAddr("blocklister");   // trusted, honest
    address rescuer     = makeAddr("rescuer");       // trusted, honest
    address custody     = makeAddr("custody");       // rescue destination
    address victim      = makeAddr("victim");        // UNPRIVILEGED
    address staker1     = makeAddr("staker1");
    address staker2     = makeAddr("staker2");

    function _initBlob() internal view returns (bytes memory) {
        return abi.encode(
            FirelightVault.InitParams({
                defaultAdmin: admin,
                limitUpdater: admin,
                blocklister: blocklister,
                pauser: admin,
                periodConfigurationUpdater: admin,
                rescuer: rescuer,
                depositLimit: DEPOSIT_LIMIT,
                periodConfigurationDuration: PERIOD
            })
        );
    }

    function setUp() public {
        vm.warp(1_800_000_000);
        asset = new FXRP();

        // REAL FirelightVault implementation behind a REAL ERC1967 proxy.
        FirelightVault impl = new FirelightVault();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(FirelightVault.initialize, (IERC20(address(asset)), "stfXRP", "stfXRP", _initBlob()))
        );
        vault = FirelightVault(address(proxy));

        _seed(staker1, STAKER1_DEP);
        _seed(staker2, STAKER2_DEP);
        _seed(victim,  VICTIM_DEP);
    }

    function _seed(address who, uint256 amt) internal {
        asset.mint(who, amt);
        vm.startPrank(who);
        asset.approve(address(vault), type(uint256).max);
        vault.deposit(amt, who);
        vm.stopPrank();
    }

    // ---------------------------------------------------------------- main PoC

    function test_PoC_rescue_permanently_strands_a_later_legitimate_withdrawal() public {
        // ============================= ACT 1: BASELINE =============================
        uint256 tvlBefore = vault.totalAssets();
        uint256 bucket = vault.currentPeriod() + 1;

        vm.prank(victim);
        vault.withdraw(FIRST_WD, victim, victim);

        uint256 owedFirst = vault.withdrawalsOf(bucket, victim);
        console.log("=== ACT 1  BASELINE ===");
        console.log("  vault TVL                 :", tvlBefore);
        console.log("  victim position           :", VICTIM_DEP);
        console.log("  bucket under test         :", bucket);
        console.log("  owed after 1st withdraw   :", owedFirst);

        assertEq(owedFirst, FIRST_WD, "baseline: first withdrawal is owed in full");
        assertFalse(vault.isWithdrawClaimed(bucket, victim), "baseline: not flagged claimed");

        // INVARIANT HOLDS BEFORE: a fresh account can claim what it is owed. Proven on staker1,
        // an untouched control account running the identical flow in the same bucket.
        vm.prank(staker1);
        vault.withdraw(FIRST_WD, staker1, staker1);
        assertEq(vault.withdrawalsOf(bucket, staker1), FIRST_WD, "baseline: control owed in full");

        // ========= ACT 2: HONEST OPERATOR SETUP, THEN UNPRIVILEGED WITHDRAWAL =========
        console.log("=== ACT 2  SEQUENCE ===");

        // (setup, honest) compliance flags the victim
        vm.prank(blocklister);
        vault.addToBlocklist(victim);
        console.log("  [operator, honest] blocklisted victim");

        // (setup, honest) RESCUER moves the pending withdrawal to custody, the function's documented purpose.
        // THIS is what sets the permanent flag.
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;
        vm.prank(rescuer);
        vault.rescueWithdrawFromBlocklisted(victim, custody, periods);
        console.log("  [operator, honest] rescued pending withdrawal to custody");
        assertTrue(vault.isWithdrawClaimed(bucket, victim), "rescue sets the flag");

        // (setup, honest) review exonerates the victim; the designation is reversed via the
        // protocol's own first-class function.
        vm.prank(blocklister);
        vault.removeFromBlocklist(victim);
        assertFalse(vault.isBlocklisted(victim), "victim is un-blocklisted");
        assertTrue(vault.isWithdrawClaimed(bucket, victim), "flag SURVIVES un-blocklisting");
        console.log("  [operator, honest] un-blocklisted victim; flag still set:", vault.isWithdrawClaimed(bucket, victim));

        // ---- EXPLOIT STEP: the victim, entirely unprivileged, makes an ordinary withdrawal ----
        vm.prank(victim);
        vault.withdraw(SECOND_WD, victim, victim);
        console.log("  [UNPRIVILEGED victim] withdraw(40,000) -> credits the SAME bucket");

        uint256 stranded = vault.withdrawalsOf(bucket, victim);
        assertEq(stranded, SECOND_WD, "the vault agrees the victim is owed this");

        // ============================== ACT 3: PROOF ===============================
        // move past the bucket so it is claimable
        vm.warp(block.timestamp + 2 * PERIOD);
        assertGt(vault.currentPeriod(), bucket, "bucket has elapsed and is claimable");

        console.log("=== ACT 3  PROOF ===");
        console.log("  vault reports owed to victim :", stranded);

        // (recovery path 1) the victim cannot claim
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSignature("AlreadyClaimedPeriod(uint256)", bucket));
        vault.claimWithdraw(bucket);
        console.log("  [recovery 1] claimWithdraw            -> AlreadyClaimedPeriod");

        // (recovery path 2) a second rescue cannot help: the rescue path is gated on the very
        // flag it set, so it locks its own escape hatch.
        vm.prank(blocklister);
        vault.addToBlocklist(victim);
        vm.prank(rescuer);
        vm.expectRevert(abi.encodeWithSignature("AlreadyClaimedPeriod(uint256)", bucket));
        vault.rescueWithdrawFromBlocklisted(victim, custody, periods);
        console.log("  [recovery 2] second rescue            -> AlreadyClaimedPeriod");

        // (recovery path 3) rescuing SHARES does not help either, the shares backing the
        // stranded amount were already burned by _requestWithdraw.
        vm.prank(rescuer);
        vault.rescueSharesFromBlocklisted(victim, custody);
        vm.prank(blocklister);
        vault.removeFromBlocklist(victim);
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSignature("AlreadyClaimedPeriod(uint256)", bucket));
        vault.claimWithdraw(bucket);
        console.log("  [recovery 3] rescueShares then claim  -> AlreadyClaimedPeriod");

        // (control) the untouched account claims the identical bucket successfully, isolating
        // the rescue as the sole cause.
        vm.prank(staker1);
        uint256 controlGot = vault.claimWithdraw(bucket);
        assertEq(controlGot, FIRST_WD, "control account claims the same bucket fine");
        console.log("  [control ] staker1 same bucket        -> claimed", controlGot);

        // ---------------------------- LOSS QUANTIFICATION ----------------------------
        uint256 vaultBal = asset.balanceOf(address(vault));
        uint256 pending = vault.pendingWithdrawAssets();

        console.log("--- LOSS ---");
        console.log("  stranded (absolute, 6dp)    :", stranded);
        console.log("  as %% of victim position    :", (stranded * 10000) / VICTIM_DEP, "bps");
        console.log("  as %% of vault TVL at start :", (stranded * 10000) / tvlBefore, "bps");
        console.log("  vault token balance         :", vaultBal);
        console.log("  pendingWithdrawAssets       :", pending);
        console.log("  totalAssets (suppressed)    :", vault.totalAssets());

        // THE ASSERTIONS THAT DEFINE THE BUG. A correct fix flips these.
        assertGt(stranded, 0, "vault owes the victim a non-zero amount");
        assertTrue(vault.isWithdrawClaimed(bucket, victim), "flag is still set, permanently");
        assertGe(pending, stranded, "the stranded assets remain counted as pending forever");
        assertGe(asset.balanceOf(address(vault)), stranded, "the assets are physically present");
        assertEq(asset.balanceOf(victim), 0, "victim received nothing");
    }

    // ---------------------------------------------------------------- repeatability

    function test_PoC_repeatable_across_independent_accounts() public {
        uint256 bucket = vault.currentPeriod() + 1;
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;

        uint256 cumulative;
        for (uint256 i; i < 5; ++i) {
            address v = makeAddr(string(abi.encodePacked("victim", vm.toString(i))));
            _seed(v, VICTIM_DEP);

            vm.prank(v);
            vault.withdraw(FIRST_WD, v, v);
            vm.prank(blocklister);
            vault.addToBlocklist(v);
            vm.prank(rescuer);
            vault.rescueWithdrawFromBlocklisted(v, custody, periods);
            vm.prank(blocklister);
            vault.removeFromBlocklist(v);
            vm.prank(v);
            vault.withdraw(SECOND_WD, v, v);

            cumulative += vault.withdrawalsOf(bucket, v);
        }

        vm.warp(block.timestamp + 2 * PERIOD);

        for (uint256 i; i < 5; ++i) {
            address v = makeAddr(string(abi.encodePacked("victim", vm.toString(i))));
            vm.prank(v);
            vm.expectRevert(abi.encodeWithSignature("AlreadyClaimedPeriod(uint256)", bucket));
            vault.claimWithdraw(bucket);
        }

        console.log("=== REPEATABILITY ===");
        console.log("  independent accounts stranded :", uint256(5));
        console.log("  cumulative stranded (6dp)     :", cumulative);
        console.log("  cumulative as %% of TVL       :", (cumulative * 10000) / vault.totalAssets(), "bps");
        assertEq(cumulative, 5 * SECOND_WD, "each account strands its full second withdrawal");
    }

    // ---------------------------------------------------------------- negative control

    /// The SAME sequence against `FirelightVaultFixed`, byte-identical to the in-scope file with
    /// exactly one line removed (`isWithdrawClaimed[periods[i]][from] = true;` at :870).
    /// This proves the PoC targets the bug's effect and FAILS once the bug is fixed.
    function test_NEGATIVE_CONTROL_fix_makes_the_claim_succeed() public {
        FirelightVaultFixed fImpl = new FirelightVaultFixed();
        ERC1967Proxy p = new ERC1967Proxy(
            address(fImpl),
            abi.encodeCall(FirelightVaultFixed.initialize, (IERC20(address(asset)), "stfXRP", "stfXRP", _initBlob()))
        );
        FirelightVaultFixed fv = FirelightVaultFixed(address(p));

        asset.mint(victim, VICTIM_DEP);
        vm.startPrank(victim);
        asset.approve(address(fv), type(uint256).max);
        fv.deposit(VICTIM_DEP, victim);
        vm.stopPrank();

        uint256 bucket = fv.currentPeriod() + 1;
        uint256[] memory periods = new uint256[](1);
        periods[0] = bucket;

        vm.prank(victim);
        fv.withdraw(FIRST_WD, victim, victim);
        vm.prank(blocklister);
        fv.addToBlocklist(victim);
        vm.prank(rescuer);
        fv.rescueWithdrawFromBlocklisted(victim, custody, periods);
        vm.prank(blocklister);
        fv.removeFromBlocklist(victim);
        vm.prank(victim);
        fv.withdraw(SECOND_WD, victim, victim);

        vm.warp(block.timestamp + 2 * PERIOD);

        // Under the fix, the double-claim guard still works (withdrawSharesOf was zeroed), but
        // the victim's NEW balance is claimable, as it must be.
        vm.prank(victim);
        uint256 got = fv.claimWithdraw(bucket);

        console.log("=== NEGATIVE CONTROL (one line deleted) ===");
        console.log("  victim claimed successfully :", got);
        assertEq(got, SECOND_WD, "with the fix, the victim recovers exactly the stranded amount");
        assertEq(asset.balanceOf(victim), SECOND_WD, "funds actually delivered");
    }
}
```

### Command

```bash
cd repo && forge test --match-path "test-foundry/RescuePermanentLock.t.sol" -vv
```

### Output

```
Ran 3 tests for test-foundry/RescuePermanentLock.t.sol:RescuePermanentLockPoC
[PASS] test_NEGATIVE_CONTROL_fix_makes_the_claim_succeed() (gas: 6042004)
Logs:
  === NEGATIVE CONTROL (one line deleted) ===
    victim claimed successfully : 40000000000

[PASS] test_PoC_repeatable_across_independent_accounts() (gas: 1663090)
Logs:
  === REPEATABILITY ===
    independent accounts stranded : 5
    cumulative stranded (6dp)     : 200000000000
    cumulative as % of TVL       : 377 bps

[PASS] test_PoC_rescue_permanently_strands_a_later_legitimate_withdrawal() (gas: 667124)
Logs:
  === ACT 1  BASELINE ===
    vault TVL                 : 5050000000000
    victim position           : 150000000000
    bucket under test         : 1
    owed after 1st withdraw   : 60000000000
  === ACT 2  SEQUENCE ===
    [operator, honest] blocklisted victim
    [operator, honest] rescued pending withdrawal to custody
    [operator, honest] un-blocklisted victim; flag still set: true
    [UNPRIVILEGED victim] withdraw(40,000) -> credits the SAME bucket
  === ACT 3  PROOF ===
    vault reports owed to victim : 40000000000
    [recovery 1] claimWithdraw            -> AlreadyClaimedPeriod
    [recovery 2] second rescue            -> AlreadyClaimedPeriod
    [recovery 3] rescueShares then claim  -> AlreadyClaimedPeriod
    [control ] staker1 same bucket        -> claimed 60000000000
  --- LOSS ---
    stranded (absolute, 6dp)    : 40000000000
    as % of victim position    : 2666 bps
    as % of vault TVL at start : 79 bps
    vault token balance         : 4990000000000
    pendingWithdrawAssets       : 100000000000
    totalAssets (suppressed)    : 4890000000000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 7.03ms (3.49ms CPU time)

Ran 1 test suite in 13.10ms (7.03ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Reading the output

`staker1` is an untouched control account that performs the identical withdrawal into the identical bucket without the rescue step, and claims `60000000000` successfully. That isolates the rescue as the sole cause of the failure.

The affected user is owed `40000000000` according to the vault's own `withdrawalsOf` accessor, and all three recovery attempts revert `AlreadyClaimedPeriod`. The vault holds `4990000000000` tokens while reporting `totalAssets()` of `4890000000000`.

The negative control claims exactly `40000000000` against the contract with line 870 removed. With that fix applied to the in-scope contract, the three `expectRevert(AlreadyClaimedPeriod)` assertions in the main test fail, which confirms the test targets the defect itself rather than incidental behaviour.
