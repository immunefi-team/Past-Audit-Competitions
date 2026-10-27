# 88925 sc high c 2 coverorderallocator contract values collateral at a period start snapshot that a mid period payout never updates so cover is sold paid for and receipted against a vault&#x20;

**Submitted on Aug 15th 2026 at 18:03:40 UTC by @web3pavlou for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88925
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

**Severity selected:** Critical — *Protocol insolvency*\
**Assets:** `contracts/core/CoverOrderAllocator.sol`, `contracts/core/FirelightVault.sol`, `contracts/core/lib/Checkpoints.sol`\
**Requires an attacker:** No. Reached by a routine operator sequence.

## Brief/Intro

`_computeAvailableCapacity` (`CoverOrderAllocator.sol:696`) values staked collateral at `$.vault.totalAssetsAt($.vault.currentPeriodStart())` (`:711`). `payout` writes its post-slash checkpoint at `Time.timestamp()` (`FirelightVault.sol:952`). Because `upperLookupRecent` selects the most recent checkpoint with key **≤** the search key, any payout executing after a period's first block is invisible to that period's capacity calculation.

The vault can therefore be drained to zero while the allocator continues to write cover against the pre-payout figure. The PoC below carries this all the way through to the sale: the vault reports `totalAssets() == 0`, and in the same period the allocator commits $2,000,000 of capacity, settles $100,000 of cover, collects 95.89 USD\* of premium from the buyer, and mints the CoverNFT receipt. The obligation exists, the money changed hands, and there is nothing behind it.

## Scope: this is not a privileged-actor report

`ALLOCATOR_ROLE` performs an ordinary `commitAllocation` and an ordinary `settleCoverOrder`, both within its granted permissions. The program's exclusion applies *"unless the report demonstrates that the role can exceed its intended permissions or **bypass an intended protocol constraint**."*

The intended protocol constraint here is explicit in the code: `_validateMatchingCapacity` (`:394`) exists to bound committed cover by available collateral against `minCAR`. This report demonstrates that the constraint is bypassed — not by a bad parameter, and not by the role exceeding its permissions, but because the constraint's **input is stale**. The role cannot detect the staleness and cannot avoid it. Control 1 confirms the bound itself works correctly once the input is current, which isolates the bypass to the input rather than to operator behaviour.

## Vulnerability Details

```solidity
// CoverOrderAllocator.sol:709-714
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

```solidity
// FirelightVault.sol:952 — inside payout()
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

`Checkpoints.upperLookupRecent(key)` returns the value of the latest checkpoint whose key is **≤** `key`. A payout at `periodStart + Δ` for any `Δ > 0` writes its checkpoint at a key strictly greater than `periodStart`, so `totalAssetsAt(periodStart)` walks past it and returns the pre-payout figure.

**Boundary behaviour, proved rather than asserted.** A payout landing in a period's *first block* writes its checkpoint at exactly `periodStart` and **is** seen, because the lookup is inclusive. Every later payout is not. Control 2 in the PoC demonstrates this directly: it executes the payout on the boundary, shows `totalAssetsAt(periodStart)` correctly returning `0`, and shows the same commit being correctly rejected. This matters because a reviewer reproducing the issue at the period boundary would otherwise see the mechanism behave correctly and conclude there is no bug. The finding case advances one hour into the period. Given the documented claims pipeline (48h report → 24h assessment → 72h validation), no genuine payout lands in a period's opening block.

The overstated figure flows straight into the capacity bound that `_validateMatchingCapacity` (`:394`) enforces on `commitAllocation` (`:332`), amplified by `effectiveLeverage / minCAR` and widened by `divergenceToleranceBps`. With `MAX_LEVERAGE_FACTOR = 5` and `MAX_DIVERGENCE_TOLERANCE_BPS = 1000`, the sellable overstatement reaches 5.5× the drained amount. The PoC uses a conservative 2.0× with zero tolerance.

The buffer leg is read live (`balanceOf`) and does update. Only the vault leg is stale, so the two halves of `availableCollateralCanonical` are measured at different points in time.

Note also that `recommitAllocation` re-validates against the same stale snapshot, while its NatSpec claims the recommit "stays bound to the period's real collateral within tolerance." For the vault leg it is not.

## Attack path

No attacker; this is an ordinary operator sequence.

{% stepper %}
{% step %}

### Stakers hold 1,000,000 FXRP

A cover order for period P is authored in P−1.
{% endstep %}

{% step %}

### Period P begins

The period-start snapshot records the full 1,000,000 FXRP.
{% endstep %}

{% step %}

### An approved incident drains the vault

One hour into P, an incident captured in P−1 is approved. The payout window is `capturePeriod` or `capturePeriod + 1`, so this is legal. The vault is drained to zero.
{% endstep %}

{% step %}

### The allocator commits against stale collateral

`_computeAvailableCapacity` reads `totalAssetsAt(periodStart)` — still 1,000,000 FXRP — and returns $2,000,000 of capacity against a vault holding nothing.
{% endstep %}

{% step %}

### Cover is settled and receipted

The allocator's routine settlement pass runs. The order settles, the buyer is charged a full-period premium, and the CoverNFT is minted — against collateral that no longer exists.
{% endstep %}
{% endstepper %}

