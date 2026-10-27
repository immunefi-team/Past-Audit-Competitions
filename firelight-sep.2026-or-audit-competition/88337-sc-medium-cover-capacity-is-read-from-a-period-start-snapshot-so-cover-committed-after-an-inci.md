# 88337 sc medium cover capacity is read from a period start snapshot so cover committed after an incident payout is priced against collateral the payout already spent

**Submitted on Aug 12th 2026 at 19:35:59 UTC by @johanw for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88337
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator._computeAvailableCapacity` values staked collateral as `vault.totalAssetsAt(vault.currentPeriodStart())`, a checkpoint keyed to the start of the period. `FirelightVault.payout` writes its post-payout checkpoint at `Time.timestamp()`, which is always strictly later, so that lookup cannot observe the payout for the remainder of the period.

A cover allocation committed after an incident payout is therefore validated against collateral that has already left the vault. In the PoC a claim is approved and paid through the real `IncidentManager`, taking the vault from 1,000,000 USD to 100,000 USD. The allocator then makes a fresh commitment, with no prior commitment in that period, and the on-chain capacity check authorises 5,000,000 USD of cover. Live collateral supports 500,000 USD. The check is overstating by a factor of ten and it is enforcing to the wei while doing so. That cover is then sold and premium is collected on all 5,000,000 USD of it at settlement.

There is no attacker. Every role acts correctly and the protocol still reaches a state its own on-chain control was written to prevent.

## Vulnerability Details

**Contracts:**

* `contracts/core/CoverOrderAllocator.sol` - `_computeAvailableCapacity()` (L696-725, the pinned read at L711, the live first-loss-buffer read at L705), `_validateMatchingCapacity()` (L394-403), the per-protocol concentration cap (L500-512), `_replaceableCommitment()` (L425-435), `MAX_LEVERAGE_FACTOR` and its real-backing comment (L79-82)
* `contracts/core/FirelightVault.sol` - `payout()` (L892-966, the post-payout checkpoint at L952), `totalAssetsAt()` (L456-458)
* `contracts/core/IncidentManager.sol` - `_executePayout()` (L621-706, the vault call at L684)

All three are named in the competition's Assets in Scope table. Nothing in `contracts/legacy/` or `contracts/oracle/` is involved.

**Bug Class:** Stale historical checkpoint used as a live solvency input. Capacity is computed from a period-start snapshot that a mid-period payout can never move, so underwriting capacity is overstated and cover is sold beyond the configured capital adequacy floor.

**Severity:** Medium. CVSS 4.0 vector: `CVSS:4.0/AV:N/AC:L/AT:P/PR:H/UI:N/VC:N/VI:H/VA:N/SC:N/SI:H/SA:N` = 6.9, computed with the `cvss` library rather than estimated. `AT:P` because the state needs a payout to execute inside a period before that period's allocation is committed, a normal protocol event but a condition of the deployment. `PR:H` because reaching the state requires the assessment approver to approve a claim and the allocator to commit; both act correctly and neither is an attacker, but CVSS measures the privileges of the party performing the action. `VI:H` is the driver: the on-chain solvency control is violated through the exact check written to enforce it, at ten times the configured limit. `SI:H` for the subsequent impact on the cover buyer, a separate party. Selected impact is Protocol insolvency; severity is calibrated to Medium rather than that impact's usual band because this protocol is fractional by design at 5x, and the defect degrades the ratio to 50x for the remainder of one period rather than creating insolvency from a solvent state.

## Root Cause

The two legs of the same capacity sum are read differently. The first-loss buffer is live and re-floors on its own; the vault balance is a checkpoint lookup keyed to the period start:

```solidity
// contracts/core/CoverOrderAllocator.sol:704-716
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer),   // LIVE
    _readDecimals(address(config.firstLossBufferToken)),
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),             // PINNED
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`totalAssetsAt` resolves through `upperLookupRecent`, which returns the most recent checkpoint at or before the key. The payout writes its checkpoint here:

```solidity
// contracts/core/FirelightVault.sol:952
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

A payout during period Q necessarily has `Time.timestamp() > currentPeriodStart()` for Q, so `upperLookupRecent(currentPeriodStart())` cannot return it. This is not a race and there is no window to win: every call into `_computeAvailableCapacity` for the rest of the period returns the pre-payout figure.

The result feeds the bound that is supposed to enforce solvency:

```solidity
// contracts/core/CoverOrderAllocator.sol:394-403
function _validateMatchingCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period,
    uint256 matchingCapacity,
    uint256 totalAllocated
) private view {
    uint256 maxCapacity = _computeAvailableCapacity($, period);
    if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
    if (totalAllocated > matchingCapacity) revert TotalAllocationOverflow(totalAllocated, matchingCapacity);
}
```

A second control inherits the same figure once it is frozen into the commitment:

```solidity
// contracts/core/CoverOrderAllocator.sol:506-508
uint256 protocolConcentrationCap =
    (_getEffectiveProtocolConcentration(concHash, order.period) * commit.totalAvailableCapacity) /
        BPS_DENOMINATOR;
