# 89904 sc medium rescuewithdrawfromblocklisted marks a still future withdrawal period as claimed permanently locking any withdrawal the rescued account later makes into that period

**Submitted on Aug 21st 2026 at 18:23:45 UTC by @SavantChat for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89904
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal shares to a rescue address and then stamps `isWithdrawClaimed[period][from] = true`.

That flag is a **terminal** marker — it means "this account has taken its payout for this period". But the rescued period is still in the **future**: it keeps accepting new withdrawal requests, including from the same account once the blocklist is lifted. Nothing ever clears the flag; `isWithdrawClaimed` is only ever written to `true`, at two sites, and there is no reset path of any kind.

So after an ordinary, correct sequence — rescue, then un-blocklist — the account's next withdrawal lands in a period where it already counts as paid. `claimWithdraw` reverts `AlreadyClaimedPeriod` before it ever computes the entitlement, and a second rescue reverts on the same check. The newly reserved assets sit in `pendingWithdrawAssets` permanently, excluded from `totalAssets()` and reachable by nobody.

No role misbehaves here. The rescuer performs exactly the operation the function exists for, the blocklister lifts a block exactly as intended, and the user makes an ordinary withdrawal request. The contract corrupts its own state along the way, and the loss falls on the user.

## Vulnerability Details

### The premature terminal marker

