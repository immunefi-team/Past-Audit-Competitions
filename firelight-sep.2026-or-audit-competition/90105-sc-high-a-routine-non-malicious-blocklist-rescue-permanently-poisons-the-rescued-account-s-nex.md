# 90105 sc high a routine non malicious blocklist rescue permanently poisons the rescued account s next withdrawal with no recovery path for any role

**Submitted on Aug 23rd 2026 at 02:32:29 UTC by @BalkarSingh for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90105
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` exists to move a blocklisted account's pending withdrawal claim to a clean address. It does that part correctly — but as a side effect it also permanently marks the *source* account's `(period, account)` slot as "claimed," even though the source account was paid nothing, and even though nothing requires the rescued period to have already settled. Withdrawal requests always book into whichever period is currently pending, and the storage slot they book into is additive rather than single-assignment. So the moment the rescued user is cleared (an unremarkable, expected outcome of a compliance review) and makes one more ordinary withdrawal before the period rolls over, that brand-new, fully-paid-for claim lands directly on top of the stale "claimed" flag — and is rejected forever. The one role built to correct rescue mistakes cannot touch the slot again either, because it treats "already flagged" as "already handled."

### Vulnerability Details

`rescueWithdrawFromBlocklisted` unconditionally stamps the source account's slot as settled, with no payout and no requirement that the period already be closed:

```solidity
// contracts/core/FirelightVault.sol:845-876
function rescueWithdrawFromBlocklisted(
    address from,
    address to,
    uint256[] calldata periods
)
    external
    onlyRole(RESCUER_ROLE)
    onlyBlocklisted(from)
    notBlocklisted(to)
{
    ...
    for (uint256 i = 0; i < len; i++) {
        uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

        if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
        if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
        if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

        withdrawSharesOf[periods[i]][to] += _withdrawOf;
        withdrawSharesOf[periods[i]][from] = 0;
@>      isWithdrawClaimed[periods[i]][from] = true; 
        ...
    }
}
```

The slot this writes to is not single-use. Every ordinary withdrawal request targets the currently-pending period and *adds into* whatever is already sitting in that account's slot:

```solidity
// contracts/core/FirelightVault.sol:994-1028 (_requestWithdraw)
uint256 period = currentPeriod() + 1;   // :1008 -- always the next, still-open period, never chosen by the caller
...
@>  withdrawSharesOf[period][receiver] += sharesWithdraw;
    pendingWithdrawAssets += assets;
```

And the claim path checks the stale flag before it ever looks at whether there's a real, funded balance behind it:

```solidity
// contracts/core/FirelightVault.sol:788-797 (claimWithdraw)
if (period >= currentPeriod()) revert InvalidPeriod();
address sender = _msgSender();
@> if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

`isWithdrawClaimed` has exactly two writers anywhere in the contract — `claimWithdraw:806` and the rescue's `:870` — and both only ever write `true`. Nothing resets it. Attempting a second rescue of the same `(period, from)` to work around this doesn't help either: it hits the rescue function's own duplicate guard (`:864`) and reverts, so `RESCUER_ROLE` has no path to un-poison the slot it created.

### Impact Details

* The victim's second, fully-funded withdrawal becomes permanently unclaimable — by the victim, and by `RESCUER_ROLE` itself.
* Trigger requires nothing adversarial: (1) compliance blocklists an account with an open withdrawal in the pending period, which is a routine, false-positive-prone screening action; (2) `RESCUER_ROLE` rescues that one pending claim, exactly as the function is meant to be used; (3) compliance clears the account before the period rolls over — an ordinary review turnaround, not a race; (4) the cleared account, with zero special knowledge, simply uses the vault again. Every step is either a role acting exactly inside its documented job, or an unprivileged user doing nothing unusual.
* Beyond the direct victim, the stranded assets stay counted in `pendingWithdrawAssets` forever, which `totalAssets()` (`:427-429`) subtracts from the vault's real balance — so *every other LP's* share price is permanently understated by the stranded amount too, even LPs who never touched blocklist or rescue at all. Exact figures, including the bystander-LP effect, are in the PoC below.

### References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol>

## Proof of Concept

## Proof of Concept

Below test is written in foundry

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";

