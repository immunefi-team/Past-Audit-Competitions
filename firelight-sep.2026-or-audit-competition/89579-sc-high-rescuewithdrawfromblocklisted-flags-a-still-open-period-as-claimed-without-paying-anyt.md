# 89579 sc high rescuewithdrawfromblocklisted flags a still open period as claimed without paying anything out permanently bricking the rescued user s next legitimate withdrawal into that same period

**Submitted on Aug 19th 2026 at 23:50:07 UTC by @MangalJup for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89579
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

`rescueWithdrawFromBlocklisted` moves a blocklisted user's pending withdrawal to a clean address, but also sets `isWithdrawClaimed[period][from] = true` even though `from` was paid nothing — and it does this without checking whether `period` is still open. Withdrawal requests always book into the *pending* period (`currentPeriod() + 1`) and add to the same slot with `+=`, so once the rescued user is un-blocklisted (a routine false-positive resolution) and books a new, ordinary withdrawal before that period rolls over, the new claim lands on the already-flagged slot. `claimWithdraw` checks the flag before it ever looks at the balance, so that claim reverts forever — and the rescue function itself refuses to touch the same `(period, account)` a second time, closing off the one role that could otherwise fix it.

### Vulnerability Details

The rescue marks the source account "claimed" with no payout and no period check:

```solidity
// contracts/core/FirelightVault.sol:861-873
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // :864
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;   // :870 -- no payout, no `periods[i] < currentPeriod()` check
    ...
}
```

The flagged slot is not single-use — every new withdrawal request adds into the pending period:

```solidity
// contracts/core/FirelightVault.sol:1008-1019
uint256 period = currentPeriod() + 1;   // :1008 -- always the next, still-open period
...
    withdrawSharesOf[period][receiver] += sharesWithdraw;   // :1017 -- additive, not single-assignment
    pendingWithdrawAssets += assets;
```

`claimWithdraw` consults the flag before the balance:

```solidity
// contracts/core/FirelightVault.sol:794
    if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // fires even though a real, funded claim now sits in this slot
```

`isWithdrawClaimed` has exactly two writers in the whole contract — `claimWithdraw` and the rescue — both only ever write `true`. There is no resetter, and a second rescue of the same `(period, from)` reverts on the rescue's own guard at `:864`, so `RESCUER_ROLE` cannot move the poisoned entry out either.

### Impact Details

* Alice's second, fully paid-for withdrawal is unclaimable by her, unrescuable by `RESCUER_ROLE`, and permanently excluded from `totalAssets()`.
* Only precondition: the compliance team un-blocklists the user before the pending period rolls over — an ordinary review turnaround, not a narrow race.
* No malicious input required anywhere: blocklist, rescue, and un-blocklist are all exercised exactly within their documented authority; the loss is realized by the victim's own ordinary `redeem()`. Exact figures from the PoC below.

### Mitigation

* Settle by balance, not by flag: zero `withdrawSharesOf[period][sender]` on claim and drop the `isWithdrawClaimed` write entirely — the zeroing already fully consumes a rescued or claimed entry, making the boolean redundant and the sole source of the bug.
* If keeping the flag, at minimum refuse to rescue a still-open period: `if (periods[i] >= currentPeriod()) revert InvalidPeriod();` — smaller diff, but blocks rescuing the common case (an outstanding withdrawal) until it settles.

### References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L876>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L811>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028>

## Proof of Concept

## Proof of Concept