`contracts/core/FirelightVault.sol:845-876`:

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;   // <-- terminal marker, non-terminal period

    rescuedShares[i] = _withdrawOf;
}
```

Zeroing `withdrawSharesOf[period][from]` is sufficient to express "nothing left here for this account". The extra flag adds nothing at rescue time and becomes wrong the moment the same account acquires shares in that period again.

### Nothing clears it

`isWithdrawClaimed` has exactly two writers, both setting `true`:

* `claimWithdraw` (`:806`), after a successful payout — legitimately terminal, because the period is already past (`:791` requires `period < currentPeriod()`).
* `rescueWithdrawFromBlocklisted` (`:870`) — on a period that has not started.

There is no `delete`, no setter, no admin override. Once set, it is set forever.

### Both exits close

The account now satisfies `withdrawSharesOf[period][account] > 0` **and** `isWithdrawClaimed[period][account] == true`.

* `claimWithdraw` (`:794`) checks the flag before computing anything:

  ```solidity
  if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
  ```
* A repeat rescue (`:864`) checks the same flag on `from` and reverts identically.

`rescueSharesFromBlocklisted` (`:818`) does not help either — it transfers vault *shares*, not withdrawal-bucket entitlements.

### The window

The collision requires the new request to land in the same period that was rescued. Withdrawals are booked to `currentPeriod() + 1` (`:1008`), so the sequence — rescue, un-blocklist, new withdrawal — must complete while the period counter has not advanced, i.e. within one period. With the deployed cadence (the project's own configuration uses 2 days and 1 week) that is a window of days, not seconds.

The account also cannot withdraw while blocklisted: `withdraw` carries `notBlocklisted(_msgSender())` and `notBlocklisted(owner)` (`:755-768`). The un-blocklist step is therefore load-bearing — but `removeFromBlocklist` (`:560`) is a plain `BLOCKLIST_ROLE` operation and lifting a block after a compliance matter is resolved is the expected lifecycle, not an unusual event.

## Impact

Permanent, unrecoverable locking of a user's assets, produced by correct operation of the rescue path.

The PoC walks the full sequence with a 6-decimal vault:

1. Alice deposits 10,000 and requests a 4,000 withdrawal into `targetPeriod`.
2. Alice is blocklisted.
3. `rescueWithdrawFromBlocklisted(alice, bob, [targetPeriod])` moves her 4,000 of shares to Bob and sets `isWithdrawClaimed[targetPeriod][alice] = true`.
4. Alice is un-blocklisted.
5. Alice requests a further 6,000 withdrawal — still `targetPeriod`, so `withdrawSharesOf[targetPeriod][alice] == 6_000e6` under the poisoned flag.
6. The period passes.
7. `claimWithdraw(targetPeriod)` reverts `AlreadyClaimedPeriod`.
8. Re-blocklisting and rescuing again reverts with the same error.
9. Bob still claims his rescued 4,000 normally — isolating the failure to Alice's new entitlement — while `pendingWithdrawAssets` retains her 6,000 forever.

Consequences:

* The user's assets are gone from their perspective: no claim, no rescue, no admin remedy.
* `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets` (`:427-429`), so the stranded amount is permanently subtracted from the vault's accounting while physically remaining in the contract — it stops backing any share yet cannot be withdrawn.
* The damage scales with the second request, which is unbounded: the locked amount is whatever the user withdraws next, not the rescued amount.
* The trigger is a benign operational sequence. No role has to act maliciously or outside its remit, which is what makes this likely to happen by accident rather than by attack.

## Recommendation

Drop the flag write. Zeroing the share balance already encodes everything the rescue needs:

```solidity
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;
```

The pre-checks at `:864-866` remain meaningful: they still prevent rescuing a period the account has genuinely already claimed, and `_withdrawOf == 0` still rejects an empty rescue.

If the flag is wanted as an audit trail, keep it in a separate mapping that `claimWithdraw` does not consult, or set it only when the rescued period is already in the past (`periods[i] < currentPeriod()`), where "claimed" is actually terminal.

As defence in depth, consider making `claimWithdraw` treat the flag as a guard against *double* payment rather than as a permanent seal — for example, keying it on the entitlement actually settled — so that a stale marker can never strand assets that were reserved afterwards.

## References

* `contracts/core/FirelightVault.sol:845-876` — `rescueWithdrawFromBlocklisted`
* `contracts/core/FirelightVault.sol:870` — the premature `isWithdrawClaimed[period][from] = true`
* `contracts/core/FirelightVault.sol:864-865` — the same flag blocks a second rescue
* `contracts/core/FirelightVault.sol:794` — `claimWithdraw` rejects on the flag before computing the entitlement
* `contracts/core/FirelightVault.sol:806` — the only legitimate setter, on an already-past period
* `contracts/core/FirelightVault.sol:791` — `claimWithdraw` requires `period < currentPeriod()`
* `contracts/core/FirelightVault.sol:1008` — requests are booked to `currentPeriod() + 1`
* `contracts/core/FirelightVault.sol:755-768` — `withdraw` is `notBlocklisted`, so the un-blocklist step is required
* `contracts/core/FirelightVault.sol:560` — `removeFromBlocklist`, an ordinary `BLOCKLIST_ROLE` action
* `contracts/core/FirelightVault.sol:427-429` — `totalAssets()` subtracts `pendingWithdrawAssets`
* `contracts/core/FirelightVault.sol:818` — `rescueSharesFromBlocklisted` moves vault shares, not bucket entitlements

### Relation to previously reported issues

None of the three known issues on the program page (first-loss-buffer valuation, the unescrowed buffer wallet, uncapped aggregate claims) concerns the blocklist or rescue paths.

The 0xMacro audit of 26 June – 13 July 2026 (commit `d0d3733`) contains no finding on `rescueWithdrawFromBlocklisted`, on `isWithdrawClaimed`, or on the blocklist lifecycle; its findings are M-1, M-2, L-1..L-4, G-1 and I-1..I-11.

## Proof of Concept

## Proof of Concept

Runnable Foundry PoC. Verified against a clean clone of the audited repository at tag `v1_audit_ready` (commit `42f9ea5`) with forge 1.5.1 / solc 0.8.28. It drives the **real** `FirelightVault` behind an ERC1967 proxy — nothing is simulated by a mock. Everything needed to reproduce is inlined below.

### 1. Where each file goes

The repository ships with a hardhat setup only, so the PoC adds a foundry profile on top of it. Place the four files below at these exact paths, relative to the repository root:

```
audit-comp-firelight/
├── foundry.toml                                   <- new (block 1.1)
├── remappings.txt                                 <- new (block 1.2)
└── test/
    └── foundry/
        ├── F10_RescueClaimLock.t.sol               <- the PoC (block 2)
        └── utils/
            └── Deployers.sol                       <- shared harness (block 3)
```

#### 1.1 `foundry.toml` (repository root)

`via_ir = true` is mandatory — the plain pipeline hits `Stack too deep` in `CoverOrderAllocator`.

```toml
[profile.default]
src = "contracts"
test = "test/foundry"
out = "out-foundry"
libs = ["node_modules", "lib"]
solc_version = "0.8.28"
optimizer = true
optimizer_runs = 10000
evm_version = "london"
via_ir = true
fs_permissions = [{ access = "read", path = "./" }]
skip = [
    "contracts/fasset/**",
    "contracts/legacy/**",
]

