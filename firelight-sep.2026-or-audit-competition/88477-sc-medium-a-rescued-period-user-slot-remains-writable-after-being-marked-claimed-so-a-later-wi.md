# 88477 sc medium a rescued period user slot remains writable after being marked claimed so a later withdrawal burns the user s shares into a position no one can ever claim

**Submitted on Aug 13th 2026 at 11:51:02 UTC by @zaaf for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88477
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

**Asset:** `contracts/core/FirelightVault.sol` (in-scope asset #1) **Pinned commit:** `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (branch `v1_audit_ready`) **Lines:** `845-876` (`rescueWithdrawFromBlocklisted`), offending write at **line 870**; interacts with `788-811` (`claimWithdraw`, guard at line 794), `994-1028` (`_requestWithdraw`, lines 1008 and 1017) and `560` (`removeFromBlocklist`)

## Summary

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` on the **source** address. `isWithdrawClaimed[period][account]` has **no clearing write anywhere in the in-scope contract**: it is written in exactly two places, `claimWithdraw` (line 806) and this rescue (line 870), and **both write only `true`**. Once set, the `(period, account)` claim slot is closed for the lifetime of that contract state.

The blocklist that justifies the rescue is **not** permanent (`removeFromBlocklist`, line 560), and a withdrawal request lands in `currentPeriod() + 1` (line 1008), so the rescued period stays open for new requests for the remainder of the current period.

If an account is cleared from the blocklist and requests a withdrawal to itself before the period rolls over, the second request **succeeds**: it burns the user's remaining shares and increases `pendingWithdrawAssets`. The resulting withdrawal can never pass `claimWithdraw`, because the same `(period, receiver)` pair was permanently marked claimed by the earlier rescue.

## Root cause

The rescue only creates the poisoned slot. The defect that turns it into a loss is on the request side, and it is a missing invariant:

> **A withdrawal request must never be accepted into `(period, receiver)` while `isWithdrawClaimed[period][receiver] == true`.**

`_requestWithdraw` does not check this. The only reads of `isWithdrawClaimed` in the entire contract are at lines 794, 864 and 865, and none of them is inside it. It credits `withdrawSharesOf[period][receiver]` and burns the shares without ever consulting the flag that `claimWithdraw` will later use to refuse payment. The vault therefore accepts and finalizes the accounting for a request whose resulting claim slot is already permanently closed.

## The project has already fixed the mirror image of this collision in the same function

Commit **`a2def47`**, *"Fix rescuing withdrawal to a user that already claimed them"* (2025-07-11), added this guard to this same function:

```solidity
// FirelightVault.sol:865
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
```

That commit establishes that **a latch collision in this function is a class of bug the project recognises and patches**. It closed the collision on the **beneficiary** side. The **source** side, the write on line 870, creates the same class of collision through a different state transition, where `from` later becomes the receiver of an ordinary request, and was not considered.

## Mechanism

```solidity
// FirelightVault.sol:862-872
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0)                    revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;   // monotonic: never cleared anywhere
```

Three facts combine, each in a different function and none suspicious alone: line 870 is a one-way latch, the blocklist at line 560 is reversible, and line 1008 keeps the rescued period writable for the rest of the current period.

## Recovery attempts after the loss exists

Every row below was executed against the unmodified pinned contract, and every row is paired with a **control**: Erin holds a byte-identical position, the same 600e18 credited to the same period 1 slot, differing only in that her slot was never rescued. **Every call that reverts for Alice succeeds for Erin in the same sequence**, so these are decisions by the contract and not a broken fixture.

| Recovery attempt                                                        | Result                                                                                                    | Control                        |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------ |
| Alice claims the original slot                                          | `AlreadyClaimedPeriod(1)`                                                                                 | Erin paid 600e18               |
| Re-blocklist Alice, rescuer moves the withdrawal to Dave                | `AlreadyClaimedPeriod(1)` at line 864, **before any transfer**; Alice's slot stays 600e18, Dave's stays 0 | `rescue(erin→dave)` succeeded  |
| Alice creates another withdrawal to a different receiver                | Alice's share balance is exactly **0**, burned by the second request → `InsufficientShares`               | Bob's identical call succeeded |
| Alice is funded again and withdraws to Dave                             | Dave receives exactly the **new** 100e18; not one wei of the stranded 600e18 moves                        | —                              |
| Wait 12 further periods                                                 | latch unchanged                                                                                           | Erin paid 600e18               |
| Five blocklist add and remove cycles                                    | latch unchanged                                                                                           | Erin paid 600e18               |
| `rescueSharesFromBlocklisted`                                           | `InsufficientShares`, share balance 0                                                                     | Bob to Frank succeeded         |
| Full admin surface, third-party call, allowance, period reconfiguration | latch unchanged; `NoWithdrawalAmount` — there is no owner argument, no forwarder and no sweep             | Erin paid 600e18               |
| Donate a healthy slot **into** Alice                                    | `AlreadyClaimedPeriod` — line 865 latches her out as a **destination** as well                            | Erin to Frank succeeded        |

State after the sequence: `withdrawSharesOf[1][alice] = 600000000000000000000`, still credited and unchanged; `pendingWithdrawAssets` does not decrease for Alice, with the residual asserted as `pendingAfterRequeue − carolClaimed − erinClaimed == 600000000000000000000`.

**`payout()` is the one honest complication and it is not a recovery path**, on four executed grounds: it pays an allowlisted incident payee while Alice's balance stays 0; it does not touch her slot; it draws **pro rata**, so it actively harms the innocent claimant (Erin, owed 600e18, received `499999999999999999999`); and the window closes with `InvalidCapturePeriod`.

### Write-site inventory

* **`isWithdrawClaimed`** — exactly two writes, line 806 (`claimWithdraw`) and line 870 (this rescue). **Both write `true` only. There is no `false` write anywhere in the contract.** Monotonic by construction.
* **`withdrawSharesOf`** — three writes; the rescue is blocked in both directions once either side is latched, and `_requestWithdraw` can only ever write `currentPeriod() + 1`.
* **`pendingWithdrawAssets`** — three writes. There is no emergency function, no sweep, no `rescueERC20` and no owner withdrawal.

## Impact

After the second request has executed, **no user- or rescuer-reachable path in the in-scope contract can claim or relocate the 600e18 withdrawal**. The user's shares are burned, the underlying is reserved in `pendingWithdrawAssets`, and the claim slot is permanently closed.

The value is not redistributed either: `pendingWithdrawAssets` is subtracted from `totalAssets()` (line 428), so Bob's 1000e18 of shares are still worth 1000e18. The vault holds 2000e18 while reporting 1400e18.

The exposure window is the remainder of one period, which is configurable (`SMALLEST_PERIOD_DURATION` is one day; the fixtures use seven).

## What I am not claiming

* **This is irreversibility, not likelihood.** The sequence requires `BLOCKLIST_ROLE` and `RESCUER_ROLE` actions and a `removeFromBlocklist` inside the same period. **No privileged role behaves maliciously or exceeds its intended authority**: the administrative steps are the ordinary compliance lifecycle, and the loss condition is triggered afterwards by an ordinary user withdrawal request that the contract accepts successfully. This is not a report about what a privileged role can do; it is a state-invariant violation left behind by the intended workflow. I am not claiming a third party can cause the sequence.
* **The vault is not upgradeable by itself, but it sits behind a proxy, and I am stating that plainly rather than leaving it for you to find.** `FirelightVault` does not inherit `UUPSUpgradeable`, has no `_authorizeUpgrade`, and its ABI contains zero upgrade functions. It is deployed behind a `TransparentUpgradeableProxy`: reading the ERC-1967 admin slot gives a `ProxyAdmin` at `0x61c3…33e1`. **A governance upgrade could therefore add a bespoke recovery function.** None exists today, and no function in the current implementation can un-strand the slot.
* Naming a different receiver on the second request avoids the condition entirely, because `_requestWithdraw` credits the **receiver** rather than the owner. That is a way to **avoid** the bug, available only **before** the request is made. Measured at maximum strength in the table above: once the loss exists, funding Alice again and withdrawing to Dave moves the new amount and none of the stranded amount.
* I did not find a path that strands anything without `removeFromBlocklist` being called. Both `withdraw` and `redeem` carry `notBlocklisted(owner)`, so the source must be cleared first.
* I did not find a path where the re-queue lands in a different period from the rescue. If the operator clears the account after the period has rolled over, nothing is lost.
* None of the three published Known Issues involves the withdrawal queue, the blocklist rescue, `isWithdrawClaimed`, or the acceptance of a new request into an already-claimed `(period, receiver)` slot. No price feed, peg, first-loss buffer, claim or cover accounting is reached by this path at all.

## Prior art — including two related published findings I want to put in front of you myself

**This class has been found twice before on this codebase, by OpenZeppelin, and both entry points were closed individually. The invariant was never added, and the rescue path is a third entry point that is still open at the pinned commit.** I would rather state that plainly than have it discovered.

**OpenZeppelin, `2025-07-21_OpenZeppelin_Firelight.pdf`, M-02 "Period Duration Cannot Be Changed"** describes the same *loss mechanism* through a different trigger. Its own words:

> "If a user had requested a withdrawal in a previous period and, after the `periodDuration` change, the recalculated `currentPeriod` still matches the earlier request period, the user may attempt to withdraw again. In such cases, their shares would be burned during the call to `_requestWithdraw`, but the subsequent call to `claimWithdraw` would revert due to the `isWithdrawClaimed` check."

Its trigger is a `periodDuration` change; the recommendation was to keep `periodDuration` constant, and it is marked **Resolved in pull request #6 at commit 1433252**.

**M-03 "Missing Blacklist Check Can Lock Funds"** is the same class again, through a third route: a withdrawal to a blacklisted receiver becomes irretrievable because `claimWithdraw` fails on the blacklist check. Marked **Resolved in pull request #5 at commit 68a9dd2**, by validating the receiver.

**What that history shows, and why this report is not a re-report of either.** Both fixes closed a specific way of reaching an unclaimable slot. Neither added the property that connects them:

> a request must never be accepted into `(period, receiver)` while `isWithdrawClaimed[period][receiver] == true`.

The rescue path reaches the same state by a route neither finding covers. `rescueWithdrawFromBlocklisted` did not exist in the audited scope of M-02, the trigger here is not a `periodDuration` change, the receiver is not blacklisted at the time of the request (`removeFromBlocklist` has already been called, and it must have been, since `redeem` carries `notBlocklisted(owner)`), and the defect is live at `42f9ea5e…` while both cited findings are resolved. **The recommended fix in this report is precisely the generalization the two previous fixes stopped short of.**

**Other corpora searched, with results:**

* **0xMacro `Firelight A-1`, 29 July 2026** — searched for `rescue`, `blocklist`, `isWithdrawClaimed`, `AlreadyClaimedPeriod` and `withdrawSharesOf`: **no matching finding**. Its lower-severity findings are M-1 leverage, M-2 `batchSettle` denial of service, L-1 zero-share deposits, L-2 concentration caps, L-3 grace period and L-4 CAR floor, none of which touch the rescue path.
* **`2025-09-01_Coinspect_Firelight_Frontend.pdf`** — frontend scope; no occurrence of any of the five terms.
* **The November 2025 Firelight Audit Competition** (191 participating researchers; 6 Low and 40 Insight reports awarded from a $15,000 pool). Its published reports contain **one** entry touching this function, `#59288`, which is a **gas-optimisation note** about repeated array access in the `rescueWithdrawFromBlocklisted` loop. Its proof-of-concept ships a `reset()` helper that flips `isWithdrawClaimed` back, i.e. its author treated the flag as ordinary mutable state rather than as a latch.
* **Repository surfaces**: `immunefi-team/audit-comp-firelight` carries three refs in total and has issues and pull requests disabled. The upstream `firelight-protocol/firelight-core` still contains the identical unguarded code at the time of writing. A grep of the in-scope contracts for bounty or audit identifiers and for `must never` / `do not remove` style remediation comments returns nothing, against a control query on `isWithdrawClaimed` that returns five rows.