```

This is what the code itself claims to guarantee:

```solidity
// contracts/core/CoverOrderAllocator.sol:79-81
// Cap on `effectiveLeverage` relative to `minCAR`. Real backing behind sold cover is
// `minCAR / (effectiveLeverage * (1 + tolerance))`, so bounding the ratio floors it at
// `1 / (MAX_LEVERAGE_FACTOR * (1 + tolerance))` regardless of the configured `minCAR`.
```

A floor asserted in code and not enforced by it is the defect, independent of who calls the function.

**Sequence, all of it ordinary operation:**

1. Stakers hold 1,000,000 USD; the first-loss buffer holds 200,000 USD. Period 1 cover commits and settles honestly against the 6,000,000 USD of capacity that supports.
2. An incident captures period 1. The curator confirms it, records an assessed loss of 1,100,000 USD against the settled cover, and submits the assessment.
3. Period 2 opens. The approver approves. `_executePayout` draws 200,000 USD from the buffer and calls `vault.payout` for the remaining 900,000 USD. The vault holds 100,000 USD; the buffer is empty.
4. Still in period 2, the allocator makes a first-time commitment for that period. `_validateMatchingCapacity` computes capacity from the period-2-start snapshot, which never saw the payout, and authorises 5,000,000 USD. Live collateral supports 500,000 USD.
5. The order settles. 5,000,000 USD of cover is live against 100,000 USD of collateral, and premium is charged on all of it.

Step 3 is not exceptional. The payout window is the capture period or the one after it, and one allocation commitment exists per period, so a payout preceding a period's commitment is an expected operating state for a cover protocol.

## Proof of Concept

See the Proof of Concept field below for the full runnable Foundry test and output.

## Impact

The capital adequacy bound is the product. `minCAR` and `effectiveLeverage` fix a minimum ratio of real collateral to sold cover, and `_validateMatchingCapacity` is the on-chain control that enforces it. After a payout, that control authorises allocations it should reject, for the rest of the period.

Measured in the PoC:

* The protocol sold, and collected premium on, 5,000,000 USD of cover at a moment when it held 100,000 USD of collateral. Ten times the configured limit of underwriting exists on the book.
* Stakers carried 50x leverage on their capital where the configuration permits 5x.
* Because `_hasActiveIncident()` gates `deposit` and `mint` (`FirelightVault.sol:660`, `:696`), stakers cannot top the vault back up while the incident is open, so the gap cannot be closed by new deposits during the exposure window.

On the causal boundary, stated so it is not overread: the PoC also validates a 3,000,000 USD claim against that cover and shows it paying 100,000 USD. That 100,000 USD figure is what the vault holds and would be the same had capacity been floored correctly, because `addAssessmentLosses` caps an assessed loss at the order's `allocatedCoverAmount` and `payout` caps at live assets either way. The shortfall on that single claim is therefore not attributable to this defect and is not claimed as such. What this defect causes is that the cover existed to be sold at all: premium taken for underwriting capacity that provably did not exist, and a book carrying ten times the liability the configured floor permits, so that aggregate claims across all buyers in the period exceed what the collateral can serve.

Firelight's staker documentation at <https://docs.firelight.finance/for-stakers> states that "once coverage is live on mainnet, all staked capital in the vault backs coverage and is slashable". The test shows cover sold at ten times the collateral the configuration permits to back it.

## Limitations, stated up front

* The stale window is bounded to the remainder of the period in which the payout lands. At the next boundary the checkpoint is visible and capacity re-floors without intervention. Proven in `test_scope_staleWindowClosesAtTheNextPeriodBoundary`.
* The claim that the per-protocol concentration cap at `CoverOrderAllocator.sol:507` inherits the same stale figure is from reading the code, not from the PoC. The test sets `maxProtocolConcentrationBps` to 10,000, which disables that control.
* The bound is an upper bound, so an operator may always commit a smaller capacity voluntarily. What is proven is narrower and still the point: the contract is the only on-chain source of the figure, and it reports the stale one to whoever asks.

## Distinctness From Disclosed Issues

Known issue 3, "aggregate claims not capped on-chain", concerns claims exceeding cover sold, in `IncidentManager`, and the vault says so itself at `FirelightVault.sol:885`: "The vault applies only these per-call limits; ensuring aggregate payouts stay within underwritten exposure is the responsibility of the curator and assessor roles." This report is the opposite direction, cover exceeding collateral, in `CoverOrderAllocator`. That direction is not delegated to roles; the protocol built an on-chain cap for it and the evidence shows the cap live and enforcing to the wei while computing from an input a payout can never move.

The first-loss-buffer accepted risk, that a buffer funded at commitment and then drawn down leaves sold cover underbacked, is the nearest relative and this is strictly worse. For the buffer the read is live, so only already-committed cover goes stale. For the vault the read is pinned, so a brand-new commitment made after the drawdown is mispriced too. The PoC shows both legs side by side.

Macro M-2 and Firelight's L-2 response discuss a stale-pin pattern, but that is the off-chain matching engine's pin block, not this on-chain storage read.

Macro M-1 was unbounded `effectiveLeverage`, fixed at `CoverOrderAllocator.sol:665` with `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR`. That bounds the multiplier; this is about the collateral input the multiplier applies to. The PoC runs with the M-1 fix in force at maximum legal leverage and the bound is still breached.

I reviewed the full Macro finding list at <https://library-pr-440.onrender.com/library/audits/firelight-1> (2 Medium, 4 Low, 1 Gas, 11 Informational, published 2026-07-29). None addresses period-start snapshotting or capacity-input staleness. The strings `totalAssetsAt` and `currentPeriodStart` do not appear in it.

## Recommended Fix

Keep the period-start snapshot, since the off-chain matcher needs a deterministic capacity figure and that is why `matchingCapacity` is a commit parameter. Add a live floor so the pinned value can never exceed present reality:

```solidity
// contracts/core/CoverOrderAllocator.sol - _computeAvailableCapacity()
uint256 liveCapacity = _capacityFromCollateral($, period, $.vault.totalAssets());
totalAvailableCapacity = Math.min(snapshotCapacity, liveCapacity);
```

`min` preserves determinism in the normal case, where the two agree, and floors capacity the moment a payout removes collateral. This is sufficient for the sequence proven here, where the commitment is made after the payout.

Two further changes close the case where the commitment already exists when the payout lands:

1. Bound `commit.totalAvailableCapacity` by live capacity inside `_settleCoverOrder` rather than trusting the value frozen at commit time, and use the same bounded value for the per-protocol concentration cap at L507.
2. Give payouts a way to invalidate a live commitment. `_replaceableCommitment` rejects both `recommitAllocation` and `cancelCommitAllocation` with `SettlementsAlreadyStarted` once any cover has settled, so a partially settled period cannot be corrected at all. Either have `IncidentManager` re-floor the current period's commitment when `vault.payout` moves collateral, or permit a downward-only recommit after settlement has begun.

## Supporting Materials

* `poc-test/CapacityStaleAfterPayout.t.sol` - full runnable Foundry PoC, 5 tests: the defect, the realized-loss measurement, a negative control, and two scope tests that state the limits of the finding. Source inlined in the Proof of Concept field below.
* `foundry.toml` - the build configuration used, inlined below. The repository ships no `foundry.toml` at this commit.

## Proof of Concept

## Proof of Concept

Full runnable test at `poc-test/CapacityStaleAfterPayout.t.sol`, source inlined at the end of this field. It is self-contained: the vault, allocator, cover NFT, IncidentManager and the two mocks are all deployed inside the test, so it runs offline with no fork, no RPC endpoint and no API key, and it touches no deployed contract and no real funds.

Nothing is mocked in the primary test. The payout is driven through the real IncidentManager lifecycle, `createIncident` then `confirmIncident` then `addAssessmentLosses` then `submitCurrentAssessment` then `approveCurrentAssessment`, which reaches `_executePayout` and calls `vault.payout` exactly as deployed. The two mocks used are the repository's own `contracts/test/MockERC20.sol` and `contracts/test/MockAggregatorV3.sol`, standing in for the collateral token and the price feed only.

The first-loss buffer is funded with 200,000 and drawn by the real payout, deliberately. An empty buffer would force 100 percent of the capacity sum onto the pinned vault leg and overstate the defect; funding it shows the live buffer leg re-flooring correctly next to the pinned vault leg that does not.

### Setup

Requires Foundry and git only.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git firelight
cd firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62

mkdir -p foundry-lib poc-test
git clone --depth 1 --branch v5.2.0 https://github.com/OpenZeppelin/openzeppelin-contracts.git foundry-lib/openzeppelin-contracts
git clone --depth 1 --branch v5.4.0 https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable.git foundry-lib/openzeppelin-contracts-upgradeable
git clone --depth 1 --branch v1.9.6 https://github.com/foundry-rs/forge-std.git foundry-lib/forge-std
```