[profile.default.fuzz]
runs = 32

[profile.ci]
fuzz_runs = 128
```

#### 1.2 `remappings.txt` (repository root)

```
@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/
@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/
```

### 2. The PoC — `test/foundry/F10_RescueClaimLock.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {Deployers} from "./utils/Deployers.sol";
import {FirelightVault} from "../../contracts/core/FirelightVault.sol";
import {MockERC20} from "../../contracts/test/MockERC20.sol";

// rescueWithdrawFromBlocklisted stamps a terminal "claimed" marker on a period that is
// not yet terminal, permanently locking any withdrawal the same account makes into it.
//
// Root cause: the rescue loop sets isWithdrawClaimed[period][from] = true
// (FirelightVault.sol:870) after moving the pending shares away. The rescued period is
// still in the future and can keep accepting withdrawal requests, but the flag is never
// cleared -- isWithdrawClaimed is only ever written to true (:806, :870) and no path
// resets it. Once the account is un-blocklisted (:560, an ordinary BLOCKLIST_ROLE action)
// and requests a withdrawal that lands in the same period, it holds positive
// withdrawSharesOf under a set claim flag: claimWithdraw reverts AlreadyClaimedPeriod
// (:794) and a second rescue reverts on the same check (:864), so the newly reserved
// assets stay in pendingWithdrawAssets forever.