**Ordering is not a meaningful constraint.** Periods are currently 1 day, and `commitAllocation` requires `graceExpiresAt < currentPeriodEnd()`, so with `MAX_GRACE_PERIOD = 8h` the commit must fall inside roughly the first 16 hours of the period — leaving ample room for a payout to precede it. A payout is not an exotic event; it is the protocol's core function.

## Impact Details

**Protocol insolvency.** The protocol sells cover obligations in excess of the collateral backing them, and the PoC carries this through to a completed sale rather than stopping at the accounting error. At the moment `settleCoverOrder` returns:

| Quantity                         |                Value |
| -------------------------------- | -------------------: |
| `vault.totalAssets()`            |                **0** |
| Cover written (canonical)        |   **100,000 × 10¹⁸** |
| Premium collected from the buyer |       **95.89 USDC** |
| `coverNFT.ownerOf(orderId)`      |            the buyer |
| Committed capacity               | **2,000,000 × 10¹⁸** |

The invariant `totalDeclaredAllocated ≤ matchingCapacity ≤ _computeAvailableCapacity()` is false the moment the commit lands, because the input to the last term is stale.

**Cover buyers** pay a full-period premium for cover backed by collateral the protocol has already paid out, and hold a claim on a vault that cannot pay it.

**Stakers** are levered beyond the `minCAR` floor the contract exists to enforce.

Every downstream control that derives from `commit.totalAvailableCapacity`, including the per-protocol concentration caps applied in `_settleCoverOrder`, inherits the overstatement.

## Distinction from the published Known Issues

Two entries on the program's scope page are adjacent and should be addressed directly:

* *"If the underlying staked asset decreases in USD value, available capital and the Capital Adequacy Ratio decrease. Severe collateral drawdowns can reduce coverage capacity and may cause shortfalls or undercollateralization relative to outstanding cover obligations."* — This concerns **price** decline in the staked asset, which the oracle reads live at commit time and which therefore *does* reduce capacity correctly. This finding concerns a **payout**, which reduces the asset balance but is structurally invisible to the period-start lookup. Different mechanism, and one of them is a functioning control while the other is not.
* *"Aggregate claims are not capped on-chain against total cover sold."* — That is the claims side: payouts not bounded by cover written. This is the underwriting side: cover written not bounded by collateral held. Opposite direction.

This does not appear in Macro A-1's issue list (M-1, M-2, L-1 through L-4, G-1, I-1 through I-11), in the three Known Issues, or in the 46 disclosed reports from the November 2025 competition — which had only the vault in scope. Known Issue #3 concerns consuming the same cover more than once; this concerns selling cover against collateral that is gone.

## Recommended mitigation

Bound the snapshot by live assets. At `CoverOrderAllocator.sol:711`:

```diff
-            $.vault.totalAssetsAt($.vault.currentPeriodStart()),
+            Math.min($.vault.totalAssetsAt($.vault.currentPeriodStart()), $.vault.totalAssets()),
```

This preserves the determinism the period-start snapshot provides for the off-chain matcher, while ensuring a payout can only ever reduce capacity, never leave it overstated. Control 1 shows the bound behaving correctly once its input is current, so no other change is required.

A defence-in-depth companion, if the team wants the settle path guarded independently of the commit path, is to re-validate live capacity inside `_settleCoverOrder` rather than trusting `commit.totalAvailableCapacity` alone.

## References

* `contracts/core/CoverOrderAllocator.sol:696-726` — `_computeAvailableCapacity`
* `contracts/core/CoverOrderAllocator.sol:711` — the stale period-start read (root cause)
* `contracts/core/CoverOrderAllocator.sol:394` — `_validateMatchingCapacity`
* `contracts/core/CoverOrderAllocator.sol:332` — `commitAllocation`
* `contracts/core/CoverOrderAllocator.sol:457` — `_settleCoverOrder`, which inherits the overstatement
* `contracts/core/FirelightVault.sol:952` — post-payout checkpoint written at `Time.timestamp()`
* `contracts/core/lib/Checkpoints.sol` — `upperLookupRecent`, inclusive of its key

## Proof of Concept

Foundry. Uses the shared `Base.t.sol` fixture, a Solidity port of the project's own `test/setup/fixtures.js`. This PoC calls `deployVault()` and `deployFullStack()` — the latter wires the allocator to the **real** `FirelightVault` rather than `MockCoverOrderAllocatorVault`, because the finding depends on genuine vault state and on a real `payout` executing. The first-loss buffer wallet is left at zero balance so capacity reduces cleanly to `assets × 1e12 × effectiveLeverage / minCAR`, isolating the vault leg.

`IncidentManager` is not needed: `payout` is called directly under `PAYOUT_ROLE`, and the resulting vault state is identical to the one `IncidentManager._executePayout` produces.

Four tests: the finding, the mechanism in isolation, and two controls.

### `test/Foundry/Base.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "contracts/core/CoverNFT.sol";
import {VaultRewardDistributor} from "contracts/core/VaultRewardDistributor.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "contracts/core/interfaces/IAggregatorV3.sol";