Save the `foundry.toml` below to the repository root, and the test below to `poc-test/CapacityStaleAfterPayout.t.sol`.

The OpenZeppelin versions are the pair `package.json` pins: `"@openzeppelin/contracts": "5.2.0"` under `overrides`, and `"@openzeppelin/contracts-upgradeable": "~5.4.0"`. `foundry.toml` skips `contracts/fasset/**` and `contracts/oracle/**`, the only sources importing the `flare-smart-contracts` npm packages. Verified from a cold cache with exactly these pins.

### Run

```bash
forge test --match-path 'poc-test/CapacityStaleAfterPayout*' -vv
```

**Scenario:** a 1,000,000 USD staked vault with a 200,000 USD first-loss buffer, `minCAR` 12000 (the contract's own `MIN_CAR_BPS`), `effectiveLeverage` 60000 (the maximum `_setCapacityConfig` accepts, `MAX_LEVERAGE_FACTOR * minCAR`), `divergenceToleranceBps` 0, period duration 7 days. Tolerance 0 makes the result harder to reach, not easier, since a nonzero tolerance widens capacity further. Leverage cancels out of the backing ratio, so the tenfold overstatement is set by the payout fraction rather than by the leverage choice.

### Result (actual forge output)

```
Ran 5 tests for poc-test/CapacityStaleAfterPayout.t.sol:CapacityStaleAfterPayout
[PASS] test_capacityIgnoresAPayoutForTheRestOfThePeriod() (gas: 2329179)
Logs:
  collateral live at commit time (USD 18d): 1e23
  cover the stale read authorised   (18d): 5e24
  cover live collateral supports    (18d): 5e23
  cover actually settled            (18d): 5e24
  overstatement factor                   : 10x
[PASS] test_control_withoutAPayoutTheBoundIsExact() (gas: 623876)
[PASS] test_scope_aSingleClaimMayDrainTheVaultEntirely() (gas: 1519411)
Logs:
  vault fully drained, period-start snapshot still reports (6d): 1e12
[PASS] test_scope_staleWindowClosesAtTheNextPeriodBoundary() (gas: 1547545)
Logs:
  stale window is bounded to the remainder of the payout period
[PASS] test_underBackedCoverPaysAFractionOfAValidatedClaim() (gas: 2842793)
Logs:
  validated claim   (USD 18d): 3e24
  actually paid out (USD 18d): 1e23
  shortfall         (USD 18d): 2.9e24
  recovery              (bps): 333
Suite result: ok. 5 passed; 0 failed; 0 skipped
```

### What each test establishes

`test_capacityIgnoresAPayoutForTheRestOfThePeriod` is the defect. The commitment it makes is a first-time commitment for that period, made after the payout, so the pinned read is the only thing authorising it. It also asserts that one wei above the pre-payout figure still reverts `MatchingCapacityOverflow(5000000000000000000000001, 5000000000000000000000000)`, matching the exact revert arguments. That rules out the reading that the check is disabled and shows it live, exact, and computing from a stale input.

`test_control_withoutAPayoutTheBoundIsExact` is the negative control. With no payout, committing 6,000,000 USD succeeds and one wei more reverts `MatchingCapacityOverflow(6000000000000000000000001, 6000000000000000000000000)`. The check itself works; only its input is wrong.

`test_underBackedCoverPaysAFractionOfAValidatedClaim` measures what happens when a claim lands against the cover the stale read authorised.

`test_scope_aSingleClaimMayDrainTheVaultEntirely` answers whether the drawdown size is an artificial parameter. It is not: no admin-settable per-call payout limit exists, `payout` is bounded only by `totalAssetsAt(capturePeriodStart)` and live payable assets, so one approved claim can take the vault to zero while the period-start snapshot still reports the full pre-payout figure.

`test_scope_staleWindowClosesAtTheNextPeriodBoundary` states the limit of the defect rather than leaving it to be found: the window closes at the next period boundary, where the checkpoint becomes visible and capacity re-floors without intervention.

A first run compiles the whole tree, so the compile line differs from a cached run. Gas figures may vary slightly by Foundry version; the asserted values do not.

### Build configuration, foundry.toml

```toml
[profile.default]
src = "contracts"
out = "foundry-out"
libs = ["foundry-lib"]
test = "poc-test"
cache_path = "foundry-cache"
solc = "0.8.28"
optimizer = true
optimizer_runs = 10000
via_ir = true
evm_version = "cancun"
skip = ["contracts/fasset/**", "contracts/oracle/**"]
remappings = [
  "@openzeppelin/contracts/=foundry-lib/openzeppelin-contracts/contracts/",
  "@openzeppelin/contracts-upgradeable/=foundry-lib/openzeppelin-contracts-upgradeable/contracts/",
  "forge-std/=foundry-lib/forge-std/src/",
]
```

### Full test file, poc-test/CapacityStaleAfterPayout.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {CoverOrderAllocator} from "../contracts/core/CoverOrderAllocator.sol";
import {ICoverOrderAllocator} from "../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IncidentManager} from "../contracts/core/IncidentManager.sol";
import {IIncidentManager} from "../contracts/core/interfaces/IIncidentManager.sol";
import {CoverNFT} from "../contracts/core/CoverNFT.sol";
import {IAggregatorV3} from "../contracts/core/interfaces/IAggregatorV3.sol";

import {MockERC20} from "../contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "../contracts/test/MockAggregatorV3.sol";

/// @notice `CoverOrderAllocator._computeAvailableCapacity` values staked collateral at
///         `vault.totalAssetsAt(vault.currentPeriodStart())`, a checkpoint keyed to the period start.
///         `FirelightVault.payout` writes its post-payout checkpoint at `Time.timestamp()`, which is
///         always strictly later, so for the remainder of that period the capacity formula cannot see
///         the payout. New cover is then committed and settled against collateral already paid out.
///
///         The first-loss-buffer leg of the same sum is read LIVE
///         (`config.firstLossBufferToken.balanceOf(...)`, line 705) and does re-floor correctly. The
///         buffer here is funded and drained so that asymmetry is visible rather than hidden.
///
///         Every step runs through the real contracts, including the full IncidentManager approval
///         path. Nothing is mocked in the primary test.
contract CapacityStaleAfterPayout is Test {
    address admin = makeAddr("admin");
    address curator = makeAddr("curator");
    address allocatorBot = makeAddr("allocatorBot");
    address configAdmin = makeAddr("configAdmin");
    address approver = makeAddr("approver");
    address premiumCollector = makeAddr("premiumCollector");
    address flbWallet = makeAddr("flbWallet");
    address lp = makeAddr("lp");
    address coverBuyer = makeAddr("coverBuyer");
    address payoutReceiver = makeAddr("payoutReceiver");

    MockERC20 asset;
    MockERC20 flbToken;
    MockAggregatorV3 oracle;
    FirelightVault vault;
    CoverOrderAllocator allocator;
    IncidentManager incidentManager;
    CoverNFT coverNFT;

    uint48 constant PERIOD = 7 days;
    uint256 constant MIN_CAR = 12_000; // 1.2x, the contract's MIN_CAR_BPS
    uint256 constant EFF_LEVERAGE = 60_000; // 5 * minCAR, the maximum _setCapacityConfig accepts
    uint16 constant TOLERANCE_BPS = 0; // strict; a nonzero tolerance only widens capacity further
    uint48 constant GRACE = 1 hours;

    uint256 constant LP_DEPOSIT = 1_000_000e6; // $1,000,000 of staked collateral
    uint256 constant FLB_FUNDING = 200_000e6; // $200,000 first-loss buffer, funded and approved
    uint256 constant ASSESSED_LOSS = 1_100_000e18; // $1,100,000 validated claim, canonical 18 decimals

    bytes32 marketId;
    uint48 t0;

    function setUp() public {
        vm.warp(1_800_000_000);
        t0 = uint48(block.timestamp);

        asset = new MockERC20("USD Stable", "USD", 6);
        flbToken = new MockERC20("First Loss Buffer", "FLB", 6);
        oracle = new MockAggregatorV3(8, 1e8); // $1.00 on an 8-decimal feed

        FirelightVault vaultImpl = new FirelightVault();
        IFirelightVault.InitParams memory vp = IFirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 1_000_000_000e6,
            periodConfigurationDuration: PERIOD
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(vaultImpl),
                    abi.encodeCall(
                        FirelightVault.initialize, (IERC20(address(asset)), "Firelight", "FLT", abi.encode(vp))
                    )
                )
            )
        );

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeCall(CoverNFT.initialize, ("Cover", "CVR", "", admin, admin, admin, admin))
                )
            )
        );

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(asset);

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1,
            protocol: "aave-v3",
            maxProtocolConcentrationBps: 10_000
        });

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "aave-v3", market: bytes32("usdc-pool")});
        marketId = keccak256(abi.encode(uint64(1), string("aave-v3"), bytes32("usdc-pool")));

        CoverOrderAllocator allocImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(allocImpl),
                    abi.encodeCall(
                        CoverOrderAllocator.initialize,
                        (
                            ICoverOrderAllocator.InitParams({
                                vault: IFirelightVault(address(vault)),
                                premiumCollector: premiumCollector,
                                coverNFT: coverNFT,
                                priceFeedAdapter: IAggregatorV3(address(oracle)),
                                maxPriceAge: 1 days,
                                premiumTokens: premiumTokens,
                                admin: admin,
                                adminRole: admin,
                                curatorRole: curator,
                                allocatorRole: allocatorBot,
                                configAdminRole: configAdmin,
                                initialProtocolConcentrations: concs,
                                newMarkets: markets,
                                capacityConfig: ICoverOrderAllocator.CapacityConfig({
                                    minCAR: MIN_CAR,
                                    firstLossBufferToken: IERC20(address(flbToken)),
                                    firstLossBuffer: flbWallet,
                                    effectiveLeverage: EFF_LEVERAGE,
                                    minOrderMarketCoverAmount: 1e18,
                                    divergenceToleranceBps: TOLERANCE_BPS
                                })
                            })
                        )
                    )
                )
            )
        );

        IncidentManager imImpl = new IncidentManager();
        incidentManager = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(imImpl),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            admin,
                            curator,
                            approver,
                            approver,
                            approver,
                            configAdmin,
                            admin,
                            admin,
                            payoutReceiver,
                            ICoverOrderAllocator(address(allocator)),
                            IAggregatorV3(address(oracle)),
                            1 days
                        )
                    )
                )
            )
        );

        vm.startPrank(admin);
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));
        // The IncidentManager is the real payout caller, exactly as deployed.
        vault.grantRole(vault.PAYOUT_ROLE(), address(incidentManager));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidentManager));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(payoutReceiver);
        vm.stopPrank();

        vm.prank(configAdmin);
        allocator.setSettlementGracePeriod(GRACE);

        // Fund and approve the first-loss buffer, so the payout draws from it first and its
        // live-read leg of the capacity formula genuinely re-floors.
        flbToken.mint(flbWallet, FLB_FUNDING);
        vm.prank(flbWallet);
        flbToken.approve(address(incidentManager), type(uint256).max);

        asset.mint(lp, LP_DEPOSIT);
        vm.startPrank(lp);
        asset.approve(address(vault), LP_DEPOSIT);
        vault.deposit(LP_DEPOSIT, lp);
        vm.stopPrank();

        asset.mint(coverBuyer, 1_000_000e6);
        vm.prank(coverBuyer);
        asset.approve(address(allocator), type(uint256).max);
    }

    function _freshOracle() internal {
        oracle.setAnswer(1e8);
    }

    function _createOrder(uint256 coverAmount) internal returns (uint256 orderId) {
        ICoverOrderAllocator.MarketAllocationInput[] memory mai =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mai[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId,
            coverRateAnnual: 200,
            coverAmount: coverAmount
        });
        vm.prank(curator);
        orderId = allocator.createCoverOrder(
            coverBuyer, coverBuyer, "flare:beneficiary", address(asset), mai, ICoverOrderAllocator.CoverOrderType.NEW
        );
    }

    function _commitAndSettle(uint256 period, uint256 orderId, uint256 cover, uint256 capacity) internal {
        ICoverOrderAllocator.MarketCoverAllocation[] memory mca =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mca[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: cover});
        bytes32 root = keccak256(bytes.concat(keccak256(abi.encode(orderId, mca))));

        vm.prank(allocatorBot);
        allocator.commitAllocation(period, root, cover, capacity);

        vm.warp(block.timestamp + GRACE + 1);
        _freshOracle();

        bytes32[] memory proof = new bytes32[](0);
        vm.prank(allocatorBot);
        allocator.settleCoverOrder(orderId, mca, proof);
    }

    /// @notice The full defect, driven end to end through the real IncidentManager approval path.
    function test_capacityIgnoresAPayoutForTheRestOfThePeriod() public {
        // ---- period 0: the curator books cover for period 1 ----
        uint256 order1 = _createOrder(2_000_000e18);

        // ---- period 1: that cover commits and settles against honest collateral ----
        vm.warp(t0 + PERIOD + 1);
        _freshOracle();
        assertEq(vault.currentPeriod(), 1);
        // $1,000,000 staked + $200,000 buffer, at 5x, is $6,000,000 of capacity.
        _commitAndSettle(1, order1, 2_000_000e18, 6_000_000e18);

        // Book the next period's cover, and open an incident against period 1.
        uint256 order2 = _createOrder(5_000_000e18);

        uint48 captureTs = uint48(block.timestamp);
        vm.startPrank(curator);
        uint256 incidentId = incidentManager.createIncident(captureTs, "Aave v3 pool exploit", bytes32("INC-1"));
        incidentManager.confirmIncident(incidentId, "ipfs://report");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({
            coverTokenId: order1,
            marketId: marketId,
            amount: ASSESSED_LOSS
        });
        incidentManager.addAssessmentLosses(incidentId, losses);
        incidentManager.submitCurrentAssessment(incidentId);
        vm.stopPrank();

        // ---- period 2 ----
        vm.warp(t0 + 2 * PERIOD + 1);
        _freshOracle();
        assertEq(vault.currentPeriod(), 2);

        uint48 period2Start = vault.currentPeriodStart();
        assertEq(vault.totalAssets(), LP_DEPOSIT, "vault whole at the period boundary");
        assertEq(vault.totalAssetsAt(period2Start), LP_DEPOSIT, "and the period-start checkpoint agrees");

        // The claim is approved. This is the real payout path: IncidentManager draws the buffer
        // first, then calls vault.payout for the remainder.
        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incidentId);

        assertEq(flbToken.balanceOf(flbWallet), 0, "buffer fully drawn");
        assertEq(vault.totalAssets(), 100_000e6, "vault down to $100,000");

        // ---- the defect ----
        // The buffer leg of the capacity formula is read live and correctly reports zero.
        // The vault leg is read from the period-start checkpoint and still reports the old figure.
        assertEq(vault.totalAssetsAt(period2Start), LP_DEPOSIT, "period-start snapshot never saw the payout");

        // Live collateral is now $100,000, which supports $500,000 of cover at the configured 5x.
        // The stale read still authorises $5,000,000, and the commit lands AFTER the payout.
        ICoverOrderAllocator.MarketCoverAllocation[] memory mca =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mca[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: 5_000_000e18});
        bytes32 root = keccak256(bytes.concat(keccak256(abi.encode(order2, mca))));

        vm.prank(allocatorBot);
        allocator.commitAllocation(2, root, 5_000_000e18, 5_000_000e18);

        // The bound is live and exact, it is simply computing from the stale figure: one wei more
        // than the pre-payout capacity is still rejected, against the pre-payout number.
        vm.prank(configAdmin);
        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector, 5_000_000e18 + 1, 5_000_000e18
            )
        );
        allocator.recommitAllocation(2, root, 5_000_000e18 + 1, 5_000_000e18 + 1);

        vm.warp(block.timestamp + GRACE + 1);
        _freshOracle();
        bytes32[] memory proof = new bytes32[](0);
        vm.prank(allocatorBot);
        allocator.settleCoverOrder(order2, mca, proof);

        uint256 settledCover = allocator.getAllocationCommitment(2).totalSettledCover;
        uint256 liveCollateral = vault.totalAssets() * 1e12 + flbToken.balanceOf(flbWallet) * 1e12;
        uint256 authorisedCover = 5_000_000e18;
        uint256 supportableCover = (liveCollateral * EFF_LEVERAGE) / MIN_CAR;

        console2.log("collateral live at commit time (USD 18d): %e", liveCollateral);
        console2.log("cover the stale read authorised   (18d): %e", authorisedCover);
        console2.log("cover live collateral supports    (18d): %e", supportableCover);
        console2.log("cover actually settled            (18d): %e", settledCover);
        console2.log("overstatement factor                   : %sx", authorisedCover / supportableCover);

        assertEq(supportableCover, 500_000e18, "live collateral supports $500,000 of cover");
        assertEq(settledCover, 5_000_000e18, "$5,000,000 settled anyway");
        assertEq(authorisedCover / supportableCover, 10, "capacity overstated tenfold");
    }

    /// @notice The realized loss. A validated claim lands against the cover that the stale read
    ///         authorised, and the cover buyer is paid a fraction of it, because `payout` is capped
    ///         by collateral that was already spent on the earlier incident.
    function test_underBackedCoverPaysAFractionOfAValidatedClaim() public {
        uint256 order1 = _createOrder(2_000_000e18);

        vm.warp(t0 + PERIOD + 1);
        _freshOracle();
        _commitAndSettle(1, order1, 2_000_000e18, 6_000_000e18);
        uint256 order2 = _createOrder(5_000_000e18);

        uint48 capture1 = t0 + PERIOD + 1 hours;
        assertEq(vault.periodAtTimestamp(capture1), 1, "first incident captures period 1");
        vm.startPrank(curator);
        uint256 incident1 = incidentManager.createIncident(capture1, "Incident one", bytes32("INC-1"));
        incidentManager.confirmIncident(incident1, "ipfs://one");
        IIncidentManager.AssessmentLoss[] memory l1 = new IIncidentManager.AssessmentLoss[](1);
        l1[0] = IIncidentManager.AssessmentLoss({coverTokenId: order1, marketId: marketId, amount: ASSESSED_LOSS});
        incidentManager.addAssessmentLosses(incident1, l1);
        incidentManager.submitCurrentAssessment(incident1);
        vm.stopPrank();

        // Period 2: the first claim is paid, then the stale read authorises $5,000,000 of new cover.
        vm.warp(t0 + 2 * PERIOD + 1);
        _freshOracle();
        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incident1);
        assertEq(vault.totalAssets(), 100_000e6, "collateral spent on the first claim");

        _commitAndSettle(2, order2, 5_000_000e18, 5_000_000e18);
        assertEq(allocator.getAllocationCommitment(2).totalSettledCover, 5_000_000e18, "under-backed cover is live");

        // A second, entirely legitimate claim lands against that cover, in the same period.
        uint256 secondClaim = 3_000_000e18; // $3,000,000, well inside the $5,000,000 of cover sold
        uint48 capture2 = t0 + 2 * PERIOD + 1 hours;
        assertEq(vault.periodAtTimestamp(capture2), 2, "second incident captures period 2");
        assertLe(capture2, uint48(block.timestamp), "capture timestamp is in the past");
        vm.startPrank(curator);
        uint256 incident2 = incidentManager.createIncident(capture2, "Incident two", bytes32("INC-2"));
        incidentManager.confirmIncident(incident2, "ipfs://two");
        IIncidentManager.AssessmentLoss[] memory l2 = new IIncidentManager.AssessmentLoss[](1);
        l2[0] = IIncidentManager.AssessmentLoss({coverTokenId: order2, marketId: marketId, amount: secondClaim});
        incidentManager.addAssessmentLosses(incident2, l2);
        incidentManager.submitCurrentAssessment(incident2);
        vm.stopPrank();

        uint256 receiverBefore = asset.balanceOf(payoutReceiver);
        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incident2);
        uint256 actuallyPaid = (asset.balanceOf(payoutReceiver) - receiverBefore) * 1e12; // to 18 decimals

        uint256 shortfall = secondClaim - actuallyPaid;
        uint256 recoveryBps = (actuallyPaid * 10_000) / secondClaim;

        console2.log("validated claim   (USD 18d): %e", secondClaim);
        console2.log("actually paid out (USD 18d): %e", actuallyPaid);
        console2.log("shortfall         (USD 18d): %e", shortfall);
        console2.log("recovery              (bps): %s", recoveryBps);

        assertEq(actuallyPaid, 100_000e18, "the vault pays only what is left, $100,000");
        assertEq(shortfall, 2_900_000e18, "$2,900,000 of validated claim goes unpaid");
        assertEq(recoveryBps, 333, "cover buyer recovers 3.33 percent of a claim inside their cover");
    }

    /// @notice Scope of the drawdown. No admin-settable per-call payout limit exists: `payout`
    ///         is bounded only by `totalAssetsAt(capturePeriodStart)` and live payable assets,
    ///         so a single approved claim may legitimately remove the entire vault. The size of
    ///         the gap is therefore not an artificial parameter choice.
    function test_scope_aSingleClaimMayDrainTheVaultEntirely() public {
        uint256 order1 = _createOrder(2_000_000e18);
        vm.warp(t0 + PERIOD + 1);
        _freshOracle();
        _commitAndSettle(1, order1, 2_000_000e18, 6_000_000e18);

        uint48 captureTs = uint48(block.timestamp);
        vm.startPrank(curator);
        uint256 incidentId = incidentManager.createIncident(captureTs, "Total loss", bytes32("INC-MAX"));
        incidentManager.confirmIncident(incidentId, "ipfs://max");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        // The full allocated cover of the order, the largest loss the assessment path accepts.
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: order1, marketId: marketId, amount: 2_000_000e18});
        incidentManager.addAssessmentLosses(incidentId, losses);
        incidentManager.submitCurrentAssessment(incidentId);
        vm.stopPrank();

        vm.warp(t0 + 2 * PERIOD + 1);
        _freshOracle();
        uint48 period2Start = vault.currentPeriodStart();

        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incidentId);

        // $200,000 came from the buffer, the remaining $1,800,000 was requested from the vault and
        // capped at everything the vault held.
        assertEq(vault.totalAssets(), 0, "a single approved claim emptied the vault");
        assertEq(vault.totalAssetsAt(period2Start), LP_DEPOSIT, "capacity still reads the full pre-payout figure");
        console2.log("vault fully drained, period-start snapshot still reports (6d): %e", vault.totalAssetsAt(period2Start));
    }

    /// @notice Control, no payout: the same bound is exact, so the check itself works.
    function test_control_withoutAPayoutTheBoundIsExact() public {
        uint256 order1 = _createOrder(2_000_000e18);
        vm.warp(t0 + PERIOD + 1);
        _freshOracle();

        ICoverOrderAllocator.MarketCoverAllocation[] memory mca =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mca[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: 2_000_000e18});
        bytes32 root = keccak256(bytes.concat(keccak256(abi.encode(order1, mca))));

        // $1,200,000 of collateral at 5x is exactly $6,000,000, and one wei more is refused.
        vm.prank(allocatorBot);
        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector, 6_000_000e18 + 1, 6_000_000e18
            )
        );
        allocator.commitAllocation(1, root, 2_000_000e18, 6_000_000e18 + 1);

        vm.prank(allocatorBot);
        allocator.commitAllocation(1, root, 2_000_000e18, 6_000_000e18);
        assertEq(allocator.getAllocationCommitment(1).totalAvailableCapacity, 6_000_000e18);
    }

    /// @notice Scope of the defect, stated up front: the stale window is bounded to the remainder
    ///         of the period in which the payout lands. At the next boundary the checkpoint is
    ///         visible again and capacity re-floors without intervention.
    function test_scope_staleWindowClosesAtTheNextPeriodBoundary() public {
        uint256 order1 = _createOrder(2_000_000e18);
        vm.warp(t0 + PERIOD + 1);
        _freshOracle();
        _commitAndSettle(1, order1, 2_000_000e18, 6_000_000e18);

        uint48 captureTs = uint48(block.timestamp);
        vm.startPrank(curator);
        uint256 incidentId = incidentManager.createIncident(captureTs, "Aave v3 pool exploit", bytes32("INC-1"));
        incidentManager.confirmIncident(incidentId, "ipfs://report");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: order1, marketId: marketId, amount: ASSESSED_LOSS});
        incidentManager.addAssessmentLosses(incidentId, losses);
        incidentManager.submitCurrentAssessment(incidentId);
        vm.stopPrank();

        vm.warp(t0 + 2 * PERIOD + 1);
        _freshOracle();
        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incidentId);

        assertEq(vault.totalAssetsAt(vault.currentPeriodStart()), LP_DEPOSIT, "period 2: stale");

        vm.warp(t0 + 3 * PERIOD + 1);
        _freshOracle();
        assertEq(vault.totalAssetsAt(vault.currentPeriodStart()), 100_000e6, "period 3: re-floored");
        console2.log("stale window is bounded to the remainder of the payout period");
    }
}
```