contract F10_RescueClaimLockTest is Test, Deployers {
    function test_rescuePrematureClaimFlagLocksFunds() public {
        VaultCtx memory ctx = deployVault(6, 100_000e6);
        FirelightVault vault = ctx.vault;
        MockERC20 token = ctx.token;
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");

        uint256 deposit = 10_000e6;
        mintAndApprove(ctx, alice, deposit);
        vm.prank(alice);
        vault.deposit(deposit, alice);

        uint256 startPeriod = vault.currentPeriod();
        uint256 targetPeriod = startPeriod + 1;

        // 1) Alice requests a withdrawal (assigned to targetPeriod).
        vm.prank(alice);
        vault.withdraw(4_000e6, alice, alice);

        // 2) Alice is blocklisted.
        vm.prank(ctx.blocklister);
        vault.addToBlocklist(alice);

        // 3) Rescue moves the pending shares to Bob and (bug) stamps
        //    isWithdrawClaimed[targetPeriod][alice] = true.
        vm.prank(ctx.rescuer);
        vault.rescueWithdrawFromBlocklisted(alice, bob, _arr(targetPeriod));

        assertEq(vault.withdrawSharesOf(targetPeriod, alice), 0);
        assertEq(vault.withdrawSharesOf(targetPeriod, bob), 4_000e6);
        assertTrue(vault.isWithdrawClaimed(targetPeriod, alice));

        // 4) Alice is unblocked before the period advances.
        vm.prank(ctx.blocklister);
        vault.removeFromBlocklist(alice);

        // 5) A new withdrawal still targets the same period; Alice has positive
        //    shares AND a poisoned claim flag.
        vm.prank(alice);
        vault.withdraw(6_000e6, alice, alice);
        assertEq(vault.withdrawSharesOf(targetPeriod, alice), 6_000e6);
        assertTrue(vault.isWithdrawClaimed(targetPeriod, alice));

        // 6) Advance beyond targetPeriod.
        vm.warp(vault.currentPeriodStart() + ctx.periodDuration * 2);
        assertGt(vault.currentPeriod(), targetPeriod);

        // 7) claimWithdraw reverts before considering the new entitlement.
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
        vault.claimWithdraw(targetPeriod);

        // 8) Re-blocklist + second rescue also reverts: the lock is permanent.
        vm.prank(ctx.blocklister);
        vault.addToBlocklist(alice);
        vm.prank(ctx.rescuer);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
        vault.rescueWithdrawFromBlocklisted(alice, bob, _arr(targetPeriod));

        // 9) The 6_000e6 newly reserved assets are stranded inside pendingWithdrawAssets
        //    (Bob's rescued 4_000e6 remains claimable, only Alice's new request is locked).
        assertEq(vault.pendingWithdrawAssets(), 10_000e6);
        uint256 reserves = token.balanceOf(address(vault));
        assertEq(vault.totalAssets(), reserves - 10_000e6);

        // Bob CAN still claim his rescued entitlement...
        vm.prank(bob);
        vault.claimWithdraw(targetPeriod);
        assertEq(token.balanceOf(bob), 4_000e6);
        // ...while Alice's newly reserved 6_000e6 remains permanently locked above.
        assertEq(vault.pendingWithdrawAssets(), 6_000e6);
    }

    function _arr(uint256 a) internal pure returns (uint256[] memory arr) {
        arr = new uint256[](1);
        arr[0] = a;
    }
}
```

### 3. Shared harness — `test/foundry/utils/Deployers.sol`

Deploys `FirelightVault` behind an ERC1967 proxy and grants the roles the test needs — here `BLOCKLIST_ROLE` and `RESCUER_ROLE` — with a 1-day period configuration. It uses only mocks already present in the repository under `contracts/test/`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../../../contracts/core/FirelightVault.sol";
import {IncidentManager} from "../../../contracts/core/IncidentManager.sol";
import {CoverOrderAllocator} from "../../../contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "../../../contracts/core/CoverNFT.sol";
import {ICoverOrderAllocator} from "../../../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "../../../contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "../../../contracts/core/interfaces/IAggregatorV3.sol";
import {IIncidentManager} from "../../../contracts/core/interfaces/IIncidentManager.sol";
import {MockERC20} from "../../../contracts/test/MockERC20.sol";
import {MockPriceFeed} from "../../../contracts/test/MockPriceFeed.sol";
import {MockAggregatorV3} from "../../../contracts/test/MockAggregatorV3.sol";
import {MockIncidentManagerVault} from "../../../contracts/test/MockIncidentManagerVault.sol";
import {MockIncidentManagerCoverOrderAllocator} from "../../../contracts/test/MockIncidentManagerCoverOrderAllocator.sol";
import {MockCoverOrderAllocatorVault} from "../../../contracts/test/MockCoverOrderAllocatorVault.sol";

/// @dev Shared deployment harness for the Firelight foundry PoC testbed.
contract Deployers is Test {
    uint48 public constant PERIOD_DURATION = 1 days;
    uint256 public constant BPS = 10_000;
    uint256 public constant SECONDS_PER_YEAR = 365 days;

    struct VaultCtx {
        FirelightVault vault;
        MockERC20 token;
        address admin;
        address rescuer;
        address blocklister;
        address pauser;
        address limitUpdater;
        address periodConfigUpdater;
        address payoutSigner; // PAYOUT_ROLE
        address payoutAllowlister; // PAYOUT_ALLOWLIST_ROLE
        address payoutReceiver; // allowlisted payout destination
        uint48 periodDuration;
    }

    event Dummy(); // silence unused-var warnings

    function deployVault(uint8 decimals, uint256 depositLimit) internal returns (VaultCtx memory ctx) {
        ctx = _deployVault(decimals, depositLimit, PERIOD_DURATION);
    }

    function deployVault(uint8 decimals, uint256 depositLimit, uint48 duration) internal returns (VaultCtx memory) {
        return _deployVault(decimals, depositLimit, duration);
    }

    function _deployVault(uint8 decimals, uint256 depositLimit, uint48 duration) private returns (VaultCtx memory ctx) {
        ctx.admin = makeAddr("admin");
        ctx.rescuer = makeAddr("rescuer");
        ctx.blocklister = makeAddr("blocklister");
        ctx.pauser = makeAddr("pauser");
        ctx.limitUpdater = makeAddr("limitUpdater");
        ctx.periodConfigUpdater = makeAddr("periodConfigUpdater");
        ctx.payoutSigner = makeAddr("payoutSigner");
        ctx.payoutAllowlister = makeAddr("payoutAllowlister");
        ctx.payoutReceiver = makeAddr("payoutReceiver");

        ctx.token = new MockERC20("MockAsset", "MKA", decimals);
        ctx.periodDuration = duration;

        FirelightVault impl = new FirelightVault();
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: ctx.admin,
            limitUpdater: ctx.limitUpdater,
            blocklister: ctx.blocklister,
            pauser: ctx.pauser,
            periodConfigurationUpdater: ctx.periodConfigUpdater,
            rescuer: ctx.rescuer,
            depositLimit: depositLimit,
            periodConfigurationDuration: duration
        });
        bytes memory initData = abi.encodeCall(
            FirelightVault.initialize,
            (IERC20(address(ctx.token)), "FirelightVault", "FLV", abi.encode(params))
        );
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.vault = FirelightVault(address(proxy));

        vm.startPrank(ctx.admin);
        ctx.vault.grantRole(ctx.vault.PAYOUT_ROLE(), ctx.payoutSigner);
        ctx.vault.grantRole(ctx.vault.PAYOUT_ALLOWLIST_ROLE(), ctx.payoutAllowlister);
        vm.stopPrank();
        vm.prank(ctx.payoutAllowlister);
        ctx.vault.addToPayoutAllowlist(ctx.payoutReceiver);
    }

    function mintAndApprove(VaultCtx memory ctx, address user, uint256 amount) internal {
        ctx.token.mint(user, amount);
        vm.prank(user);
        ctx.token.approve(address(ctx.vault), amount);
    }

    // ---------------------------------------------------------------------
    // IncidentManager fixture (mirrors test/incidentManager/fixtures.js)
    // ---------------------------------------------------------------------

    uint48 public constant CAPTURE_TS = 1_700_000_000;
    uint256 public constant INCIDENT_PERIOD = 2;

    struct IncidentCtx {
        IncidentManager incidentManager;
        MockIncidentManagerCoverOrderAllocator mockAllocator;
        MockIncidentManagerVault mockVault;
        MockERC20 vaultAsset; // 18 decimals
        MockERC20 flbToken; // 6 decimals
        MockPriceFeed priceFeed; // 8 decimals, $1
        address admin;
        address curator;
        address assessmentApprover;
        address assessmentRejecter;
        address incidentInvalidator;
        address configAdmin;
        address payoutAdmin;
        address priceFeedAdmin;
        address payoutReceiver;
        address flbPayer;
        address payoutRecipient1;
        address payoutRecipient2;
        address stranger;
        bytes32 marketIdA;
        bytes32 marketIdB;
    }

    function deployIncidentManager() internal returns (IncidentCtx memory ctx) {
        // Ensure block.timestamp >= CAPTURE_TS so incidents can reference it.
        vm.warp(CAPTURE_TS + 1 days);

        ctx.admin = makeAddr("im.admin");
        ctx.curator = makeAddr("im.curator");
        ctx.assessmentApprover = makeAddr("im.approver");
        ctx.assessmentRejecter = makeAddr("im.rejecter");
        ctx.incidentInvalidator = makeAddr("im.invalidator");
        ctx.configAdmin = makeAddr("im.configAdmin");
        ctx.payoutAdmin = makeAddr("im.payoutAdmin");
        ctx.priceFeedAdmin = makeAddr("im.priceFeedAdmin");
        ctx.payoutReceiver = makeAddr("im.payoutReceiver");
        ctx.flbPayer = makeAddr("im.flbPayer");
        ctx.payoutRecipient1 = makeAddr("im.payoutRecipient1");
        ctx.payoutRecipient2 = makeAddr("im.payoutRecipient2");
        ctx.stranger = makeAddr("im.stranger");

        ctx.mockVault = new MockIncidentManagerVault();
        ctx.mockVault.setCurrentPeriod(INCIDENT_PERIOD);
        ctx.mockVault.setPeriodAtTimestamp(CAPTURE_TS, INCIDENT_PERIOD);

        ctx.vaultAsset = new MockERC20("MockVaultAsset", "mVA", 18);
        ctx.mockVault.setAsset(address(ctx.vaultAsset));
        ctx.flbToken = new MockERC20("MockUSDC", "mUSDC", 6);

        ctx.mockAllocator = new MockIncidentManagerCoverOrderAllocator();
        ctx.mockAllocator.setVault(IFirelightVault(address(ctx.mockVault)));
        ctx.mockAllocator.setCanonicalDecimals(18);
        ctx.mockAllocator.setMockCapacityConfig(
            ICoverOrderAllocator.CapacityConfig({
                minCAR: 10_000,
                firstLossBufferToken: IERC20(address(ctx.flbToken)),
                firstLossBuffer: ctx.flbPayer,
                effectiveLeverage: 20_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        );

        ctx.priceFeed = new MockPriceFeed(8, int256(10 ** 8)); // $1.00, 8 decimals
        ctx.marketIdA = keccak256("market-A");
        ctx.marketIdB = keccak256("market-B");

        IncidentManager impl = new IncidentManager();
        bytes memory initData = abi.encodeCall(
            IncidentManager.initialize,
            (
                ctx.admin,
                ctx.curator,
                ctx.assessmentApprover,
                ctx.assessmentRejecter,
                ctx.incidentInvalidator,
                ctx.configAdmin,
                ctx.payoutAdmin,
                ctx.priceFeedAdmin,
                ctx.payoutReceiver,
                ICoverOrderAllocator(address(ctx.mockAllocator)),
                IAggregatorV3(address(ctx.priceFeed)),
                3600
            )
        );
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.incidentManager = IncidentManager(address(proxy));
    }

    function fundFlb(IncidentCtx memory ctx, uint256 amount) internal {
        ctx.flbToken.mint(ctx.flbPayer, amount);
        vm.prank(ctx.flbPayer);
        ctx.flbToken.approve(address(ctx.incidentManager), amount);
    }

    function setOrderMarket(IncidentCtx memory ctx, uint256 coverTokenId, bytes32 marketId, uint256 allocated, address recipient) internal {
        ctx.mockAllocator.setOrderMarket(coverTokenId, marketId, INCIDENT_PERIOD, allocated, recipient);
    }

    function setOrderMarket(IncidentCtx memory ctx, uint256 coverTokenId, bytes32 marketId, uint256 period, uint256 allocated, address recipient) internal {
        ctx.mockAllocator.setOrderMarket(coverTokenId, marketId, period, allocated, recipient);
    }

    function createAndSubmitIncident(IncidentCtx memory ctx, uint256 lossAmount, uint256 allocatedCover) internal {
        vm.startPrank(ctx.curator);
        ctx.incidentManager.createIncident(CAPTURE_TS, "A", keccak256("a"));
        ctx.incidentManager.confirmIncident(1, "ipfs://r");
        vm.stopPrank();
        setOrderMarket(ctx, 1, ctx.marketIdA, allocatedCover, ctx.payoutRecipient1);
        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: 1, marketId: ctx.marketIdA, amount: lossAmount});
        vm.prank(ctx.curator);
        ctx.incidentManager.addAssessmentLosses(1, losses);
        vm.prank(ctx.curator);
        ctx.incidentManager.submitCurrentAssessment(1);
    }

    // ---------------------------------------------------------------------
    // CoverOrderAllocator fixture (mirrors test/setup/fixtures.js)
    // ---------------------------------------------------------------------

    struct AllocatorCtx {
        CoverOrderAllocator allocator;
        CoverNFT coverNFT;
        MockCoverOrderAllocatorVault mockVault;
        MockERC20 usdc; // premium token + FLB, 6 decimals
        MockERC20 usdt; // premium token, 6 decimals
        MockAggregatorV3 priceFeed; // 18 decimals, $1
        address deployer;
        address admin;
        address curator;
        address allocatorRole;
        address configAdmin;
        address premiumCollector;
        address flbWallet;
        address buyer1;
        address buyer2;
        bytes32 marketIdA; // Morpho / chain 1 / market-A
        bytes32 marketIdB; // Aave / chain 10 / market-B
        bytes32 concHashMorpho;
        bytes32 concHashAave;
        uint48 periodDuration;
    }

    function deployAllocator() internal returns (AllocatorCtx memory ctx) {
        ctx.deployer = address(this);
        ctx.admin = makeAddr("coa.admin");
        ctx.curator = makeAddr("coa.curator");
        ctx.allocatorRole = makeAddr("coa.allocator");
        ctx.configAdmin = makeAddr("coa.configAdmin");
        ctx.premiumCollector = makeAddr("coa.premiumCollector");
        ctx.flbWallet = makeAddr("coa.flbWallet");
        ctx.buyer1 = makeAddr("coa.buyer1");
        ctx.buyer2 = makeAddr("coa.buyer2");

        ctx.mockVault = new MockCoverOrderAllocatorVault();
        ctx.mockVault.setCurrentPeriod(1);
        ctx.mockVault.setTotalAssets(0);
        ctx.mockVault.setPeriodConfiguration(1, pc(0, 7 days, 1));
        ctx.mockVault.setPeriodConfiguration(2, pc(0, 7 days, 2));
        ctx.mockVault.setPeriodConfiguration(3, pc(0, 7 days, 3));

        ctx.usdc = new MockERC20("MockUSDC", "mUSDC", 6);
        ctx.usdt = new MockERC20("MockUSDT", "mUSDT", 6);
        ctx.mockVault.setAsset(address(ctx.usdc));

        ctx.priceFeed = new MockAggregatorV3(18, int256(10 ** 18)); // $1, 18 decimals

        // CoverNFT proxy
        CoverNFT nftImpl = new CoverNFT();
        bytes memory nftInit = abi.encodeCall(
            CoverNFT.initialize, ("Firelight Cover", "FLCOVER", "", ctx.admin, ctx.deployer, address(0), address(0))
        );
        ERC1967Proxy nftProxy = new ERC1967Proxy(address(nftImpl), nftInit);
        ctx.coverNFT = CoverNFT(address(nftProxy));

        // markets / concentrations / capacity config
        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](2);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Morpho", market: bytes32("market-A")});
        markets[1] = ICoverOrderAllocator.Market({chainId: 10, protocol: "Aave", market: bytes32("market-B")});

        ICoverOrderAllocator.ProtocolConcentrationInput[] memory concs =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](2);
        concs[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            protocol: "Morpho", chainId: 1, maxProtocolConcentrationBps: 6000
        });
        concs[1] = ICoverOrderAllocator.ProtocolConcentrationInput({
            protocol: "Aave", chainId: 10, maxProtocolConcentrationBps: 4000
        });

        address[] memory premiumTokens = new address[](2);
        premiumTokens[0] = address(ctx.usdc);
        premiumTokens[1] = address(ctx.usdt);

        ICoverOrderAllocator.InitParams memory params = ICoverOrderAllocator.InitParams({
            vault: IFirelightVault(address(ctx.mockVault)),
            premiumCollector: ctx.premiumCollector,
            coverNFT: ctx.coverNFT,
            priceFeedAdapter: IAggregatorV3(address(ctx.priceFeed)),
            maxPriceAge: 7 days,
            premiumTokens: premiumTokens,
            admin: ctx.admin,
            adminRole: ctx.admin,
            curatorRole: ctx.curator,
            allocatorRole: ctx.allocatorRole,
            configAdminRole: ctx.configAdmin,
            initialProtocolConcentrations: concs,
            newMarkets: markets,
            capacityConfig: ICoverOrderAllocator.CapacityConfig({
                minCAR: 12_000,
                firstLossBufferToken: IERC20(address(ctx.usdc)),
                firstLossBuffer: ctx.flbWallet,
                effectiveLeverage: 24_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        });

        CoverOrderAllocator impl = new CoverOrderAllocator();
        bytes memory initData = abi.encodeCall(CoverOrderAllocator.initialize, (params));
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        ctx.allocator = CoverOrderAllocator(address(proxy));

        vm.prank(ctx.admin);
        ctx.coverNFT.grantRole(keccak256("MINTER_ROLE"), address(ctx.allocator));

        ctx.marketIdA = keccak256(abi.encode(uint64(1), "Morpho", bytes32("market-A")));
        ctx.marketIdB = keccak256(abi.encode(uint64(10), "Aave", bytes32("market-B")));
        ctx.concHashMorpho = keccak256(abi.encode(uint64(1), "Morpho"));
        ctx.concHashAave = keccak256(abi.encode(uint64(10), "Aave"));
        ctx.periodDuration = 7 days;
    }

    function fundAndApprove(AllocatorCtx memory ctx, address buyer, MockERC20 token, uint256 amount) internal {
        token.mint(buyer, amount);
        vm.prank(buyer);
        token.approve(address(ctx.allocator), amount);
    }

    // ---------------------------------------------------------------------
    // Merkle helpers (OZ standard double-hash leaves, sorted-pair nodes)
    // ---------------------------------------------------------------------

    function mktLeaf(uint256 orderId, ICoverOrderAllocator.MarketCoverAllocation[] memory allocs) internal pure returns (bytes32) {
        return keccak256(bytes.concat(keccak256(abi.encode(orderId, allocs))));
    }

    /// Single-leaf tree: root == leaf, empty proof.
    function singleTree(bytes32 leaf) internal pure returns (bytes32 root, bytes32[] memory proof) {
        root = leaf;
        proof = new bytes32[](0);
    }

    /// Two-leaf tree (OZ standard merkle tree): sorted-pair root.
    function doubleTree(bytes32 leafA, bytes32 leafB)
        internal
        pure
        returns (bytes32 root, bytes32[] memory proofForA, bytes32[] memory proofForB)
    {
        root = hashPair(leafA, leafB);
        proofForA = new bytes32[](1);
        proofForA[0] = leafB;
        proofForB = new bytes32[](1);
        proofForB[0] = leafA;
    }

    function hashPair(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(bytes.concat(a, b)) : keccak256(bytes.concat(b, a));
    }

    /// ceil(cover * rateBps * duration / (BPS * 365 days)) — mirrors _calculatePremium
    function prorate(uint256 cover, uint32 rateBps, uint256 duration) internal pure returns (uint256) {
        uint256 num = cover * uint256(rateBps) * duration;
        uint256 den = BPS * SECONDS_PER_YEAR;
        return (num + den - 1) / den;
    }

    /// ceil-scale canonical 18d amount to 6d native
    function scale18to6Ceil(uint256 v) internal pure returns (uint256) {
        uint256 d = 10 ** 12;
        return (v + d - 1) / d;
    }

    function pc(uint48 epoch, uint48 duration, uint256 startingPeriod)
        private
        pure
        returns (IFirelightVault.PeriodConfiguration memory)
    {
        return IFirelightVault.PeriodConfiguration({epoch: epoch, duration: duration, startingPeriod: startingPeriod});
    }
}
```