import {MockFAsset} from "./mocks/MockFAsset.sol";
import {MockERC20} from "contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "contracts/test/MockAggregatorV3.sol";
import {MockCoverOrderAllocatorVault} from "contracts/test/MockCoverOrderAllocatorVault.sol";

/**
 * @title Base
 * @notice Solidity port of test/setup/fixtures.js — `deployVault` and
 *         `deployCoverOrderAllocator` — so the Hardhat suite and the Foundry suite
 *         exercise identical deployments.
 *
 * Two deliberate deviations from the JS, both noted where they matter:
 *
 *  1. Proxies are ERC1967Proxy rather than the Transparent proxies `upgrades.deployProxy`
 *     creates. Storage layout and delegatecall semantics are identical, and it removes the
 *     ProxyAdmin indirection that would otherwise make every `vm.prank` two-hop.
 *
 *  2. The underlying is `MockFAsset`, not the real `deployFAsset` helper — contracts/fasset
 *     is pragma 0.8.23 and skipped in foundry.toml. MockFAsset reproduces
 *     `FAsset.transferFrom` line for line, including the round-UP fee, and defaults to a
 *     zero fee so it behaves exactly like a plain ERC20 until a test opts in.
 */
abstract contract Base is Test {
    // --- config (mirrors DEFAULT_CONFIG) ---
    uint8 internal constant DECIMALS = 6;
    string internal constant UNDERLYING = "fXRP";
    string internal constant LST = "stfXRP";
    uint256 internal constant INITIAL_DEPOSIT_LIMIT = 50_000_000_000; // 50k tokens
    uint48 internal constant PERIOD_DURATION = 604800; // 1 week

    // --- constants (mirrors the JS constants block) ---
    uint256 internal constant BPS = 10_000;
    uint256 internal constant LEV = 10_000;
    uint256 internal constant SECS_PER_YEAR = 365 days;
    string internal constant PROTOCOL_MORPHO = "Morpho";
    string internal constant PROTOCOL_AAVE = "Aave";
    bytes32 internal constant MARKET_A = bytes32("market-A");
    bytes32 internal constant MARKET_B = bytes32("market-B");

    // --- vault fixture ---
    MockFAsset internal token;
    FirelightVault internal vault;
    address internal deployer;
    address internal rescuer;
    address internal blocklister;
    address internal pauser;
    address internal limitUpdater;
    address internal periodConfigurationUpdater;
    address internal payoutSigner;
    address internal payoutAllowlister;
    address internal payoutReceiver;
    address internal incidentRole;
    address internal checkpointer;
    address[3] internal users;
    VaultRewardDistributor internal distributor;
    address internal distributorEOA;

    // --- allocator fixture ---
    CoverOrderAllocator internal allocator;
    CoverNFT internal coverNFT;
    MockCoverOrderAllocatorVault internal mockVault;
    MockAggregatorV3 internal priceFeed;
    MockERC20 internal usdc;
    MockERC20 internal usdt;
    address internal admin;
    address internal curator;
    address internal allocatorRole;
    address internal configAdmin;
    address internal adminRole;
    address internal premiumCollector;
    address internal firstLossBufferWallet;
    address internal buyer1;
    address internal buyer2;
    address internal beneficiary;
    bytes32 internal marketIdA;
    bytes32 internal marketIdB;
    bytes32 internal concHashMorpho;
    bytes32 internal concHashAave;

    // =====================================================================
    // deployVault  (port of fixtures.js::deployVault)
    // =====================================================================
    function deployVault() internal {
        // periodConfigurationAtTimestamp() rejects a zero epoch, and Foundry starts at
        // block.timestamp == 1. Warp to a realistic clock before deploying.
        if (block.timestamp < 1_700_000_000) vm.warp(1_700_000_000);

        deployer = _signer("deployer");
        rescuer = _signer("rescuer");
        blocklister = _signer("blocklister");
        pauser = _signer("pauser");
        limitUpdater = _signer("limitUpdater");
        periodConfigurationUpdater = _signer("periodConfigurationUpdater");
        users[0] = _signer("user1");
        users[1] = _signer("user2");
        users[2] = _signer("user3");

        token = new MockFAsset(UNDERLYING, UNDERLYING, DECIMALS);

        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: deployer,
            limitUpdater: limitUpdater,
            blocklister: blocklister,
            pauser: pauser,
            periodConfigurationUpdater: periodConfigurationUpdater,
            rescuer: rescuer,
            depositLimit: INITIAL_DEPOSIT_LIMIT,
            periodConfigurationDuration: PERIOD_DURATION
        });

        FirelightVault impl = new FirelightVault();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl), abi.encodeCall(FirelightVault.initialize, (IERC20(address(token)), LST, LST, abi.encode(p)))
        );
        vault = FirelightVault(address(proxy));

        // Payout wiring (mirrors the JS block after deployProxy)
        payoutSigner = _signer("payoutSigner");
        payoutAllowlister = _signer("payoutAllowlister");
        payoutReceiver = _signer("payoutReceiver");
        incidentRole = _signer("incident");
        checkpointer = _signer("checkpointer");

        vm.startPrank(deployer);
        vault.grantRole(vault.PAYOUT_ROLE(), payoutSigner);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), payoutAllowlister);
        vault.grantRole(vault.INCIDENT_ROLE(), incidentRole);
        vault.grantRole(vault.CHECKPOINT_ROLE(), checkpointer);
        vm.stopPrank();

        vm.prank(payoutAllowlister);
        vault.addToPayoutAllowlist(payoutReceiver);
    }

    // =====================================================================
    // deployCoverOrderAllocator  (port of fixtures.js::deployCoverOrderAllocator)
    // =====================================================================
    function deployCoverOrderAllocator() internal {
        if (block.timestamp < 1_700_000_000) vm.warp(1_700_000_000);

        admin = _signer("admin");
        curator = _signer("curator");
        allocatorRole = _signer("allocatorRole");
        configAdmin = _signer("configAdmin");
        adminRole = _signer("adminRole");
        premiumCollector = _signer("premiumCollector");
        firstLossBufferWallet = _signer("firstLossBufferWallet");
        buyer1 = _signer("buyer1");
        buyer2 = _signer("buyer2");
        beneficiary = _signer("beneficiary");

        // Mock vault — starts at currentPeriod = 1 to match production (period 0 never exists).
        // initialize() pushes the capacity-config checkpoint at effectivePeriod = 2, so tests
        // advance to period 2 before matching.
        mockVault = new MockCoverOrderAllocatorVault();
        mockVault.setCurrentPeriod(1);
        mockVault.setTotalAssets(0);
        for (uint256 i = 1; i <= 3; i++) {
            mockVault.setPeriodConfiguration(
                i, IFirelightVault.PeriodConfiguration({epoch: 0, duration: PERIOD_DURATION, startingPeriod: i})
            );
        }

        usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        usdt = new MockERC20("MockUSDT", "mUSDT", 6);
        mockVault.setAsset(address(usdc));

        // $1.00, 18 decimals. Override with priceFeed.setAnswer(x) in feed-native decimals.
        priceFeed = new MockAggregatorV3(18, int256(10 ** 18));

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLCOVER", "", admin, address(this), address(0), address(0))
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](2);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A});
        markets[1] = ICoverOrderAllocator.Market({chainId: 10, protocol: PROTOCOL_AAVE, market: MARKET_B});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](2);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: PROTOCOL_MORPHO, maxProtocolConcentrationBps: 6000
        });
        concs[1] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 10, protocol: PROTOCOL_AAVE, maxProtocolConcentrationBps: 4000
        });

        address[] memory premiumTokens = new address[](2);
        premiumTokens[0] = address(usdc);
        premiumTokens[1] = address(usdt);

        ICoverOrderAllocator.InitParams memory ip = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(mockVault)),
            premiumCollector: premiumCollector,
            coverNFT: coverNFT,
            priceFeedAdapter: IAggregatorV3(address(priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: adminRole,
            curatorRole: curator,
            allocatorRole: allocatorRole,
            configAdminRole: configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12000, // 1.2x, the MIN_CAR_BPS floor
                firstLossBufferToken: IERC20(address(usdc)),
                firstLossBuffer: firstLossBufferWallet,
                effectiveLeverage: 24000, // 2.0x multiplier (24000/12000)
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(new ERC1967Proxy(address(aImpl), abi.encodeCall(CoverOrderAllocator.initialize, (ip))))
        );

        // Same prank-consumption trap as above: cache the role first.
        bytes32 minterRole = coverNFT.MINTER_ROLE();
        vm.prank(admin);
        coverNFT.grantRole(minterRole, address(allocator));

        marketIdA = computeMarketId(1, PROTOCOL_MORPHO, MARKET_A);
        marketIdB = computeMarketId(10, PROTOCOL_AAVE, MARKET_B);
        concHashMorpho = computeProtocolConcentrationHash(1, PROTOCOL_MORPHO);
        concHashAave = computeProtocolConcentrationHash(10, PROTOCOL_AAVE);
    }

    // =====================================================================
    // deployFullStack — allocator wired to the REAL vault
    // =====================================================================
    /**
     * @dev `deployCoverOrderAllocator()` points the allocator at `MockCoverOrderAllocatorVault`,
     *      which returns hardcoded values and cannot execute a real `payout`. Anything that
     *      tests the *interaction* between vault state and allocator capacity needs the real
     *      vault instead.
     *
     *      `FirelightVault` does not inherit `IFirelightVault`, but every selector the allocator
     *      calls matches, so the interface cast resolves at runtime. (Struct nominal typing only
     *      binds at compile time; through an interface it is a plain ABI call.)
     *
     *      Capacity is deliberately isolated to the vault leg: the first-loss buffer wallet is
     *      left at zero balance, so `_computeAvailableCapacity` reduces to
     *      `totalAssetsAt(periodStart) * 1e12 * effectiveLeverage / minCAR`.
     *
     *      Call after `deployVault()`.
     */
    function deployFullStack() internal {
        admin = _signer("admin");
        curator = _signer("curator");
        allocatorRole = _signer("allocatorRole");
        configAdmin = _signer("configAdmin");
        adminRole = _signer("adminRole");
        premiumCollector = _signer("premiumCollector");
        firstLossBufferWallet = _signer("firstLossBufferWallet");
        buyer1 = _signer("buyer1");
        buyer2 = _signer("buyer2");
        beneficiary = _signer("beneficiary");

        usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        priceFeed = new MockAggregatorV3(18, int256(10 ** 18)); // $1.00

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLCOVER", "", admin, address(this), address(0), address(0))
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: PROTOCOL_MORPHO, maxProtocolConcentrationBps: 10000
        });

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(usdc);

        ICoverOrderAllocator.InitParams memory ip = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)), // THE REAL VAULT
            premiumCollector: premiumCollector,
            coverNFT: coverNFT,
            priceFeedAdapter: IAggregatorV3(address(priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: adminRole,
            curatorRole: curator,
            allocatorRole: allocatorRole,
            configAdminRole: configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12000,
                firstLossBufferToken: IERC20(address(usdc)),
                firstLossBuffer: firstLossBufferWallet, // left at ZERO balance
                effectiveLeverage: 24000, // 2.0x multiplier
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(new ERC1967Proxy(address(aImpl), abi.encodeCall(CoverOrderAllocator.initialize, (ip))))
        );

        bytes32 minterRole = coverNFT.MINTER_ROLE();
        vm.prank(admin);
        coverNFT.grantRole(minterRole, address(allocator));

        marketIdA = computeMarketId(1, PROTOCOL_MORPHO, MARKET_A);
        concHashMorpho = computeProtocolConcentrationHash(1, PROTOCOL_MORPHO);
    }

    /// @dev MockAggregatorV3 pins `updatedAt` at construction, so any `vm.warp` past
    ///      `maxPriceAge` (7 days here) makes `PriceFeed.getPrice` revert `PriceFeedTooOld`.
    ///      Call before any commit/recommit that follows a warp.
    function refreshPriceFeed() internal {
        priceFeed.setUpdatedAt(block.timestamp);
    }

    /// @dev Capacity the allocator will compute, replicating `_computeAvailableCapacity`
    ///      with a zero first-loss buffer and a $1.00 / 18-decimal feed.
    function expectedCapacity(uint256 vaultAssets6dp) internal pure returns (uint256) {
        return (vaultAssets6dp * 10 ** 12) * 24000 / 12000;
    }

    // =====================================================================
    // Helpers (ports of the JS utility closures)
    // =====================================================================

    /// @dev fixtures.js::utils.mintAndApprove
    function mintAndApprove(uint256 amount, address user) internal {
        token.mintTo(user, amount);
        vm.prank(user);
        token.approve(address(vault), amount);
    }

    /**
     * @dev Wires the real `VaultRewardDistributor` — the protocol's ONLY on-chain path for
     *      returning premium to stakers. Premiums are collected in a premium token, swapped
     *      off-chain, and pushed back in as vault asset; the distributor then calls
     *      `checkpointTotalAssets()`, which is why the DISTRIBUTOR CONTRACT (not the EOA)
     *      needs CHECKPOINT_ROLE on the vault.
     *
     *      `distributeRewards` pulls with `safeTransferFrom(msg.sender, vault, amount)`, so
     *      the EOA holding DISTRIBUTOR_ROLE must hold the asset and approve the distributor.
     *
     *      Call after `deployVault()`.
     */
    function deployRewardDistributor() internal {
        distributorEOA = _signer("distributorEOA");

        VaultRewardDistributor dImpl = new VaultRewardDistributor();
        distributor = VaultRewardDistributor(
            address(
                new ERC1967Proxy(
                    address(dImpl),
                    abi.encodeCall(
                        VaultRewardDistributor.initialize,
                        (IFirelightVault(address(vault)), deployer, distributorEOA, address(0))
                    )
                )
            )
        );

        // NOTE: cache the role BEFORE vm.prank. `vault.CHECKPOINT_ROLE()` is an external
        // staticcall, Solidity evaluates arguments before the call, and vm.prank applies to
        // the NEXT call only — so inlining it consumes the prank and grantRole runs as the
        // test contract, reverting AccessControlUnauthorizedAccount(<test>, DEFAULT_ADMIN_ROLE).
        bytes32 checkpointRole = vault.CHECKPOINT_ROLE();
        vm.prank(deployer);
        vault.grantRole(checkpointRole, address(distributor));
    }

    /// @dev Push `amount` of vault asset in as period premium, exactly as production does.
    function distributePremium(uint256 amount) internal {
        token.mintTo(distributorEOA, amount);
        vm.startPrank(distributorEOA);
        token.approve(address(distributor), amount);
        distributor.distributeRewards(amount, address(token), amount, uint48(block.timestamp));
        vm.stopPrank();
    }

    /**
     * @dev The fixture keeps `fixtures.js`'s 50,000-token deposit limit (5e10 at 6 decimals).
     *      That is small enough that most PoCs trip `DepositLimitExceeded` before reaching
     *      the behaviour under test — `_depositFunds` checks `totalAssets() + assets > depositLimit`.
     *      Raise it through the real `DEPOSIT_LIMIT_UPDATE_ROLE` rather than editing the fixture,
     *      so the deployment stays identical to the Hardhat suite.
     *
     *      1e15 == 1 billion tokens: ample headroom, and small enough that
     *      `convertToShares(depositLimit)` inside `maxMint` cannot overflow `mulDiv`.
     */
    function raiseDepositLimit() internal {
        raiseDepositLimit(1_000_000_000e6);
    }

    function raiseDepositLimit(uint256 newLimit) internal {
        vm.prank(limitUpdater);
        vault.updateDepositLimit(newLimit);
    }

    /// @dev fixtures.js::fundAndApprove
    function fundAndApprove(address buyer, MockERC20 t, uint256 amount) internal {
        t.mint(buyer, amount);
        vm.prank(buyer);
        t.approve(address(allocator), amount);
    }

    /// @dev fixtures.js::advanceToPeriod (mock vault only)
    function advanceToPeriod(uint256 p) internal {
        mockVault.setCurrentPeriod(p);
    }

    /// @dev Advance the REAL vault by whole periods.
    function warpPeriods(uint256 n) internal {
        vm.warp(block.timestamp + n * PERIOD_DURATION);
    }

    /**
     * @dev Timestamp at which `period` begins, for the real vault.
     *
     * NOTE the type. `PeriodConfiguration` is declared TWICE in this codebase —
     * `FirelightVaultStorage.sol:51` and `IFirelightVault.sol:33` — and they are distinct
     * nominal types. `FirelightVault` inherits FirelightVaultStorage and does NOT implement
     * IFirelightVault, so this getter returns the STORAGE struct. Using the interface struct
     * here fails with "not implicitly convertible".
     *
     * Conversely `MockCoverOrderAllocatorVault is IFirelightVault`, so its
     * `setPeriodConfiguration` takes the INTERFACE struct. Both spellings are correct in
     * their own place; do not unify them.
     *
     * For the current period prefer `vault.currentPeriodStart()`, which needs no struct.
     */
    function periodStart(uint256 period) internal view returns (uint48) {
        FirelightVault.PeriodConfiguration memory c = vault.periodConfigurationAtNumber(period);
        return uint48(c.epoch + (period - c.startingPeriod) * c.duration);
    }

    /// @dev fixtures.js::computeMarketId
    function computeMarketId(uint64 chainId, string memory protocol, bytes32 market) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol, market));
    }

    /// @dev fixtures.js::computeProtocolConcentrationHash
    function computeProtocolConcentrationHash(uint64 chainId, string memory protocol) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol));
    }

    /**
     * @dev Leaf encoding used by `_settleCoverOrder`: OZ StandardMerkleTree double-hashing.
     *      For a SINGLE-order tree the root IS the leaf and the proof is empty, which is all
     *      most PoCs need. Multi-order trees need a real tree builder.
     */
    function settleLeaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }

    /// @dev Two-leaf tree, sorted-pair hashing (OZ MerkleProof convention).
    function merkleRoot2(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(abi.encode(a, b)) : keccak256(abi.encode(b, a));
    }

    /// @dev Funded EOA, replacing the JS `randomSigner()`.
    function _signer(string memory label) internal returns (address a) {
        a = makeAddr(label);
        vm.deal(a, 1000 ether);
    }

    /// @dev Single-market allocation array, the shape most PoCs want.
    function oneMarket(bytes32 marketId, uint256 amount)
        internal
        pure
        returns (ICoverOrderAllocator.MarketCoverAllocation[] memory arr)
    {
        arr = new ICoverOrderAllocator.MarketCoverAllocation[](1);
        arr[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: amount});
    }

    function oneMarketInput(bytes32 marketId, uint32 rateAnnual, uint256 amount)
        internal
        pure
        returns (ICoverOrderAllocator.MarketAllocationInput[] memory arr)
    {
        arr = new ICoverOrderAllocator.MarketAllocationInput[](1);
        arr[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: rateAnnual, coverAmount: amount
        });
    }
}
```

### `test/Foundry/mocks/MockFAsset.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/// Replica of contracts/fasset/fassetToken/implementation/FAsset.sol, which is pragma
/// 0.8.23 and skipped in foundry.toml. The transfer fee defaults to zero, so this behaves
/// as a plain ERC20 throughout this PoC.
contract MockFAsset is ERC20 {
    uint8   private immutable _decimals;
    uint256 public  transferFeeMillionths;
    address public  feeCollector = address(0xFEE);

    constructor(string memory n, string memory s, uint8 d) ERC20(n, s) { _decimals = d; }

    function decimals() public view override returns (uint8) { return _decimals; }
    function setTransferFeeMillionths(uint256 v) external { transferFeeMillionths = v; }
    function mintTo(address to, uint256 amount) external { _mint(to, amount); }
    function mint(address to, uint256 amount)   external { _mint(to, amount); }

    function _transferFeeAmount(uint256 amount) public view returns (uint256) {
        if (transferFeeMillionths == 0) return 0;
        return Math.mulDiv(amount, transferFeeMillionths, 1_000_000, Math.Rounding.Ceil);
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 fee = _transferFeeAmount(amount);
        _transfer(_msgSender(), to, amount - fee);
        if (fee > 0) _transfer(_msgSender(), feeCollector, fee);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        _spendAllowance(from, _msgSender(), amount);
        uint256 fee = _transferFeeAmount(amount);
        _transfer(from, to, amount - fee);
        if (fee > 0) _transfer(from, feeCollector, fee);
        return true;
    }
}
```

### `test/Foundry/UnderwritingAfterLoss.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Base} from "./Base.t.sol";
import {console2} from "forge-std/Test.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";

