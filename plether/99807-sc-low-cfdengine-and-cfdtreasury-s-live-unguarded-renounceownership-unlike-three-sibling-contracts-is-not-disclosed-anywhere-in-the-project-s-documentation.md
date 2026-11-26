# \#99807 [SC-Low] CfdEngine and CfdTreasury's live, unguarded renounceOwnership() — unlike three sibling contracts — is not disclosed anywhere in the project's documentation

**Submitted on Nov 15th 2026 at 11:14:18 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99807
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

Category: Insight - Documentation Improvements.

`CfdEngine` and `CfdTreasury` have a live,
callable, undocumented `renounceOwnership()` function — unlike three sibling contracts, where the
identical capability is explicitly guarded against and the guard is documented as a deliberate decision. This
is an explicit gap between what the project's documentation describes and what the deployed code
actually allows.

`CfdEngine` (`packages/perps/src/CfdEngine.sol` line 46: `contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdTreasury, ReentrancyGuardTransient`) inherits `CfdTreasury` (`packages/perps/src/CfdTreasury.sol` line 10: `abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step`), which inherits OpenZeppelin's `Ownable2Step`. `CfdTreasury`'s constructor sets the owner to the deployer (`CfdTreasury.sol` line 39: `constructor() Ownable(msg.sender)`); `CfdEngine`'s own constructor calls `CfdTreasury()` with no further owner argument (`CfdEngine.sol` line 294).

A search for `renounceOwnership` across all of `packages/perps/src` returns exactly three matches, and neither `CfdEngine.sol` nor `CfdTreasury.sol` is one of them: `HousePool.sol` line 265, `OrderRouterAdmin.sol` line 125, and `EmergencyPauseCoordinator.sol` line 102. All three are identical in shape: `function renounceOwnership() public view override onlyOwner { revert OwnableInvalidOwner(address(0)); }`, each preceded by a one-line natspec stating why — "Ownership cannot be renounced because governance must retain emergency recovery authority" on `HousePool.sol` and `OrderRouterAdmin.sol`, and "...authority to rotate or disable the guardian" on `EmergencyPauseCoordinator.sol`. Declaring the override `view` makes the block unconditional: the function can never reach the state-changing base implementation regardless of who calls it (a non-owner caller already reverts earlier, on the inherited `onlyOwner` check itself). A further search confirms no contract under `packages/perps/src` overrides `_transferOwnership`, `transferOwnership`, or `acceptOwnership` at all: `renounceOwnership` is the only point at which any of these three sibling contracts customises behaviour relative to the vanilla OpenZeppelin base, and `CfdEngine`/`CfdTreasury` has no equivalent customisation.

Because neither file overrides it, the inherited `Ownable.renounceOwnership()` (`lib/openzeppelin-contracts/contracts/access/Ownable.sol` lines 76-78: `public virtual onlyOwner { _transferOwnership(address(0)); }`) is live on `CfdEngine`/`CfdTreasury` exactly as OpenZeppelin wrote it: the current owner can call it at any time, with no parameters, no timelock and no second step, immediately setting the owner to `address(0)`.

This has a concrete, present-tense consequence specific to this contract. `CfdEngine.clearDegradedMode()` (`CfdEngine.sol` lines 658-667) is `external onlyOwner` and is the only function that can clear `degradedMode` (`CfdEngine.sol` line 121: `bool public degradedMode`, documented as "Whether terminal settlement detected insolvency and latched risk-increasing operations off") once solvency is restored; it reverts `CfdEngine__NotDegraded()` if the latch is not set and `CfdEngine__StillInsolvent()` if `_availableCashForFreshPoolPayouts() < _maxLiability()`. The latch itself is entered automatically and permissionlessly by `_enterDegradedModeIfInsolvent` (`CfdEngine.sol` lines 1351-1369) whenever a close or liquidation leaves the pool's effective assets below its maximum directional liability — an ordinary, foreseeable operational event, not a contrived one. If the owner ever calls the live, unguarded `renounceOwnership()` — before or after any particular degraded-mode episode — `owner()` becomes `address(0)` permanently, and the inherited `onlyOwner` check can never again be satisfied by any account. `clearDegradedMode()`, along with every other `onlyOwner` function on this lineage (`setDependencies`, `recapitalizePool`, `setProtocolTreasury`/`proposeProtocolTreasury`/`cancelProtocolTreasuryProposal`, and others), becomes permanently unusable from that point on, including for any future degraded-mode episode as well as a currently active one.