One honest user, one routine compliance rescue, no malicious actor. Save as `test/RescueBricksClaim.t.sol` and run `forge test --match-contract RescueBricksClaimReportPoC -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";

contract RBCToken is ERC20 {
    constructor() ERC20("Mock fXRP", "fXRP") {}
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

contract RescueBricksClaimReportPoC is Test {
    address internal constant ADMIN = address(0xA11CE);
    address internal constant BLOCKLISTER = address(0xB10C);
    address internal constant RESCUER = address(0x2E5C);
    address internal constant ALICE = address(0xA71CE);      // honest user, blocklisted by mistake
    address internal constant ALICE_NEW = address(0xA71CE2); // compliant rescue destination

    FirelightVault internal vault;
    RBCToken internal token;

    function setUp() public {
        vm.warp(1_800_000_000);
        token = new RBCToken();

        FirelightVault impl = new FirelightVault();
        IFirelightVault.InitParams memory p = IFirelightVault.InitParams({
            defaultAdmin: ADMIN, limitUpdater: ADMIN, blocklister: BLOCKLISTER, pauser: ADMIN,
            periodConfigurationUpdater: ADMIN, rescuer: RESCUER,
            depositLimit: type(uint128).max, periodConfigurationDuration: 7 days
        });
        vault = FirelightVault(address(new ERC1967Proxy(address(impl),
            abi.encodeCall(FirelightVault.initialize,
                (IERC20(address(token)), "Firelight Vault", "fvFXRP", abi.encode(p))))));

        token.mint(ALICE, 200_000e6);
        vm.prank(ALICE);
        token.approve(address(vault), type(uint256).max);
    }

    function test_rescueOfOpenPeriodBricksAlicesNextWithdrawal() public {
        // -- Alice deposits and requests a withdrawal; it books into the pending period P --
        vm.startPrank(ALICE);
        vault.deposit(100_000e6, ALICE);
        uint256 P = vault.currentPeriod() + 1;
        vault.redeem(vault.balanceOf(ALICE), ALICE, ALICE);
        vm.stopPrank();

        // -- compliance blocklists Alice; RESCUER_ROLE moves her PENDING claim to a clean
        // address -- exactly the documented use of the function --
        vm.prank(BLOCKLISTER);
        vault.addToBlocklist(ALICE);

        uint256[] memory periods = new uint256[](1);
        periods[0] = P;
        vm.prank(RESCUER);
        vault.rescueWithdrawFromBlocklisted(ALICE, ALICE_NEW, periods);

        // FirelightVault.sol:870 -- Alice is flagged "claimed" though nothing was paid to her.
        assertTrue(vault.isWithdrawClaimed(P, ALICE), "alice flagged claimed by the rescue");

        // -- false positive resolved, still inside period P-1: Alice resumes ordinary use --
        vm.prank(BLOCKLISTER);
        vault.removeFromBlocklist(ALICE);
        assertEq(vault.currentPeriod() + 1, P, "still inside P-1: new requests land in P");

        vm.startPrank(ALICE);
        vault.deposit(50_000e6, ALICE);
        uint256 stranded = vault.redeem(vault.balanceOf(ALICE), ALICE, ALICE);
        vm.stopPrank();

        // brand-new, fully paid-for claim right lands on the already-flagged slot (:1017 is `+=`)
        assertGt(vault.withdrawSharesOf(P, ALICE), 0, "new entry booked for alice");
        assertTrue(vault.isWithdrawClaimed(P, ALICE), "...but the slot is still flagged claimed");

        // -- two period rolls; P becomes claimable --
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
        vm.warp(uint256(vault.currentPeriodEnd()) + 1);
assertGt(vault.currentPeriod(), P);

        // -- the rescue destination is paid normally; the rescue itself worked as documented --
        vm.prank(ALICE_NEW);
        vault.claimWithdraw(P);

        // -- Alice can NEVER claim her second, legitimate withdrawal --
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), P));
        vault.claimWithdraw(P);

        // -- and not even RESCUER_ROLE can move it out --
        vm.prank(BLOCKLISTER);
        vault.addToBlocklist(ALICE);
        vm.prank(RESCUER);
        vm.expectRevert(abi.encodeWithSelector(bytes4(keccak256("AlreadyClaimedPeriod(uint256)")), P));
        vault.rescueWithdrawFromBlocklisted(ALICE, ALICE_NEW, periods);

        // -- the stranded amount is permanently excluded from totalAssets() --
        assertEq(vault.pendingWithdrawAssets(), stranded, "stranded amount stuck in pendingWithdrawAssets");
        assertEq(token.balanceOf(address(vault)) - vault.totalAssets(), stranded, "totalAssets() under-reports by the stranded amount");

        console2.log("assets Alice can never claim  =", stranded);
        console2.log("vault raw balance              =", token.balanceOf(address(vault)));
        console2.log("vault totalAssets()            =", vault.totalAssets());
        console2.log("pendingWithdrawAssets stuck    =", vault.pendingWithdrawAssets());
    }
}
```

Result (actually run — `forge test --match-contract RescueBricksClaimReportPoC -vv`):

```
Ran 1 test for audit-data/foundry/RescueBricksClaimReportPoC.t.sol:RescueBricksClaimReportPoC
[PASS] test_rescueOfOpenPeriodBricksAlicesNextWithdrawal() (gas: 733499)
Logs:
  assets Alice can never claim  = 50000000000
  vault raw balance              = 50000000000
  vault totalAssets()            = 0
  pendingWithdrawAssets stuck    = 50000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Alice's `50,000e6` fXRP is unclaimable by her, unrescuable by RESCUER\_ROLE, and permanently excluded from totalAssets() — the vault still physically holds it but reports zero assets.
