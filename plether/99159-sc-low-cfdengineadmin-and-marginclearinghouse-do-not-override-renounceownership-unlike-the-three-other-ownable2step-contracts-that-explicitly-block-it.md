# \#99159 [SC-Low] CfdEngineAdmin and MarginClearinghouse do not override renounceOwnership, unlike the three other Ownable2Step contracts that explicitly block it

**Submitted on Nov 12th 2026 at 07:11:49 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99159
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

Three of the five `Ownable2Step` contracts in the perps stack explicitly override `renounceOwnership()` to revert, each with a comment explaining that governance must retain authority: `EmergencyPauseCoordinator.sol:101-104`, `HousePool.sol:265` and `OrderRouterAdmin.sol:125`. `CfdEngineAdmin` and `MarginClearinghouse` inherit `Ownable2Step` without the override, so the inherited `renounceOwnership()` remains callable by the owner.

The consequence is a permanent loss of administrative capability in a system that is explicitly non-upgradeable. SECURITY.md states that admins can tune risk parameters and pause entrypoints but cannot swap logic or rewrite deployed code, so there is no proxy upgrade path to recover ownership once it is renounced.

For `CfdEngineAdmin`, ownership controls all three timelocked config families: `EngineRiskConfig` (VPI factor, margin basis points, execution fee, frozen close spread, settlement buffer), `EngineCalendarConfig` (FAD overrides and runway) and `EngineFreshnessConfig` (staleness limits). Renunciation freezes every timelocked parameter at its current value permanently, and additionally makes `proposeProtocolTreasury`, `cancelProtocolTreasuryProposal`, `setProtocolTreasury` and `recapitalizePool` permanently inaccessible.

For `MarginClearinghouse`, ownership controls `rescueToken` (`:324-327`). After renunciation, non-settlement tokens accidentally transferred to the clearinghouse become permanently locked. Settlement USDG is excluded from rescue by design, so this is the weaker of the two, but it contradicts the protocol's stated recovery capability.

The inconsistency is the point: the codebase already treats renunciation as a hazard worth blocking in three places, and the two contracts that do not block it are exactly the ones controlling risk parameters and token rescue.

Recommendation: override `renounceOwnership()` to revert in both `CfdEngineAdmin` and `MarginClearinghouse`, matching the existing pattern, for example `function renounceOwnership() public view override onlyOwner { revert OwnableInvalidOwner(address(0)); }`.

## Proof of Concept

This is a code-consistency insight with a verifiable state effect, so the reproduction is a call comparison.

Step 1. On EmergencyPauseCoordinator, call renounceOwnership() as the owner. The call reverts, because the contract overrides the inherited function at :101-104. The same holds for HousePool at :265 and OrderRouterAdmin at :125.

Step 2. On CfdEngineAdmin, call renounceOwnership() as the owner. The call succeeds, because the contract inherits Ownable2Step without an override. After the call, owner() returns the zero address permanently: there is no proxy and no recovery path, so every onlyOwner function becomes unreachable, including the three timelocked config families and proposeProtocolTreasury, cancelProtocolTreasuryProposal, setProtocolTreasury and recapitalizePool.

Step 3. On MarginClearinghouse, call renounceOwnership() as the owner. The call succeeds for the same reason. After the call, rescueToken is permanently unreachable, so any non-settlement ERC-20 token transferred to the clearinghouse by mistake is locked forever. Settlement USDG is excluded from rescue by design, which limits but does not remove the loss of recovery capability.

Step 4. The comparison is the evidence: three of five Ownable2Step contracts treat renunciation as a hazard and block it with an explanatory comment; the two that do not are the contract that owns risk parameters and the contract that owns token rescue.

Suggested fix verification: after adding the override to both contracts, the step 2 and step 3 calls should revert with OwnableInvalidOwner(address(0)), matching the three existing implementations.

Minimal reproducer sketch in Foundry: vm.prank(owner); vm.expectRevert(); EmergencyPauseCoordinator(epc).renounceOwnership(); then vm.prank(owner); CfdEngineAdmin(admin).renounceOwnership(); and assert admin.owner() == address(0).