The gap itself, and its correction, are entirely in the in-scope `CfdEngine.sol`/`CfdTreasury.sol`
NatSpec and source under `packages/perps/src`; the prose documentation cited below sits outside
`src` and is referenced only to establish what the project's own documentation says about this
in-scope code's behavior, matching this Insight category's own "documentation or NatSpec comments"
definition. The project's own documentation names this exact capability as a current owner action in at least two places, and never once states that it lacks the guard its sibling contracts have. `SECURITY.md`'s "Instant owner controls" enumeration — the document's own canonical list introduced by "The owner can act immediately to:" — states: "recapitalize the pool from the owner's wallet, clear degraded mode once raw solvency is restored, and rescue accidentally transferred non-settlement tokens from the Clearinghouse" and, in the same list, "initiate an ownership transfer, which the pending owner must explicitly accept through the `Ownable2Step` flow" (`SECURITY.md` lines 246-247, 248). Transfer is named explicitly as the way ownership changes; renunciation is not named at all, in either that "can" list or the paired "The owner cannot:" list immediately following it (`SECURITY.md` lines 250-256: change deployed logic, change `CAP_PRICE`, rewrite historical state, directly seize arbitrary user funds, bypass the core solvency/withdrawal model). `EMERGENCY_RESPONSE_GUIDE.md` repeats the same capability — "The Engine owner can call `clearDegradedMode()` once raw `E >= L`" (lines 45-48) — and its "Canonical references" table lists `CfdEngine` as "degraded mode and solvency-gated recovery" (line 260). `DEPLOYMENT.md` states, generally and without any carve-out, that "Ownership-bearing contracts use `Ownable2Step`: a proposed new owner must call `acceptOwnership()`" (lines 302-303) — again describing only the transfer path. None of these documents states that `CfdEngine`/`CfdTreasury` has no `renounceOwnership` override, or that this differs from the identical, explicitly-reasoned guard on `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`.

The single closest approach to the subject anywhere in the repository's prose documentation is one sentence in `SECURITY.md`'s "Protocol treasury commitment" section: "Ownership transfer preserves the schedule; renouncing ownership does not prevent activation" (`SECURITY.md` lines 127-128). Read with its surrounding paragraph (lines 114-128), this sentence addresses a single, narrow consequence — a previously scheduled treasury successor still activates automatically by timestamp regardless of who holds ownership — not a general statement about whether renunciation is possible, guarded, or consequential elsewhere on the contract. It does show that the documentation's authors specifically contemplated "the owner renounces ownership" as a live, occurring event for this exact contract; the same document goes on, 118 lines later, to list `clearDegradedMode()` as a current owner action without ever connecting the two. `WHITEPAPER.md`'s own dedicated risk section, "10.5 Governance and implementation risk" (lines 1825-1832), discusses timelock observability and non-upgradeability trade-offs and likewise does not mention renunciation.

A reader relying solely on the documented owner-action inventory in `SECURITY.md`, `EMERGENCY_RESPONSE_GUIDE.md`, or `DEPLOYMENT.md` has no way to learn that `CfdEngine`/`CfdTreasury`'s ownership can be fully and permanently renounced with a single ordinary call, or that this differs from the three other governance-bearing contracts the same documents describe.

## Possible Remediation

A documentation-only correction for this is to add `CfdEngine`/`CfdTreasury`'s unguarded, live `renounceOwnership()` status to the existing inventories. In `SECURITY.md`'s "Instant owner controls" list, state plainly that unlike `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`, this contract's `renounceOwnership()` is not overridden and remains fully live, and that calling it would permanently foreclose `clearDegradedMode()` and every other `onlyOwner` function listed for it, with no other account able to restore them afterward. The same addition, or a cross-reference to it, belongs in `EMERGENCY_RESPONSE_GUIDE.md`'s "Canonical references" entry for `CfdEngine`, since that is the document a responder consults during an active degraded-mode incident. This is a documentation-only change: no code, ABI, storage, or on-chain behaviour changes, and it applies regardless of whether the project judges the current unguarded behaviour acceptable or not.


## References

- `packages/perps/src/CfdEngine.sol` lines 46, 121, 288-337, 658-667, 1351-1369.
- `packages/perps/src/CfdTreasury.sol` lines 1-41 (declaration line 10, constructor line 39).
- `packages/perps/src/HousePool.sol` lines 264-267.
- `packages/perps/src/OrderRouterAdmin.sol` lines 124-127.
- `packages/perps/src/EmergencyPauseCoordinator.sol` lines 101-104.
- `packages/perps/SECURITY.md` lines 114-128, 230-257.
- `packages/perps/EMERGENCY_RESPONSE_GUIDE.md` lines 43-48, 255-263.
- `packages/perps/DEPLOYMENT.md` lines 302-303.
- `packages/perps/WHITEPAPER.md` lines 1825-1832.
- `lib/openzeppelin-contracts/contracts/access/Ownable.sol` lines 76-89 and `lib/openzeppelin-contracts/contracts/access/Ownable2Step.sol` (pinned dependency; inherited base behaviour).

(frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)



## Proof of Concept

# Proof of Concept

**Conditions under which this Insight is valuable.**

The gap itself is in the in-scope `CfdEngine`/`CfdTreasury` source and NatSpec under `packages/perps/src`; the prose documentation searched below is read only to confirm what the project's own documentation currently says about that in-scope code, not as the location of the finding.