### 4. How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready            # 42f9ea5

# solidity dependencies (the hardhat dev-deps are not needed)
npm i --ignore-scripts @openzeppelin/contracts@5.2.0
npx patch-package                      # applies patches/@openzeppelin+contracts+5.2.0.patch
npm i --ignore-scripts @openzeppelin/contracts-upgradeable@5.4.0
git clone --depth 1 -b v1.9.4 https://github.com/foundry-rs/forge-std lib/forge-std

# add the four files from sections 1-3 at the paths shown above

forge test --match-path "test/foundry/F10*" -vv
```

### 5. Expected output

```
Ran 1 test for test/foundry/F10_RescueClaimLock.t.sol:F10_RescueClaimLockTest
[PASS] test_rescuePrematureClaimFlagLocksFunds() (gas: 7287523)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

### 6. Key assertions

The rescue leaves the account with zero shares but a terminal flag on a period that has not started:

```solidity
vault.rescueWithdrawFromBlocklisted(alice, bob, _arr(targetPeriod));

assertEq(vault.withdrawSharesOf(targetPeriod, alice), 0);
assertEq(vault.withdrawSharesOf(targetPeriod, bob), 4_000e6);
assertTrue(vault.isWithdrawClaimed(targetPeriod, alice));      // <-- premature
```

After an ordinary un-blocklist, a fresh request lands in the same period and inherits the flag:

```solidity
vault.removeFromBlocklist(alice);
vault.withdraw(6_000e6, alice, alice);

assertEq(vault.withdrawSharesOf(targetPeriod, alice), 6_000e6); // positive entitlement
assertTrue(vault.isWithdrawClaimed(targetPeriod, alice));       // under a set claim flag
```

Both exits are closed, permanently:

```solidity
vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
vault.claimWithdraw(targetPeriod);                              // Alice cannot claim

vault.addToBlocklist(alice);
vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
vault.rescueWithdrawFromBlocklisted(alice, bob, _arr(targetPeriod));   // nor can the rescuer
```

Bob's rescued entitlement still settles normally, which isolates the failure to Alice's new reservation rather than to the rescue as a whole:

```solidity
vault.claimWithdraw(targetPeriod);                              // Bob: succeeds
assertEq(token.balanceOf(bob), 4_000e6);
assertEq(vault.pendingWithdrawAssets(), 6_000e6);               // Alice's, stranded forever
```