## Suggested remediation

Both changes were applied to the pinned code and the project's own suite was re-run. **The baseline is 448 tests passing**, and both remediations hold it at **448 with zero regressions**.

**Primary — remove the source latch.** Line 870 adds no protection the function does not already provide: line 869 zeroes `withdrawSharesOf[periods[i]][from]`, and with the slot at zero a second rescue already reverts `NoWithdrawalAmount` while a double claim is already impossible.

```diff
- isWithdrawClaimed[periods[i]][from] = true;
```

Measured effect: Alice re-queues and **recovers her own 600e18**; double rescue and double claim both remain blocked.

**Defence in depth — enforce the invariant at the request site.**

```solidity
if (isWithdrawClaimed[period][receiver]) revert AlreadyClaimedPeriod(period);
```

This guarantees that **any** future code path which closes a claim slot cannot then receive a withdrawal into it. It is structurally safe: `claimWithdraw` only latches periods strictly below `currentPeriod()`, while requests target `currentPeriod() + 1`, so the guard is unreachable except on a rescue-poisoned slot.

**Apply the removal as the fix and the guard as defence in depth, in that order.** The guard alone prevents the loss but leaves a residue: a cleared user would be unable to withdraw to herself for the rest of the period, turning a silent loss into a hard block. Removing the latch eliminates the poisoned slot altogether, after which the guard is provably unreachable.