contract UnderwritingAfterLoss is Base {
    uint256 constant STAKE = 1_000_000e6; // 1,000,000 FXRP of staked collateral (6 dp)
    uint256 constant COVER = 100_000e18; // $100,000 of cover, canonical 18 dp

    function setUp() public {
        deployVault();
        deployFullStack(); // allocator pointed at the REAL vault
        raiseDepositLimit();
    }

    // cover is sold, paid for and receipted against a vault that holds nothing.
    function testCoverIsSoldAndPaidForAgainstAVaultHoldingNothing() public {
        // period P-1: stakers post collateral, a cover order is authored
        mintAndApprove(STAKE, users[0]);
        vm.prank(users[0]);
        vault.deposit(STAKE, users[0]);

        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer1,
            beneficiary,
            "xrp-beneficiary",
            address(usdc),
            oneMarketInput(marketIdA, 500, COVER), // 5.00% annual
            ICoverOrderAllocator.CoverOrderType.NEW
        );

        // period P opens; the snapshot records the full collateral
        warpPeriods(1);
        uint48 periodStartTs = vault.currentPeriodStart();
        uint256 period = vault.currentPeriod();

        // One hour in. `upperLookupRecent` is inclusive of its key, so a payout in the
        // period's FIRST block writes its checkpoint exactly at `periodStartTs` and IS
        // seen — Control 2 below proves that. Every later payout is invisible, which is
        // every real one given the 48h/24h/72h claims pipeline.
        vm.warp(block.timestamp + 1 hours);
        refreshPriceFeed();

        uint256 capacity = expectedCapacity(vault.totalAssetsAt(periodStartTs));
        console2.log("collateral at period start :", vault.totalAssetsAt(periodStartTs));
        console2.log("sellable capacity          :", capacity);

        // a prior-period incident is approved mid-period and drains the vault Legal: the payout window is capturePeriod or capturePeriod + 1.
        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, STAKE, periodStartTs);

        assertEq(vault.totalAssets(), 0, "vault has been emptied");
        assertEq(
            vault.totalAssetsAt(periodStartTs), STAKE,
            "the period-start snapshot is untouched by the payout"
        );

        // the allocator commits against the stale snapshot
        ICoverOrderAllocator.MarketCoverAllocation[] memory allocs = oneMarket(marketIdA, COVER);
        bytes32 root = settleLeaf(orderId, allocs);

        vm.prank(allocatorRole);
        allocator.commitAllocation(period, root, COVER, capacity);

        ICoverOrderAllocator.AllocationCommitment memory c =
            allocator.getAllocationCommitment(period);
        assertEq(c.totalAvailableCapacity, capacity, "capacity unchanged by a total payout");

        // and the cover is SOLD: premium taken, receipt minted
        uint256 premium = allocator.getCoverOrder(orderId).totalPremiumAmount;
        fundAndApprove(buyer1, usdc, 1_000_000e6);

        uint256 collectorBefore = usdc.balanceOf(premiumCollector);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, allocs, new bytes32[](0));
        uint256 collected = usdc.balanceOf(premiumCollector) - collectorBefore;

        ICoverOrderAllocator.CoverOrder memory o = allocator.getCoverOrder(orderId);

        console2.log("--- at the moment the cover was sold ---");
        console2.log("vault totalAssets()        :", vault.totalAssets());
        console2.log("cover written (canonical)  :", o.allocatedCoverAmount);
        console2.log("premium charged (canonical):", premium);
        console2.log("premium collected (USDC)   :", collected);
        console2.log("CoverNFT owner             :", coverNFT.ownerOf(orderId));

        assertEq(
            uint8(o.status), uint8(ICoverOrderAllocator.CoverOrderStatus.MATCHED),
            "the order settled"
        );
        assertEq(o.allocatedCoverAmount, COVER, "full cover written");
        assertGt(collected, 0, "the buyer paid a full-period premium");
        assertEq(coverNFT.ownerOf(orderId), buyer1, "cover receipt issued to the buyer");

        // The obligation exists, money changed hands, and there is nothing behind it.
        assertEq(vault.totalAssets(), 0, "backing collateral at the moment of sale: zero");
    }

    // The mechanism in isolation — no order, no settlement, just the stale capacity number surviving a total payout.
    function testStaleSnapshotLetsCapacitySurviveAPayoutThatEmptiesTheVault() public {
        mintAndApprove(STAKE, users[0]);
        vm.prank(users[0]);
        vault.deposit(STAKE, users[0]);

        warpPeriods(1);
        uint48 periodStartTs = vault.currentPeriodStart();
        uint256 period = vault.currentPeriod();

        vm.warp(block.timestamp + 1 hours);
        refreshPriceFeed();

        uint256 capacityBefore = expectedCapacity(vault.totalAssetsAt(periodStartTs));

        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, STAKE, periodStartTs);
        assertEq(vault.totalAssets(), 0, "vault is empty");
        assertEq(
            vault.totalAssetsAt(periodStartTs), STAKE,
            "snapshot still reports pre-payout collateral"
        );

        vm.prank(allocatorRole);
        allocator.commitAllocation(period, keccak256("root"), capacityBefore, capacityBefore);

        ICoverOrderAllocator.AllocationCommitment memory c =
            allocator.getAllocationCommitment(period);
        console2.log("vault totalAssets()   :", vault.totalAssets());
        console2.log("committed capacity    :", c.totalAvailableCapacity);
        console2.log("declared allocation   :", c.totalDeclaredAllocated);

        assertEq(c.totalAvailableCapacity, capacityBefore, "capacity unchanged by a total payout");
        assertGt(
            c.totalDeclaredAllocated, 0,
            "cover committed against collateral that no longer exists"
        );
    }

    // Control 1 — the bound itself works. Once the snapshot reflects
    // reality the same commit is correctly rejected, which isolates the
    // defect to the input rather than to the check.
    function testControlCapacityIsEnforcedWhenTheSnapshotIsCurrent() public {
        mintAndApprove(STAKE, users[0]);
        vm.prank(users[0]);
        vault.deposit(STAKE, users[0]);

        warpPeriods(1);
        uint48 periodStartTs = vault.currentPeriodStart();
        uint256 capacityBefore = expectedCapacity(vault.totalAssetsAt(periodStartTs));

        vm.warp(block.timestamp + 1 hours);
        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, STAKE, periodStartTs);

        warpPeriods(1);
        refreshPriceFeed();
        uint48 freshStart = vault.currentPeriodStart();
        uint256 freshPeriod = vault.currentPeriod();
        assertEq(vault.totalAssetsAt(freshStart), 0, "snapshot has caught up");

        vm.prank(allocatorRole);
        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector, capacityBefore, 0
            )
        );
        allocator.commitAllocation(freshPeriod, keccak256("root"), capacityBefore, capacityBefore);
    }

    // Control 2 — the boundary claim, proved rather than asserted. A
    // payout landing in the period's FIRST block writes its checkpoint
    // at exactly periodStartTs, the inclusive lookup sees it, and the
    // same commit is correctly rejected. The defect is confined to
    // payouts strictly after a period's opening block.
    function testControlPayoutInThePeriodsFirstBlockIsVisibleToCapacity() public {
        mintAndApprove(STAKE, users[0]);
        vm.prank(users[0]);
        vault.deposit(STAKE, users[0]);

        warpPeriods(1);
        uint48 periodStartTs = vault.currentPeriodStart();
        uint256 period = vault.currentPeriod();
        vm.warp(periodStartTs); // sit exactly on the boundary

        uint256 capacityBefore = expectedCapacity(vault.totalAssetsAt(periodStartTs));
        assertGt(capacityBefore, 0, "collateral is visible before the payout");

        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, STAKE, periodStartTs);

        assertEq(
            vault.totalAssetsAt(periodStartTs), 0,
            "a FIRST-BLOCK payout IS visible to the period-start lookup"
        );

        refreshPriceFeed();
        vm.prank(allocatorRole);
        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector, capacityBefore, 0
            )
        );
        allocator.commitAllocation(period, keccak256("root"), capacityBefore, capacityBefore);
    }
}
```

### Output

```
$ forge test --match-contract UnderwritingAfterLoss -vv