1. The gap matters at the moment the current Engine/Treasury owner, or any future owner reached through the documented `Ownable2Step` transfer path, calls the live, zero-argument `renounceOwnership()`, for any reason. A reader who has reviewed only `SECURITY.md`, `EMERGENCY_RESPONSE_GUIDE.md` and `DEPLOYMENT.md` has no documented basis to expect this call to behave any differently on `CfdEngine`/`CfdTreasury` than it does on the three sibling contracts in the same package, where it is permanently blocked; nor does any document connect the one sentence that does discuss renunciation on this contract (the treasury-schedule sentence) to the separately documented `clearDegradedMode()` capability that renunciation would also foreclose.
2. The documentation gap has the most bite exactly when there is the least time to discover it: `SECURITY.md`'s own "owner cannot" list enumerates several things the owner is unable to do, but does not include "undo a renunciation" or flag that no other account can replace a vanished owner, so there is no documentary prompt anywhere in the repository to check this specific contract's guard before acting — unlike for the three siblings, where the guard itself makes the question moot.
3. The fix is inexpensive and applies regardless of what the project ultimately decides about the underlying behaviour: it is an addition to two already-existing inventories, not a rewrite, and it lets a reader make an informed decision that the current documentation does not support.
4. This is a present-tense documentation gap, not one contingent on a future code change: the omitted capability is already live in the source at the pinned commit, which already exposes this behavior, already callable by the already-designated owner with a single, already-possible transaction. No new deployment and no new code of any kind has to be written for the omitted fact to be true today. It is the same kind of present-tense fact `SECURITY.md`'s own "Instant owner controls" list already treats as documentable for each of its other entries -- the owner's ability to recapitalize the pool or clear degraded mode is equally contingent on the owner choosing to call it, and both are still correctly listed as current capabilities. This submission asks only that the same list be complete for this one contract.

Read-only reproduction at the frozen commit:

1. `grep -rn "renounceOwnership" packages/perps/src` returns exactly three matches: `HousePool.sol:265`, `OrderRouterAdmin.sol:125`, `EmergencyPauseCoordinator.sol:102`. Neither `CfdEngine.sol` nor `CfdTreasury.sol` appears.
2. `sed -n '260,268p' packages/perps/src/HousePool.sol`, `sed -n '120,128p' packages/perps/src/OrderRouterAdmin.sol`, and `sed -n '98,105p' packages/perps/src/EmergencyPauseCoordinator.sol` each show the identical guarded body — `function renounceOwnership() public view override onlyOwner { revert OwnableInvalidOwner(address(0)); }` — with its one-line natspec rationale directly above.
3. `sed -n '1,41p' packages/perps/src/CfdTreasury.sol` shows the contract declaration `abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step` (line 10) and its full constructor `constructor() Ownable(msg.sender)` (lines 39-41). Point 1's repository-wide search already confirms no `renounceOwnership` override exists anywhere in this file (or anywhere else in `packages/perps/src`). `grep -rn "_transferOwnership\|function transferOwnership\|function acceptOwnership" packages/perps/src/` returns no matches at all in the package (recursive, covering every subdirectory, not only the top level).
4. `grep -n "clear degraded mode\|clearDegradedMode" packages/perps/SECURITY.md packages/perps/EMERGENCY_RESPONSE_GUIDE.md` shows the capability documented at `SECURITY.md:246` and `EMERGENCY_RESPONSE_GUIDE.md:47`. `EMERGENCY_RESPONSE_GUIDE.md`'s separate "Canonical references" table additionally lists `CfdEngine` as "degraded mode and solvency-gated recovery" at line 260 (read directly; that line's wording does not contain "clear" or the function name, so it is not part of this exact search's output).
5. `grep -rln -i "renounc" --include="*.md" --exclude-dir=lib .` run against the complete project repository tree -- every `.md` file anywhere in it, not only the top-level and package-level files (this additionally covers every deployment-release README, every test-suite README, the audit-scan report, task notes, PR templates, and every other `.md` file in the tree), excluding the vendored third-party dependency trees under `lib/` (OpenZeppelin, forge-std, halmos-cheatcodes, morpho-blue -- git submodules of upstream libraries, not this project's own documentation) -- returns exactly one file: `packages/perps/SECURITY.md`. (The one hit this exclusion removes, `lib/openzeppelin-contracts/CHANGELOG.md`, is OpenZeppelin's own upstream release changelog, whose two matching lines describe unrelated `TimelockController` admin-role and `AccessControlEnumerable.renounceRole` history -- not Plether documentation and not about `Ownable`/`Ownable2Step`'s `renounceOwnership()`.) `sed -n '114,128p' packages/perps/SECURITY.md` shows its single matching line's surrounding paragraph concerns only the treasury-successor schedule.

Conclusion of the demonstration: unlike `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`, `CfdEngine`/`CfdTreasury`'s renunciation path was left exactly as OpenZeppelin wrote it, and the project's owner-action documentation, while naming this exact contract's recovery capability explicitly in two separate documents, never says so.