**Neither change un-strands slots that have already been poisoned.** Those would need a migration through the proxy's upgrade path.

## Proof of Concept

## Proof of concept

```
forge test --match-test "test_LEAD1_rescueThenRequeueSamePeriodLocksAssetsForever" -vv
```

Sequence, using only ordinary intended operations and no attacker:

1. Alice deposits 1000e18, Bob deposits 1000e18.
2. Alice redeems 400e18, queued into period 1.
3. Compliance blocklists Alice; `RESCUER_ROLE` rescues period 1 to Carol.
4. Alice is cleared with `removeFromBlocklist(alice)`.
5. Still inside the same period, Alice redeems her remaining 600e18 to herself, queued into period 1.
6. Two periods pass. Carol claims 400e18. Alice's `claimWithdraw(1)` reverts.

Output:

```
Ran 1 test for forge-test/VaultScenarios.t.sol:VaultScenarios
[PASS] test_LEAD1_rescueThenRequeueSamePeriodLocksAssetsForever() (gas: 888430)
Logs:
  alice re-queued assets: 600000000000000000000
  pendingWithdrawAssets grew by: 600000000000000000000
  carol claimed: 400000000000000000000
  assets stranded in pendingWithdrawAssets: 600000000000000000000
  vault token balance                             : 1600000000000000000000
  vault totalAssets() (balance - pending)         : 1000000000000000000000
```