/// @dev Minimal 6-decimal mock of the staked asset (fXRP), matching production decimals.
contract MockStakedFXRP is ERC20 {
    constructor() ERC20("Mock Staked FXRP", "mfXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

/// PoC — isWithdrawClaimed poisoning across a compliance rescue cycle
contract RescuePoisoningPoC is Test {
    address internal constant GOVERNOR = address(0x900D);
    address internal constant COMPLIANCE = address(0xC0111A5); // BLOCKLIST_ROLE
    address internal constant RECOVERY = address(0x2EC0);      // RESCUER_ROLE

    address internal constant FLAGGED_USER = address(0xF1A6);      // wrongly blocklisted, later cleared
    address internal constant RECOVERY_WALLET = address(0xF1A6C1); // rescue destination
    address internal constant BYSTANDER_LP = address(0xB1A);       // never touches the rescue at all

    FirelightVault internal vault;
    MockStakedFXRP internal asset;

    function setUp() public {
        vm.warp(1_800_000_000);
        asset = new MockStakedFXRP();

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory params = IFirelightVault.InitParams({
            defaultAdmin: GOVERNOR,
            limitUpdater: GOVERNOR,
            blocklister: COMPLIANCE,
            pauser: GOVERNOR,
            periodConfigurationUpdater: GOVERNOR,
            rescuer: RECOVERY,
            depositLimit: type(uint128).max,
            periodConfigurationDuration: 7 days
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(impl),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(asset)), "Firelight Vault", "fvFXRP", abi.encode(params))
                    )
                )
            )
        );

        asset.mint(FLAGGED_USER, 200_000e6);
        asset.mint(BYSTANDER_LP, 100_000e6);

        vm.prank(FLAGGED_USER);
        asset.approve(address(vault), type(uint256).max);
        vm.prank(BYSTANDER_LP);
        asset.approve(address(vault), type(uint256).max);
    }

    function _seedBystanderLP() internal {
        vm.prank(BYSTANDER_LP);
        vault.deposit(100_000e6, BYSTANDER_LP);
    }

    function test_complianceRescueOfOpenPeriodPermanentlyBricksVictimsNextWithdrawal() public {
        _seedBystanderLP();

        // Step 1 -- an ordinary user deposits and requests a withdrawal. It books into
        // the pending (still-open) period P (FirelightVault.sol:1008).
        vm.startPrank(FLAGGED_USER);
        vault.deposit(100_000e6, FLAGGED_USER);
        uint256 pendingPeriod = vault.currentPeriod() + 1;
        vault.redeem(vault.balanceOf(FLAGGED_USER), FLAGGED_USER, FLAGGED_USER);
        vm.stopPrank();

        // Step 2 -- compliance flags the account and RESCUER_ROLE moves the pending
        // claim to a clean wallet -- exactly the function's documented purpose.
        vm.prank(COMPLIANCE);
        vault.addToBlocklist(FLAGGED_USER);

        uint256[] memory targetPeriods = new uint256[](1);
        targetPeriods[0] = pendingPeriod;
        vm.prank(RECOVERY);
        vault.rescueWithdrawFromBlocklisted(FLAGGED_USER, RECOVERY_WALLET, targetPeriods);

        // FirelightVault.sol:870 -- the source slot is flagged claimed though nothing
        // was paid to it.
        assertTrue(
            vault.isWithdrawClaimed(pendingPeriod,
FLAGGED_USER),
            "rescue stamps the source slot claimed with zero payout"
        );

        // Step 3 -- compliance clears the false positive before the pending period
        // rolls over. Routine resolution, not an edge case.
        vm.prank(COMPLIANCE);
        vault.removeFromBlocklist(FLAGGED_USER);
        assertEq(
            vault.currentPeriod() + 1,
            pendingPeriod,
            "still inside the same period window as the rescue"
        );

        // Step 4 -- the now-cleared user does nothing wrong: an ordinary second
        // deposit + withdraw. FirelightVault.sol:1017 adds into the SAME slot, so the
        // fresh, fully-funded claim lands directly on the poisoned flag.
        vm.startPrank(FLAGGED_USER);
        vault.deposit(50_000e6, FLAGGED_USER);
        uint256 strandedAssets = vault.redeem(vault.balanceOf(FLAGGED_USER), FLAGGED_USER, FLAGGED_USER);
        vm.stopPrank();

        assertGt(vault.withdrawSharesOf(pendingPeriod, FLAGGED_USER), 0, "new, real claim booked");
        assertTrue(vault.isWithdrawClaimed(pendingPeriod, FLAGGED_USER), "...on an already-poisoned slot");

        // Two period rolls so `pendingPeriod` becomes claimable.
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
        assertGt(vault.currentPeriod(), pendingPeriod);

        // The rescue destination claims fine -- the rescue worked exactly as
        // documented for `to`.
        vm.prank(RECOVERY_WALLET);
        vault.claimWithdraw(pendingPeriod);

        // The victim's own second, fully-funded withdrawal is permanently unclaimable.
        vm.prank(FLAGGED_USER);
        vm.expectRevert(
            abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), pendingPeriod)
        );
        vault.claimWithdraw(pendingPeriod);

        // Not even RESCUER_ROLE can pull the poisoned slot back out -- a second
        // rescue attempt hits the same "already claimed" guard.
        vm.prank(COMPLIANCE);
        vault.addToBlocklist(FLAGGED_USER);
        vm.prank(RECOVERY);
        vm.expectRevert(
            abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), pendingPeriod)
        );
        vault.rescueWithdrawFromBlocklisted(FLAGGED_USER, RECOVERY_WALLET, targetPeriods);

        // The stranded principal stays inside pendingWithdrawAssets forever, so
        // totalAssets() under-reports for BYSTANDER_LP too, though it never touched
        // blocklist/rescue at all.
        assertEq(
            vault.pendingWithdrawAssets(),
            strandedAssets,
            "stranded principal stuck in pendingWithdrawAssets"
        );
        assertEq(
            asset.balanceOf(address(vault)) - vault.totalAssets(),
            strandedAssets,
            "raw balance vs. reported totalAssets() diverges by exactly the stranded amount"
        );

        console2.log("victim's unclaimable principal          =", strandedAssets);
        console2.log("vault raw asset balance                 =", asset.balanceOf(address(vault)));
        console2.log("vault totalAssets() (under-reported)    =", vault.totalAssets());
        console2.log("pendingWithdrawAssets permanently stuck =", vault.pendingWithdrawAssets());
    }
}
```

```bash
[PASS] test_complianceRescueOfOpenPeriodPermanentlyBricksVictimsNextWithdrawal() (gas: 869291)
```