Ran 4 tests for test/Foundry/UnderwritingAfterLoss.t.sol:UnderwritingAfterLoss
[PASS] testControlCapacityIsEnforcedWhenTheSnapshotIsCurrent() (gas: 598321)
[PASS] testControlPayoutInThePeriodsFirstBlockIsVisibleToCapacity() (gas: 594216)
[PASS] testCoverIsSoldAndPaidForAgainstAVaultHoldingNothing() (gas: 1439362)
Logs:
  collateral at period start : 1000000000000
  sellable capacity          : 2000000000000000000000000
  --- at the moment the cover was sold ---
  vault totalAssets()        : 0
  cover written (canonical)  : 100000000000000000000000
  premium charged (canonical): 95890410958904109590
  premium collected (USDC)   : 95890411
  CoverNFT owner             : 0x29805Ff5b946e7A7c5871c1Fb071f740f767Cf41

[PASS] testStaleSnapshotLetsCapacitySurviveAPayoutThatEmptiesTheVault() (gas: 698300)
Logs:
  vault totalAssets()   : 0
  committed capacity    : 2000000000000000000000000
  declared allocation   : 2000000000000000000000000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 7.28ms (4.19ms CPU time)
```

The four runs bracket the defect from both sides:

| Run             | What it establishes                                                                                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **The finding** | The vault holds **0**; $100,000 of cover settles, premium is collected, the receipt is minted.                                                                |
| Mechanism       | The same stale snapshot lets **$2,000,000** of capacity be committed against a zero vault — no order needed.                                                  |
| Control 1       | Once the snapshot is current, the identical commit reverts `MatchingCapacityOverflow`. The bound works; only its input is wrong.                              |
| Control 2       | A payout on the period's **first block** *is* seen, and the commit is correctly rejected. The defect is precisely "payouts strictly after the opening block." |

Control 1 forecloses "the capacity check is simply misconfigured" — it is not, and it functions correctly the moment it is given live data. Control 2 forecloses "the lookup is fine, you tested it wrong" — the inclusive-lookup boundary is reproduced and shown behaving correctly, which is the only configuration in which this defect does *not* appear.