The lockout is asserted, not inferred:

```solidity
vm.prank(alice);
vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, period));
vault.claimWithdraw(period);
```

### The test, in full

```solidity
function test_LEAD1_rescueThenRequeueSamePeriodLocksAssetsForever() public {
        _deposit(alice, 1_000e18);
        _deposit(bob, 1_000e18);

        // Alice queues a withdrawal into period 1.
        vm.prank(alice);
        vault.redeem(400e18, alice, alice);
        uint256 period = vault.currentPeriod() + 1;

        // Alice is blocklisted; the rescuer moves her queued withdrawal to Carol.
        vm.startPrank(admin);
        vault.addToBlocklist(alice);
        uint256[] memory ps = new uint256[](1);
        ps[0] = period;
        vault.rescueWithdrawFromBlocklisted(alice, carol, ps);
        assertTrue(vault.isWithdrawClaimed(period, alice), "rescue should mark from as claimed");
        // Alice is later cleared.
        vault.removeFromBlocklist(alice);
        vm.stopPrank();

        // Still the same period. Alice queues a NEW withdrawal with her remaining shares.
        uint256 pendingBefore = vault.pendingWithdrawAssets();
        vm.prank(alice);
        uint256 requeued = vault.redeem(600e18, alice, alice);
        assertEq(vault.currentPeriod() + 1, period, "must land in the same period");
        assertGt(requeued, 0, "POSITIVE CONTROL FAILED: requeue moved 0 assets");
        console2.log("alice re-queued assets:", requeued);
        console2.log("pendingWithdrawAssets grew by:", vault.pendingWithdrawAssets() - pendingBefore);

        _nextPeriod();
        _nextPeriod();

        // Carol can claim her rescued portion.
        vm.prank(carol);
        uint256 carolGot = vault.claimWithdraw(period);
        console2.log("carol claimed:", carolGot);
        assertGt(carolGot, 0, "carol should be paid");

        // Alice cannot claim anything, ever.
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, period));
        vault.claimWithdraw(period);

        uint256 stranded = vault.pendingWithdrawAssets();
        console2.log("assets stranded in pendingWithdrawAssets forever:", stranded);
        console2.log("vault token balance                             :", token.balanceOf(address(vault)));
        console2.log("vault totalAssets() (balance - pending)         :", vault.totalAssets());
        assertGt(stranded, 0, "expected permanently stranded assets");
        // The stranded amount is exactly Alice's re-queued withdrawal.
        assertApproxEqAbs(stranded, requeued, 2, "stranded != alice re-queued amount");
    }
```

`_deposit`, `_nextPeriod` and the fixture wiring are the ordinary harness helpers; the test runs against the unmodified contracts at the pinned commit.

### Reproducing the recovery matrix

The recovery table in the report body was produced the same way, on the same unmodified pinned code, with each row paired against a control account holding a byte-identical position that was never rescued:

```
forge test --match-contract RecoveryMatrix -vv
forge test --match-contract RemediationCheck -vv
```

`RemediationCheck` applies each proposed fix to the pinned source in an isolated clone and re-runs the project's own suite: baseline 448 passing, and 448 passing under either remediation.
