# \#99964 [SC-Critical] Junior LP exit is priced on a refundable VPI charge and takes remaining LPs' principal

**Submitted on Nov 15th 2026 at 22:31:14 UTC by @ChaseTheLight for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99964
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A positive VPI charge is a trader prepayment. A close at the same skew generates a rebate of up to the whole positive accrual, paid to the extent free pool cash is available; any unfunded portion is waived (`ACCOUNTING_SPEC.md:1061-1063`). `CloseAccountingLib` floors lifetime VPI at zero, which is what makes the whole accrual reclaimable (`CloseAccountingLib.sol:112-118`, `CfdEngineSettlementSidecar.sol:621-624`), and the specification states the rule for the mirror quantity outright: a VPI reserve "cannot be extracted by LPs before its account obligation settles" (`ACCOUNTING_SPEC.md:821-822`). That reserve exists only on the negative side, and it is held against the trader's own collateral.

On the positive side the charge is booked as arrived revenue the moment the open executes (`CfdEngineSettlementSidecar.sol:454-464`), the waterfall credits it to Junior principal whenever Senior sits at its high-water mark (`HousePoolWaterfallAccountingLib.sol:140-159`), and LP epoch settlement then prices Junior redemptions on that same storage slot five lines after the reconcile that credited it (`HousePool.sol:835`, `:840-841`). No liability bucket for the pending refund exists anywhere in pool accounting: neither the terminal-NAV snapshot nor the withdrawal reserve carries a VPI term (`HousePoolAccountingLib.sol:46-55`, `:60-78`). So a Junior LP can request a redemption while a skew-increasing position is open, settle the epoch through `OrderRouter.settleLpEpoch`, which is permissionless and payable (`OrderRouter.sol:479-484`), and claim real settlement token at a price containing a refund it will not fund.

When the trader closes at the same mark, the rebate is paid out of pool cash (`CfdEngineSettlementSidecar.sol:621-624`) and the next reconcile absorbs it from Junior principal first (`HousePoolWaterfallAccountingLib.sol:167-184`), from the holders who stayed. The exiting LP's shares are already burned and the settlement token is already in its wallet (`TrancheVault.sol:1211`).

The attacker is not a privileged party and does not need anyone's cooperation: it buys into the Junior tranche through the vault's ordinary asynchronous deposit, waits the one hour cooldown, and redeems, and the charge sits in the tranche's equity against a one hour LP epoch, so there is no block to win and no trade to foresee.

The proof of concept runs locally and offline, in process inside a network namespace whose only interface is loopback, with nothing dialled, signed or broadcast (Proof of Concept). The 2026-11-15 run passes both measurements against the audited source at the pinned commit, and the Proof of Concept section carries the figures the capture printed.

## Vulnerability Details

### Root cause

The open side folds the signed VPI charge into trade cost and loses the distinction between the refundable part and the non-refundable fee. `OpenAccountingLib.sol:135`:

```solidity
        state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

`CfdEngineSettlementSidecar.executeOpen` then books the whole non-fee remainder as revenue that has already arrived in cash (`:454-464`):

```solidity
        if (delta.tradeCostAssets > 0) {
            uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
            if (poolCashInflowAssets > 0) {
                IHousePool(host.pool())
                    .recordClaimantInflow(
                        poolCashInflowAssets,
                        IHousePool.ClaimantInflowKind.Revenue,
                        IHousePool.ClaimantInflowCashMode.CashArrived
                    );
            }
        }
```

`ClaimantInflowKind.Revenue` is distributable. The waterfall restores any Senior high-water-mark deficit and credits the whole remainder to Junior (`HousePoolWaterfallAccountingLib.sol:140-159`), so with Senior whole, which is the ordinary state, the next reconcile raises `juniorPrincipal` by the entire charge:

```solidity
        if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
            nextState.seniorHighWaterMark = nextState.seniorPrincipal;
        }

        nextState.juniorPrincipal += remaining;
```

`HousePool._settleLpEpoch` reconciles and then takes the Junior redemption price from that slot in the same call (`:835`, `:840-841`):

```solidity
        _reconcile(accountingSnapshot, statusSnapshot);
        _checkpointEngineCarryIndexes();

        result.cutoffEpoch = cutoffEpoch;
        RedemptionPhase memory juniorPhase;
        juniorPhase.pricingPrincipal = juniorPrincipal;
```

`pricingPrincipal` is the numerator the redemption sidecar divides by share supply, both for the full head quote and for the partial-funding quote (`:957-959`, `:992-1000`). The ordering is the defect: the credit arrives, and five lines later it is the price a leaver is paid at.

The refund then lands on whoever is still there. The close returns up to the whole accrual because the lifetime clamp is written to permit exactly that (`CloseAccountingLib.sol:112-118`):

```solidity
        // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
        // where an attacker opens at high depth, reduces pool depth, then closes to extract a
        // net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
        // stale-price risk is priced separately through frozenSpreadAssets.
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

The rebate is paid from pool cash to the clearinghouse (`CfdEngineSettlementSidecar.sol:621-624`) and the resulting loss is absorbed Junior first, Senior second (`HousePoolWaterfallAccountingLib.sol:167-184`). The holders who stayed therefore fund the whole of a refund they previously shared pro rata with the leaver.

### What the code reserves, and what it leaves unreserved

The project reserves the mirror quantity and says so in its own security notes: "gross negative lifetime VPI is backed by a dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`" (`SECURITY.md:866-867`). That reserve is implemented as `_negativeVpiReserveTarget` (`CfdEngineSettlementSidecar.sol:373-379`), it is a lock on the trader's collateral rather than a pool liability, and it is inert while lifetime VPI is positive.

| | negative lifetime VPI | positive lifetime VPI |
| --- | --- | --- |
| Who will owe the value later | the trader | the pool |
| Is a reserve held | yes, `max(-vpiAccrued, 0)`, nonwithdrawable | no |
| Can an LP exit holding the value | no, the specification forbids it at `ACCOUNTING_SPEC.md:821-822` | yes, it is priced into `pricingPrincipal` |
| Where the value sits meanwhile | locked collateral at the clearinghouse | `juniorPrincipal`, distributable |

Both halves of a signed, reversible charge are treated as final in the direction that favours the party who can leave. The second column is settled at the three quantities that produce a redemption price rather than at the point one is consumed. The terminal book's `distributable` is `physicalAssets` plus the signed terminal delta minus trader claims (`HousePoolAccountingLib.sol:60-78`). The withdrawal reserve's base is `maxLiabilityAssets + traderClaimBalanceAssets + supplementalReservedAssets` (`:46-55`), and its supplemental term is a basis-point share of `maxLiabilityAssets` and nothing else, at both of its two call sites (`HousePool.sol:1252-1253` and `CfdEngineProtocolLens.sol:67-69`, into `libraries/SolvencyAccountingLib.sol:77-85`). And `vpiAccrued` occurs in `TerminalNavBookV2.sol` exactly once, in a locally declared engine-view interface at `:28`, and in none of its arithmetic. No pending-refund term exists in any of the three.

### Trigger and preconditions

The firing state, in the protocol's own terms: a positive VPI charge from a skew-increasing open is outstanding and unreserved, that position is still open when the epoch carrying the attacker's Junior redemption request settles, Senior is at its high-water mark so the whole charge lands on Junior, and free pool cash covers the exit.

**What is attacked is the shared pool's Junior tranche**, specifically the Junior LP shares held by somebody other than the attacker, and behind them the Senior tranche, which absorbs the remainder once Junior principal is exhausted (`HousePoolWaterfallAccountingLib.sol:167-184`).

**The attacker occupies its position by its own act, and that is the whole precondition.** It joins the tranche later, through `TrancheVault.requestDeposit` (`:562`, `:569`), waits for the deposit epoch to activate, claims shares with `claimDeposit` (`:968`), waits the one hour `DEPOSIT_COOLDOWN` (`TrancheVault.sol:38`, enforced at `:617`), and requests redemption of its whole balance (`:606-631`). The minimum deposit is 1 USDG (`HousePool.sol:176`) and the size is the attacker's choice, which is what sets its share of the charge. No invitation, no allowance, no counterparty consent, and no privilege at any step.

**A co-holder always exists while trading is live**, by protocol invariant rather than by survey. `activateTrading` cannot be called until both tranche seeds are initialized (`HousePool.sol:532-536`), seed shares are permanently non-redeemable and non-transferable below the floor (`TrancheVault.sol:322`, `:1630-1637`), and Senior is capped at 80% of the pool (`DEPLOYMENT.md:155`). So on any pool with trading active there is at least one Junior holder other than the attacker, and when Junior equity is thin the shortfall reaches Senior. The release seed is 1 USDG (`DEPLOYMENT.md:156`, `:249`), so the seed holder alone is a dust-sized counterparty: the magnitude of the transfer scales with the other holders' share of the tranche, which the programme's own rule makes an amount question rather than an impact question.

The one step that is caller-restricted is open to any caller. With open positions and a live oracle, `HousePool.settleLpEpoch` admits only the Router, only with the Engine's exact current mark, and only with a publish time inside the current epoch (`HousePool.sol:809-819`):

```solidity
        if (accountingSnapshot.hasOpenPositions && !statusSnapshot.oracleFrozen) {
            if (
                msg.sender != ENGINE.orderRouter() || ENGINE.lastMarkPrice() != expectedMarkPrice
                    || statusSnapshot.lastMarkTime != expectedPublishTime
            ) {
                revert HousePool__Unauthorized();
            }
```

`OrderRouter.settleLpEpoch(bytes[])` is permissionless and payable (`OrderRouter.sol:479-484`), and the logic it delegates to installs that exact mark on the Engine and then calls the pool in the same frame (`router/OrderLiquidationBatchLogic.sol:549-581`), so it satisfies all three conditions on behalf of whoever calls it. The order side is the same: `commitOrder` (`OrderRouter.sol:85-90`) and `executeOrder` (`:240-247`) are both open calls, and the programme's scope statement puts this surface in scope explicitly, since keepers hold no special privileges and anyone may execute orders and price updates.

The programme's own active-features statement puts the in-scope code on Arbitrum One with USDG as the settlement token, and every parameter the state depends on is at the release value the programme nominates for severity assessment (`packages/perps/DEPLOYMENT.md` at this commit): `vpiFactor 0.01e18` (`:131`), `maxSkewRatio 0.4e18` (`:133`), `executionFeeBps 4` (`:142`), `settlementBufferBps 25` (`:143`), Senior limits `40_000_000e6` and `8_000` bps (`:154-155`), and `juniorMaintenanceFeeAprBps 0` (`:152`), so nothing erodes the attacker's shares while it waits. `LP_EPOCH_DURATION` and `DEPOSIT_COOLDOWN` are compiled constants of one hour (`HousePool.sol:178`, `TrancheVault.sol:38`) rather than deployment inputs. Trading on the basket, both tranches and permissionless keepers are active at launch; nothing on this path touches the one feature that is not.

Nothing on the exit path offsets the extraction. The LP settlement fee is zero whenever the oracle is not frozen (`HousePool.sol:1147-1155`):

```solidity
    function _settlementFeeBps(
        bool isSenior,
        bool oracleFrozen
    ) internal view returns (uint256) {
        if (!oracleFrozen) {
            return 0;
        }
        return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps;
    }
```

so the configured frozen tranche fees are inert in the live regime, and the exit is funded in full from free cash after `maxLiabilityAssets`, trader claims and the settlement buffer have been reserved (`HousePool.sol:842-843`, `:881`).

The charge is held in `accountedAssets` from the moment the open executes (`HousePool.sol:640-642`), and shows up at once in the tranche's pending waterfall equity, which `getPendingTrancheState` recomputes from it (`:1295`, `:1311`); the reconcile inside `HousePool._settleLpEpoch` (`:835`) is what writes it into stored `juniorPrincipal`, five lines before the exit is priced on it (`:840-841`). The trader's outstanding accrual carries no time decay: it changes only when the position does. The Junior share of it is not static, though. Every reconcile pays the senior coupon out of junior principal, capped by it (`HousePoolWaterfallAccountingLib.sol:56-77`), and losses and other LP flows move the same slot, so on a large Senior tranche the charge erodes as time passes. The window the attack needs is therefore one LP epoch, a compiled one hour, falling inside the life of a position that carries an outstanding positive charge. The attacker does not have to act within a block or predict a trade.

### Recommended fix

Treat an outstanding positive lifetime VPI accrual as a pool liability for as long as the position that can reclaim it is open, which is what the negative side already does from the other direction.

The structural change is to carry the aggregate `max(vpiAccrued, 0)` over open positions into the pool's accounting snapshots, deducting it in `HousePoolAccountingLib.buildReconcileSnapshot` from `distributable` and in `buildWithdrawalSnapshot` from `freeAssets`, and to subtract it from the pricing principal that LP entry and exit both read. The value is already available to the pool: the Engine's terminal snapshot is fetched on every settlement, and the interface getter for `vpiAccrued` already exists.

The minimal change that closes this path alone is to stop pricing a redemption on a principal that includes an unsettled prepayment. In `HousePool._settleLpEpoch`, set `juniorPhase.pricingPrincipal` to `juniorPrincipal` reduced by the outstanding positive VPI accrual rather than to `juniorPrincipal` itself, and apply the same reduction to the deposit phase's `pricingAssets` at `:915` so entry and exit stay symmetric, which the project's own invariant requires (`SECURITY.md:392`).

Either change can be made without touching the open-side booking. If the booking is changed instead, the clean form is a `ClaimantInflowKind` for a refundable prepayment that credits `accountedAssets` without entering the waterfall, released into revenue only when the position closes and the lifetime balance is final.

## Impact Details

### The in-scope impact this maps to

The governing entry, quoted from the programme's Impacts in Scope list:

> Critical: direct theft of any user funds (at-rest or in-motion) other than unclaimed yield; permanent freezing of funds; protocol insolvency.

This report files at **Critical** on direct theft of at-rest user funds. A remaining Junior holder's deposited principal moves to the exiting holder's wallet. The transfer is not a share of a gain the pool made and not a timing artefact on a mark: the pool's cash went up by a prepayment the closing trader can reclaim out of free pool cash, and the leaver was paid its fraction of that prepayment in settlement token while the obligation stayed behind.

The test in the Proof of Concept section takes both arms of that claim from one snapshot: the attacker is strictly better off exiting than holding the same shares through the close, and the holder who stayed is strictly worse off than in the hold control, with the attacker's gain and that holder's loss asserted to agree within the test's two percent relative tolerance. The holder who stayed is asserted to end below its deposited principal, not merely below its accrued yield, which is what distinguishes this from the tier below, where "theft of unclaimed yield" sits.

### Loss mechanism and its bound

Whose balance moves, by how much, and through which path:

- The attacker's balance moves through `TrancheVault.claimRedeem` (`:1014`) into `_consumeRedeemClaim`, which transfers settlement token out of the vault to the receiver (`:1211`). After that call the value is an ERC-20 balance in an ordinary wallet, which is why permanence is asserted on that balance rather than on a share price.
- The remaining holder's redeemable claim on pool assets falls below its deposited principal when the next reconcile absorbs the rebate from `juniorPrincipal` (`HousePoolWaterfallAccountingLib.sol:167-184`), the slot its own redemption price is computed from.
- The trader is whole either way. It pays the charge on the open and receives it back on the close, net of its own execution fees, and it is not the beneficiary.

The driver of the size is the attacker's tranche share of the refund it escaped, with the charge as the ceiling, since the clamp bounds the refund by the accrual. The per-event ceiling follows from the charge curve and the open cliff, both of which are fixed at the release parameters. `CfdMath.tryGetSkewCost` is `cost = vpiFactor * skew^2 / depth / 2` (`:134-139`), and an open is rejected with `SKEW_TOO_HIGH` when post-open skew exceeds `maxSkewRatio` of economic depth without strictly reducing it (`libraries/CfdEnginePlanLib.sol:473-480`), so one event's charge tops out at `vpiFactor * maxSkewRatio^2 / 2`, which at `0.01e18` and `0.4e18` is 8 basis points of pool depth. Depth is the Engine's own economic depth, canonical assets net of outstanding trader claims (`libraries/CfdEnginePlanLib.sol:487`, documented at `SECURITY.md:391` and `:1438`), so the figure scales linearly with the size of the pool rather than being a property of a fixture. Eight basis points bounds the charge, not the extraction. The take is the attacker's Junior-tranche share of that charge, so the realized extraction sits below the charge ceiling; the ceiling the test checks the accrual against is taken on gross pool assets, which is the looser of the two bounds, and the economic-depth figure is not read anywhere in the test. On the 2026-11-15 run the one event booked a charge of 5,780 USDG on a 10,000,002 USDG pool, which is 5.78 basis points of gross assets, under the 8,000 USDG release ceiling because the open sits below the skew cap; the attacker was paid 2,646 USDG more by exiting than by holding the same shares through the close, while the remaining holder lost 2,650 USDG, the two agreeing to within 0.13 percent.

Whether that loss reaches deposited principal, which is what separates this from the tier below, depends on the exiting holder's share of the Junior tranche, and that share is the attacker's own lever: it is set by the deposit size the attacker chooses, and the programme does not treat recoverable attacker capital as a downgrade reason. In the Proof of Concept the exiting holder holds about 60 percent of the tranche, and of the remaining holder's 2,650 USDG loss, 2,102 USDG is capital taken below its deposited principal and 548 USDG is yield it would otherwise have kept. A smaller exiting share moves less principal, and a sufficiently small one would confine the loss to yield, but nothing obliges the attacker to choose one.

The programme's own rule governs what that bound means for the rung:

> Impact is calculated first, separately from feasibility. Feasibility does not change impact/severity, only the amount paid. A large-impact but infeasible finding keeps its impact and severity, paid reduced.

### A second victim on the same root cause

The same unreserved prepayment reaches the deposit side. A Junior LP that deposits while a positive charge is outstanding enters at a NAV built from that charge, then absorbs its share of the refund after entry, Junior first. In the Proof of Concept's second measurement the entrant deposits 2,000,000 USDG and ends at 1,999,426 USDG, 574 USDG below its own deposit, which is the clean, measured loss this case rests on. The incumbent holders do end ahead when the entrant's cash is present, but only part of that gain is the entrant's funded loss; the rest is the trader's smaller close rebate against the deeper pool the entrant's deposit creates, which is the separate, already-known depth effect discussed below. The finding here is therefore the entrant's sub-deposit loss, not the incumbent-side figure. The case runs with Senior at its high-water mark and ordinary tranche deposits open, so it needs no prior loss event of any kind.

A heavier instance sits on top of it. If a bad-debt event has already wiped Junior and left Senior just below its high-water mark, a positive VPI charge restores Senior exactly to the mark through the same `distributeRevenue` call and creates temporary Junior principal. The senior-impairment term in `HousePoolTrancheGateLib.trancheDepositsAllowed` is `projectedSeniorPrincipal < projectedSeniorHighWaterMark` alone (`src/libraries/HousePoolTrancheGateLib.sol:36-38`), so once the charge has restored the mark that gate stops blocking and ordinary tranche deposits reopen. The gate then treats a refundable prepayment as permanent Senior capital and admits the depositor it exists to keep out. That instance needs the prior wipe; neither the general case above nor the headline redemption path needs it, or the gate.

Both reach the same defective quantity: a refundable positive VPI accrual booked as `Revenue` with `CashArrived` and no offsetting liability.

### Why not lower, and the out-of-scope entries to weigh

The competing reading is the tier below, which reads "theft of unclaimed yield". It fails on the quantity the loss lands in: the rebate the close reclaims is absorbed from `juniorPrincipal` itself (`HousePoolWaterfallAccountingLib.sol:167-184`), which is the slot the remaining holder's own redemption price is computed from, so what moves is not the yield it had accrued but the capital behind it. That is the Critical rung this report claims, and the 2026-11-15 run measures it: the remaining holder ends 2,102 USDG below its deposited principal, not merely below its accrued yield, against a hold control in which the same holder ends above its deposit. The Medium entry for griefing is explicitly for damage with "no profit motive", which is not this case either: the attacker's own gain against the alternative of holding the same shares is the point of the path, and it is what the first measurement is built to compare.

The exclusion entries that touch this report, each checked against what the path actually does:

- **"attacks requiring privileged-address access"** and **"malicious/compromised privileged role"** do not apply. Every call on the attack path is open to any caller, and the Proof of Concept sets the steps out in order. No owner, pauser or guardian action is part of the attack. Privileged calls do appear elsewhere in the test, and none of them is an attack step. Its inherited setup performs the ordinary deployment, wiring, seeding and activation bootstrap that any live instance needs; the test file adds two timelocked Senior-limit updates that move the harness onto the committed release values, which is configuration parity, and the programme's own rule on that is that "Setting a parameter to a value the code allows is a trusted action, not a bug." Two further owner calls are taken after every measured figure has been read, and they test whether the owner can reverse the completed transfer rather than help it. The programme's scope statement also carves the execution surface in explicitly, on the ground that keepers have no special privileges and any caller may execute orders and price updates.
- **"incorrect third-party oracle data"** does not apply. There is no price move at all: the position opens and closes at the same mark, and the transfer comes from the booking of the charge rather than from any oracle value.
- **"lack-of-liquidity"** does not apply. The exit is funded in full out of free cash with the solvency reservations in place (`HousePool.sol:842-843`, `:881`).
- **"basic economic/governance attacks (51%)"**, **"Sybil"** and **"centralization risks"** do not apply. One ordinary address, one tranche, no voting weight and no second identity.
- The containment clause reduces severity "ONLY where demonstrated monitoring coverage and response timing establish that containment limits the exploit", and adds that the existence of monitoring or a pause does not by itself justify a downgrade and that no detection or response deadline is assumed unless documented. No such deadline is documented anywhere in the repository. The clause also states that "already-funded claims remain available", which is the step that completes this transfer: once the redemption is funded and claimed, a later pause cannot reach it. The separate pre-impact monitoring clause, which allows a one-level downgrade for automated prevention with 100% certainty and requires objective certainty of both detection and prevention where a human acts, is not engaged either: every step of the path passes the protocol's own on-chain checks, and the guardian's three fixed actions are manual calls rather than automatic controls.
- On attacker investment, the programme states that high capital is generally not a downgrade reason. The attacker's capital is its Junior deposit, which it recovers in the same exit.

### Not a duplicate, and scope

Nothing in the four limitation sets the programme declares known carries this root cause. In `SECURITY.md`, `## Known Limitations` runs from `:1328` to `:1457`, with `## Emergency Procedures` opening at `:1458`. The seven `### VPI limitations` bullets (`:1436-1444`) cover liquidation's missing fresh delta, depth being computed net of trader claims, behaviour at zero depth, the clamp applying identically across regimes, the frozen-market spread, the clamp zeroing rebate-only round trips, and partial-close linearity. None of them says an LP may exit at a price carrying an outstanding refundable charge, and none mentions a liability for positive VPI. The nine `### LP accounting limitations` bullets (`:1382-1399`) are about mark currency, book non-upgradeability, frozen-exit policy, backlog passes, coupon caps, Senior limit changes, grandfathered reservations and cooldown mixing. The nearest one runs the other way: "a positive marked receivable is not cash, so share NAV can exceed immediately redeemable liquidity" (`:1384`) warns that NAV can exceed redeemable liquidity, not that NAV may contain somebody else's refund. Of the other two sets, the Router and keeper bullets (`:1343-1375`) are queue, cleanup, client-id and retry-lane limitations, and the live basket component selection limitation (`:687-699`) is about which component mark a caller may choose within the permitted window, which this path does not turn on, since both legs execute at one mark.

The project also states a design intent that reads as if it could cover this, and it does not. `ACCOUNTING_SPEC.md:730-740` sets a "collection-time ownership policy": LP ownership follows collection time, with no entry or exit barrier, so revenue collected while an LP is present is that LP's to keep. That policy is written about carry, which the same passage calls trader debt that "becomes pool revenue" once collected, and `CloseAccountingLib` excludes carry from close economics entirely (`:43`, `:56`). Carry is not refunded to the trader through close economics, and once collected it is pool revenue, so its timing moves yield between LP cohorts. A positive VPI accrual is the opposite kind of quantity: the close clamp returns it to the trader up to the whole accrual (`CloseAccountingLib.sol:112-118`), so pricing an LP's exit on it moves principal, not yield. That reversibility is why the project reserves the mirror negative-VPI quantity and forbids LPs from extracting it before the account settles (`ACCOUNTING_SPEC.md:821-822`), while reserving nothing for carry timing. The collection-time policy governs the non-refundable quantity; it says nothing about pricing an exit on a refundable one. The stated LP invariant runs the other way: new LPs "do not inherit old trader obligations at a higher deposit NAV" (`SECURITY.md:859-860`), which is exactly what the deposit-side case makes them do.

The two VPI close items in the AI scan are both execution-time depth manipulation. The first is titled "Temporary pool liquidity can reduce a skew-increasing close charge" and its claim is that "An unprivileged trader who can temporarily supply HousePool liquidity before executing that close can inflate pool depth, reduce the positive VPI charge, and remove the liquidity afterward." The second is titled "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" and its claim is that "A trader who can temporarily reduce HousePool `totalAssets` around execution of a skew-reducing close can increase the magnitude of the close's negative VPI". Both are about moving the depth denominator around an execution so the curve returns a different number, and both are the trader's own charge. This report needs neither lever, and the clamp is why. `CloseAccountingLib.sol:112-118` bounds the refund by the accrual on the size being closed whatever the pool's depth is at that moment, so a redemption that takes assets out of the pool before the close cannot make the rebate exceed the charge the open already paid. The 2026-11-15 run shows the two close outflows almost identical across the arms, 4,414 USDG in the exit arm against 4,410 USDG in the hold arm, a 3 USDG difference beside a 2,646 USDG transfer, which isolates any residual depth effect at about 0.13 percent of the extraction; the separation also rests on the clamp in source. What changes between the arms is who holds Junior shares at the moment the charge is priced into the redemption. A fix to depth sampling, time-weighted or commitment-time, leaves this path exactly as it is, because the charge still reaches `pricingPrincipal` as distributable revenue.

The other entries in the known set that touch this neighbourhood are also different mechanisms. The pre-audit "Deposit Before Yield is Accrued Allows to Dilute Yield" is the Senior coupon accrual being synced into Senior principal so that the payment can be front-run. The defective quantity here is an outstanding trader VPI prepayment, and no coupon is involved in the headline redemption or in the deposit-side case. The impairment check in the deposit gate also predates that report, and what extended it to Junior deposits was a later change set titled "Harden junior-funded coupon deposits", which deleted the arm that had exempted Junior deposits from the check. Open Cecuro MED-24 ("LP can delay other redeemers with zero-payout refund epochs") is a bounded-work queue limitation and MED-29 ("Small redeemers can lose funded withdrawal value to the pool and seed account") is a tiny-share entitlement rounding case; neither concerns what the redemption price is composed of. The remaining open items are oracle price selection, oracle rotation, stale cached commit prices, carry rounding and the missing frozen-market stop-loss trigger, none of which touches tranche pricing. The known HIGH-1 clock alignment item requires a price the trader can see before committing; here the open and the close execute at the same mark and no price advantage is used.

The repository's own history carries a fix to a different quantity whose harm sentence reads close to this one, so it is worth separating. A commit titled "Fix C-03: _reconcile excludes transient funding cash from distributable" made `_reconcile` deduct a negative `netUnsettledFunding` before computing distributable cash, because without the deduction LPs could redeem at an inflated price at the expense of remaining depositors. That quantity does not exist at this commit. `netUnsettledFunding` was replaced hours later the same day by aggregate entry-funding tracking, in a change that deliberately made the funding spread "automatically distributable as LP revenue instead of being permanently locked as a ghost liability", and `netUnsettledFunding` returns no hit anywhere under `packages/perps/src` at `a82d4fef`. The reservation that fix mirrored does have a modern successor, `traderClaimBalanceAssets`, and it is deducted at this commit in both snapshots: `HousePoolAccountingLib.buildWithdrawalSnapshot` subtracts it at `:51-52` and `buildReconcileSnapshot` subtracts it at `:64`. The outstanding positive VPI accrual is a third quantity. It is held by the Engine on the position (`engine.positions(account)` returns it as `vpiAccrued`), the pool's input snapshot has no field for it (`src/interfaces/HousePoolEngineViewTypes.sol:25-36`), and it appears in neither of the two snapshots above. A sibling obligation bucket is reserved on exactly this path, and this one is not.

Scope: every file on the path sits in `packages/perps/src`, which the programme puts in scope wholesale, and three of them (`HousePool.sol`, `TrancheVault.sol`, `MarginClearinghouse.sol`) are listed asset rows of their own. None of the six excluded view-only contracts is on the path. Nothing in `packages/shared`, `packages/spot`, `packages/options`, `perps-aa`, `lib/`, the keepers, the oracle worker or the frontend is required, and the inactive bridge route is not touched. The Proof of Concept runs in the repository's own Foundry suite at the commit in scope and modifies no file under `packages/perps/src`. The two out-of-scope components the programme permits replacing, the Pyth price updates and the settlement token, are replaced by the repository's own mocks, and the Router is deployed through the repository's own out-of-scope test adapter, which adds no logic to the production commit path and is described in the Proof of Concept.

There is no path back once the redemption is claimed, and that is settled at the pool's and the clearinghouse's whole transfer-out surface rather than assumed. `HousePool.sweepExcess` moves at most `excessAssets()` and does not change `accountedAssets`, so it cannot touch accounted assets (`:585-597`). `assignUnassignedAssets` acts only on ownerless pool assets and moves no settlement token at all (`:660-704`), and it reverts outright while that balance is zero (`:673-676`). The pool's two other transfer-out sites are `payOut` (`:604-612`), callable only by the Engine or its settlement sidecar, and the redemption funding transfer to a tranche vault (`:1007`); neither moves value toward a departed LP. The only `rescueToken` in the tree belongs to the clearinghouse and reverts for the settlement asset (`MarginClearinghouse.sol:324-330`), which the project also documents as a recovery limitation at `SECURITY.md:1449-1454`. Neither vault reverses a funded and claimed redemption.

## References

Repository: `https://github.com/immunefi-team/audit-comp-plether`, at the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. All paths below are relative to `packages/perps`.

1. `src/libraries/OpenAccountingLib.sol` - `#L127-L135` (the VPI charge folded into trade cost)
2. `src/CfdEngineSettlementSidecar.sol` - `#L454-L464` (the charge booked as `Revenue` with `CashArrived`), `#L373-L379` (`_negativeVpiReserveTarget`, the reserve that exists only on the negative side), `#L621-L624` (the close rebate paid out of pool cash)
3. `src/libraries/HousePoolWaterfallAccountingLib.sol` - `#L140-L159` (`distributeRevenue`, Senior to its mark then all of the remainder to Junior), `#L167-L184` (`absorbLoss`, Junior first)
4. `src/HousePool.sol` - `#L835` and `#L840-L841` (the reconcile, then the Junior redemption price taken from the slot it credited), `#L957-L959` and `#L992-L1000` (the two quotes that consume `pricingPrincipal`), `#L809-L819` (the Router-only branch), `#L842-L843` and `#L881` (the free-cash budget the exit is funded from), `#L1147-L1155` (`_settlementFeeBps`, zero while the oracle is live), `#L624-L651` (`recordClaimantInflow`), `#L176` and `#L178` (minimum tranche deposit, LP epoch duration), `#L532-L536` (trading activation requires both seeds), `#L585-L597` and `#L660-L703` (the owner transfer-out surface, neither of which reaches an accounted asset), `#L604-L612` (`payOut`, reachable only by the Engine or its settlement sidecar), `#L1007` (the redemption funding transfer to a tranche vault)
5. `src/libraries/CloseAccountingLib.sol` - `#L112-L118` (the lifetime clamp, which makes the charge refundable in full)
6. `src/libraries/HousePoolAccountingLib.sol` - `#L46-L55` (`buildWithdrawalSnapshot`: the reserve carries no VPI term, and deducts `traderClaimBalanceAssets` at `#L51-L52`), `#L60-L78` (`buildReconcileSnapshot`: `distributable` carries no VPI term, and deducts `traderClaimBalanceAssets` at `#L64`); `src/interfaces/HousePoolEngineViewTypes.sol` - `#L25-L36` (the engine input snapshot both of them are built from, which has no VPI field)
7. `src/OrderRouter.sol` - `#L85-L90` (`commitOrder`), `#L240-L247` (`executeOrder`), `#L479-L484` (`settleLpEpoch`, permissionless and payable); `src/router/OrderLiquidationBatchLogic.sol` - `#L549-L581` (the delegated body that installs the mark and calls the pool in one frame)
8. `src/TrancheVault.sol` - `#L38` and `#L617` (the one hour deposit cooldown and its check), `#L562-L575` (`requestDeposit`), `#L606-L631` (`requestRedeem`), `#L968` (`claimDeposit`), `#L1014-L1021` and `#L1211` (`claimRedeem` and the settlement-token transfer out), `#L322` and `#L1630-L1637` (the seed floor that makes the seed holder permanent)
9. `src/CfdMath.sol` - `#L118-L140` (`tryGetSkewCost`, the quadratic charge curve); `src/libraries/CfdEnginePlanLib.sol` - `#L473-L480` (the skew cliff that bounds one event), `#L487` (the open-side depth input)
10. `src/MarginClearinghouse.sol` - `#L324-L330` (`rescueToken` excludes the settlement asset)
11. Project documents at the same commit: `ACCOUNTING_SPEC.md#L821-L822` (the VPI reserve rule), `#L1061-L1063` (close-time rebates are paid from free cash and the unfunded part is waived); `SECURITY.md#L866-L867` (the negative-side reserve), `#L859-L860` (new LPs do not inherit old trader obligations), `#L391` and `#L1438` (economic depth), `#L392` (entry and exit use the same pricing), `#L1384` (the nearest LP accounting limitation), `#L1449-L1454` (the settlement-token recovery limitation); `DEPLOYMENT.md#L131`, `#L133`, `#L142`, `#L143`, `#L152`, `#L154-L156` (the release parameters severity is assessed with)

File fingerprints at the frozen commit:

| File | Bytes | Lines | sha256 |
| --- | --- | --- | --- |
| `src/HousePool.sol` | 91,455 | 1,886 | `57cde380a49f5fa0607fa101e790e64f539b32a52da7baaae520daac0ee40159` |
| `src/CfdEngineSettlementSidecar.sol` | 44,567 | 928 | `5b9dd94454fdb8daeca9736e2c6ce750b2a61b13278ad0e2a8b9a2366b2c8d2f` |
| `src/libraries/CloseAccountingLib.sol` | 8,067 | 129 | `5d252ee0652cfc52259511daadaa54404e809c5486f434477a585720071796b9` |
| `src/libraries/HousePoolWaterfallAccountingLib.sol` | 9,406 | 186 | `83cc6cf276620f439d8c7500e43ca50370214c41304b7e26c9ee682274c477ab` |
| `src/libraries/OpenAccountingLib.sol` | 8,985 | 156 | `349933d658c9e9f8c21acd18bb4d4d9e67e29458bbaffcd0ca0c6d2f1134e7c0` |
| `src/libraries/HousePoolAccountingLib.sol` | 5,795 | 108 | `16859869089b962720926ab780257a6bdc82c2a5004518cc5b03cd8ce60d1c8d` |
| `src/TrancheVault.sol` | 72,328 | 1,875 | `6150a59fa6078881a2186f44100f6c0f066d70704a6b668f821a79ae1bfcb2f2` |
| `src/OrderRouter.sol` | 29,515 | 557 | `8f8318db0889f5d098d2f5c8c6afc41c473d1ca516817fe2363ba3899bb48d7d` |

## Link to Proof of Concept

https://gist.github.com/ChaseTheLight01/37bef92e689c4b65bd0f8b1a63b6ecda

## Proof of Concept

STATUS: EXECUTED 2026-11-15, RESULT GREEN. Both measurements pass against the audited source at the
pinned commit: `Suite result: ok. 2 passed; 0 failed`.

The test runs in process under `forge test` against a local checkout at the frozen commit, inside a
network namespace whose only interface is loopback, with nothing dialled, signed or broadcast. The
runner's own isolation self-check, that the only interface is `lo`, that no IPv4 and no IPv6 route
exists, and that outbound TCP connects are refused, and the eleven source fingerprints matching the
pinned commit, are the opening lines of `CAPTURE.log`. Every assertion in both measurements executed
and passed; forge prints nothing for a passing assertion, so the evidence each one held is that the
run reached the end of both tests and the suite is green. The figures behind the headline comparisons
are printed in the capture's `Logs:` blocks, and the capture is reproduced byte for byte in
`POC_EXECUTED_GREEN.md`. Runnable copies of the five files live in a secret Gist, linked in the
Proof of Concept section below; nothing in this report depends on opening it.

## Proof of Concept

One test file, in the repository's own Foundry suite at the frozen commit, extending the project's
own `test/perps/BasePerpTest.sol` and using its fixtures. No file under `packages/perps/src` is
modified. Solidity 0.8.35 with `via_ir = true` and `optimizer_runs = 200`, which is the
repository's own default profile in `packages/perps/foundry.toml`.

| Measurement | Enters at | Measures | Result |
| --- | --- | --- | --- |
| 1. `test_aJuniorLpRedeemsAtAPriceContainingARefundableTraderPrepayment` | `TrancheVault.requestDeposit`, `OrderRouter.settleLpEpoch`, `TrancheVault.claimDeposit`, `OrderRouter.commitOrder`, `OrderRouter.executeOrder`, `TrancheVault.requestRedeem`, `OrderRouter.settleLpEpoch`, `TrancheVault.claimRedeem`, in that order, with no caller substituted for any of them and the trader's two commits entering the production bounded commit through the repository's own test adapter | that the charge reaches Junior equity with nothing reserved against it, that the exit is priced on it, funded in full from free cash and paid out in settlement token, that the holder who stayed ends below its deposited principal, and that the leaver is better off exiting than holding, against a hold control in the same test | **PASS, 2026-11-15.** Both branches run from one snapshot and every assertion executes and passes. In the exit branch the leaver is paid 6,003,468 USDG, its 6,000,000 deposit plus its 3,468 share of the charge, which is 2,646 USDG more than the hold branch leaves it; the remaining holder ends at 3,997,898 USDG, 2,102 below its 4,000,000 deposit, and the leaver's gain and that holder's loss agree within the branch tolerance |
| 2. `test_aNewJuniorLpBuysInAtTheSamePriceAndFundsTheRefundAfterEntry` | the same entry points as measurement 1, with a Junior deposit request in place of the redemption | that a Junior LP that buys in while the charge is outstanding is priced on it and ends below its deposited principal, funding part of the refund after entry | **PASS, 2026-11-15.** Both branches run from one snapshot and every assertion executes and passes. The entrant deposits 2,000,000 USDG and ends at 1,999,426 USDG, 574 below its own deposit |

Measurement 1 carries the finding. Measurement 2 is the entry side of the same unreserved
prepayment, described in the Impact Details section.

This section sets out the attack steps and what the measurements do and do not establish, and it
carries the 2026-11-15 run's result lines, copied from the capture. A single
secret Gist
holds runnable copies of five files, so a reviewer who wants to execute it can: the test file
`VpiPrepaymentJuniorExit.t.sol`, the runner `run_local_only.sh`, the recipe and configuration table
`RUN.md`, the run record `POC_EXECUTED_GREEN.md`, and the captured output `CAPTURE.log`. The link is
https://gist.github.com/ChaseTheLight01/37bef92e689c4b65bd0f8b1a63b6ecda. Nothing below depends on
opening it.

### Where the test runs, and what it never touches

The test runs locally. It runs in process under `forge test` against a local checkout of the
audited source at the frozen commit, with the pool, both vaults, the engine, the router, the order
lifecycle book, the oracle and the clearinghouse all deployed inside the test by the project's own
`BasePerpTest` setup.

The shipped runner enters a network namespace itself (`unshare -rn`) and checks from inside it
before any test runs: that the only interface present is `lo`, that no IPv4 and no IPv6 route
exists, and that outbound TCP connects to two public addresses do not succeed. If any check fails
the runner prints the failure and exits without running anything, and those checks are the first
lines of `CAPTURE.log`. Forge runs with `FOUNDRY_OFFLINE=true`.

Nothing of the programme's is contacted. No node is started, no chain RPC is reached, no block
explorer, indexer, API or other system of the programme's is used, no key material of any kind is
present, nothing is signed, and there is no fork of any public network and no deployed address is
read.

The two in-process stand-ins are the two the programme permits replacing: the project's own
`MockPyth` supplies the price update blobs through the project's own `_mockPythUpdateData` helper,
and the project's own `MockSettlementToken` stands in for USDG. Everything else under test is the
audited source. There is no `vm.store`, no `stdstore`, no `checked_write` and no `vm.mockCall` in
the file, no address is substituted for the Router or the Engine, and no depth, skew or execution
price is supplied to a call that production derives for itself; the basket level reaches the
protocol only as a price update through the permitted Pyth stand-in, and the Engine derives the
mark from it.

The trader's two order commits enter through `LegacyOrderRouterHarness`
(`test/utils/LegacyOrderRouterHarness.sol`), the repository's own test adapter, which is the type
`BasePerpTest` declares for its Router. It translates a scalar commit into the production bounded
`OrderRouter.commitOrder(OrderRequest)` and reaches that function by self-delegatecall, preserving
the original account as `msg.sender` (`:38-40`, `:79-88`). The fields it fills are the
caller-authored execution limits of the production request; they are consumed as admission ceilings
and mode filters by the lifecycle book and the policy evaluator (`src/OrderLifecycleBook.sol:341-345`,
`src/CfdOrderPolicyEvaluatorBase.sol:172`), and none of them is an input to the VPI charge curve, to
the waterfall or to the redemption price. Besides a test marker constant the adapter adds nothing
but that one function, so every other call in the file is the production function, `updateMarkPrice`
and `settleLpEpoch` included (`src/OrderRouter.sol:429`, `:479-484`). It sits in the test tree, which the programme puts out of
scope, so no in-scope contract is modified. Eighty of the repository's own test files use that same
scalar commit form, the project's own VPI round-trip test among them
(`test/perps/accounting/close/VpiRoundTripAccounting.t.sol:41-59`).

### Measurement 1: the exit, driven end to end

Two branches from one snapshot:

- the **hold** branch, in which the Junior LP keeps its shares through the trader's close. This is
  the best legitimate alternative to the exit, not an idle baseline.
- the **exit** branch, in which it requests redemption while the position is open, the epoch that
  prices it is settled through the Router, and it claims.

The attacker's steps run through production entry points, and the trader's commit enters the
production bounded commit path through the repository's own test adapter (see "Where the test runs"
above):

1. Two Junior LPs call `TrancheVault.requestDeposit` on the Junior vault, both requests batch into
   one epoch window, a settlement pass activates the epoch, and each calls `claimDeposit`. Their
   shares are minted by the system's own creation path rather than handed to them. The tranche is
   asserted to hold the principal deposited into it at this point, and no charge, to within the one
   settlement token the permanent seed accrues.
2. One hour passes, which is the `DEPOSIT_COOLDOWN` the vault enforces at `TrancheVault.sol:617`.
   The test asserts `maxRequestRedeem` has become the whole balance.
3. A third party calls `OrderRouter.commitOrder` for a skew-increasing long, and an ordinary account
   with no role executes it with `OrderRouter.executeOrder(orderId, updateData)`. The VPI depth is
   whatever the Engine computes for itself; nothing hands it a depth, a mark or an execution price.
   The test reads the Engine's own record of the outstanding accrual on that position out of
   `engine.positions`, and asserts that Junior tranche equity rose by that whole amount.
4. The leaver calls `TrancheVault.requestRedeem` for its whole Junior balance, and the test asserts
   the request landed in the epoch both branches reach next.
5. At the epoch boundary the epoch is settled by `OrderRouter.settleLpEpoch(updateData)` from the
   same roleless account. The test asserts, after the warp into that epoch and before the call,
   that the position is still open, with nothing in between that can close one, so the settlement
   that prices the exit is the caller-restricted branch rather than the permissionless no-position
   one.
6. The leaver calls `TrancheVault.claimRedeem` and the settlement token lands in its own wallet.
7. The trader closes through `commitOrder` and `executeOrder` at the same mark, and one more
   settlement pass books the resulting shortfall.

A negative control runs beside step 5: the same roleless account calling `HousePool.settleLpEpoch`
directly, with the Engine's own installed mark and publish time, reverts with
`HousePool__Unauthorized`. That measures two things at once, that the branch really is
caller-restricted and that the Router entry is what opens it to any caller.

```solidity
// test_aJuniorLpRedeemsAtAPriceContainingARefundableTraderPrepayment, outer body, lines 455-495
assertGt(
    exit.leaverValue,
    hold.leaverValue,
    "the leaver is strictly better off redeeming before the close than holding the same shares through it"
);
assertLt(
    exit.stayerValue,
    hold.stayerValue,
    "the holder that stayed is strictly worse off than it is in the hold branch"
);

// The loss reaches deposited principal, not only accrued yield: over the whole tranche, and over the
// staying holder's own position. The hold branch is the control that shows the trade on its own does not
// do this.
assertLt(
    exit.juniorTrancheEquity,
    STAYER_DEPOSIT + RELEASE_TRANCHE_SEED,
    "the whole Junior tranche ends below the principal deposited into the shares that still exist"
);
assertLt(
    exit.juniorPrincipalStored,
    STAYER_DEPOSIT + RELEASE_TRANCHE_SEED,
    "Junior principal in pool storage ends below that principal too, once the pass books the shortfall"
);
assertLt(exit.stayerValue, STAYER_DEPOSIT, "the holder that stayed ends below its own deposited principal");
assertGt(
    hold.stayerValue, STAYER_DEPOSIT, "in the hold branch the same holder ends above its deposited principal"
);
assertGt(
    hold.juniorTrancheEquity,
    STAYER_DEPOSIT + LEAVER_DEPOSIT + RELEASE_TRANCHE_SEED,
    "in the hold branch the tranche ends above the whole principal deposited into it"
);

// Whose balance moves, and by how much.
assertApproxEqRel(
    leaverGain,
    stayerLoss,
    BRANCH_TOLERANCE,
    "the leaver's gain and the staying holder's loss agree inside the branch tolerance"
);
```

### Measurement 2: the entry, on the same unreserved charge

`HousePool._settleLpEpoch` prices the Junior deposit phase on the same principal the reconcile has
just credited the charge to (`:915`, the deposit phase, reading the same slot the redemption phase
reads at `:840-841`). So a Junior LP that buys in while the charge is outstanding enters at a NAV
built from a refund it will fund rather than receive.

Three things are measured, two branches from one snapshot. The main case lets a Junior LP enter
through `TrancheVault.requestDeposit` while the position is open, has a roleless account settle the
epoch that prices the entry, closes the position, and measures the entrant against its own deposit.
The first control is the branch with no entrant, which shows what the incumbent holders keep when
nobody else's cash is in the tranche. The clean measure is the entrant's own loss below its deposit;
the incumbent-side difference between the branches is larger than that loss, because the entrant's
deposit also deepens the pool and shrinks the trader's close rebate, a separate and already-known
depth effect, so the finding here rests on the entrant's sub-deposit loss.

```solidity
// test_aNewJuniorLpBuysInAtTheSamePriceAndFundsTheRefundAfterEntry, outer body, lines 707-721
assertLt(
    joined.entrantValue,
    ENTRANT_DEPOSIT,
    "the Junior LP that bought in while the charge was outstanding ends below its deposited principal"
);
assertGt(
    joined.incumbentValue,
    alone.incumbentValue,
    "the incumbent holders keep more when a new depositor's cash funds part of the refund"
);
assertGt(
    alone.incumbentValue,
    STAYER_DEPOSIT + LEAVER_DEPOSIT,
    "with no new depositor the incumbents fund the whole refund out of what the charge gave them"
);
```

### Attack steps: reproducing this by hand

The steps below are the production path. The measurements in this report run in process, in the
test, against a local checkout, with nothing dialled (see "Where the test runs" above).

Given a live pool with trading active, a Junior tranche with at least one other holder, Senior at
its high-water mark, and any trader with an open skew-increasing position:

1. **Join the tranche.** Call `requestDeposit(assets, controller, owner)` on the Junior
   `TrancheVault` for the size you choose, at least 1 USDG. Settle the deposit epoch with
   `OrderRouter.settleLpEpoch(updateData)`, then
   `claimDeposit(requestId, assets, receiver, controller)`. Nobody's permission is needed at any
   point, and this is the whole of the setup.
2. **Wait one hour.** The vault requires `DEPOSIT_COOLDOWN` between a deposit and a redeem request.
3. **Pick your moment.** Any open skew-increasing position has already paid a positive VPI charge
   into the Junior tranche's equity and can reclaim up to the whole accrual at its close. The
   accrual carries no time decay, so what you need is one LP epoch, an hour, falling inside that
   position's life. What erodes the Junior share of it meanwhile is the senior coupon, which every
   reconcile pays out of junior principal, and any loss the pool takes.
4. **Request the exit.** `requestRedeem(shares, controller, owner)` for your whole balance.
5. **Settle the epoch yourself.** `OrderRouter.settleLpEpoch{value: pythFee}(updateData)`. This
   installs the Engine's mark and calls the pool in the same frame, which is the only way the
   Router-only branch is reached while a position is open. Your redemption is priced on
   `juniorPrincipal` as it stands after the reconcile, including the charge.
6. **Claim.** `claimRedeem(requestId, shares, receiver, controller)`. Real settlement token leaves
   the vault for your wallet and your shares are burned.
7. **Let the close happen.** When the position closes at the same mark, the lifetime clamp returns
   up to the whole charge out of pool cash, and the next reconcile takes it from `juniorPrincipal`,
   which no longer contains your shares.

### The counterfactual

The counterfactual is built into the test rather than taken by patching the source. The hold branch
is the best legitimate alternative to the exit, both branches run from one snapshot, and the
comparison between them is the finding: the test asserts that the leaver ends ahead of where holding
would have left it, that the holder that stayed ends behind and below the principal it deposited,
and that in the hold branch the same holder ends slightly ahead of its deposited principal, which is
what the trade on its own does. Those comparisons sit in the outer test body (`:455-495`) and executed and passed on the 2026-11-15
run. Reserving the outstanding positive accrual, or
excluding it from the principal that prices LP entry and exit, moves both of those numbers.

The minimal form of that change belongs in `HousePool._settleLpEpoch`, at `:840-841` for the
redemption phase and `:915` for the deposit phase, and no patch ships with this report because that
contract has almost no room left. The repository's own size test holds `HousePool` runtime at or
below a measured 24,571 bytes against the 24,576 byte EIP-170 limit, at
`test/perps/gas/AtomicLpEpochSettlementGas.t.sol:39-44` against the target at
`test/perps/shared/AtomicLpEpochSettlementFixture.sol:101-103`. With that little headroom in the
current optimized build, a direct patch has to be size-tested and will likely need a refactor or a
move into a sidecar: the test establishes the current build's size rather than the byte cost of any
particular edit, and via-IR optimization is not monotonic in source length. Every form in the Bug
Description needs some code in `HousePool` itself, so the same constraint applies to all three: the
structural form and the minimal form both have to subtract the accrual from
`juniorPhase.pricingPrincipal` (`:840-841`) and from `juniorDepositPhase.pricingAssets` (`:915`),
and the booking-side form has to add a branch to `recordClaimantInflow`, whose kind dispatch and
bucket writes are both in that contract (`:644-648`, `:1794-1802`).

### What the proof of concept shows, and what follows from the code

The suite is green: `2 passed; 0 failed`. Every assertion
in both measurements executed and passed, so each row below is established by execution unless its
right-hand cell says it is argued from source. A row that is argued from source rather than measured
states a structural fact about the code that no single run turns on, and says so. The figures the
capture printed are in its `Logs:` blocks and reproduced in `POC_EXECUTED_GREEN.md`; line
references are to the test file as shipped.

| Claim | How it is established |
| --- | --- |
| A positive VPI charge from a skew-increasing open reaches Junior tranche equity in full | Measured, measurement 1: the Engine's own outstanding accrual on the position is read out of `engine.positions`, and Junior tranche equity before and after the open is asserted to differ by that whole amount, to within the two atoms of integer senior coupon the seconds of an order execution can accrue |
| **Nothing in pool accounting is held back against the refund the close can claim** | Measured, measurement 1, for the quantity the redemption price is taken from: Junior equity rises by the whole accrual, so the reconcile path that produces `pricingPrincipal` holds nothing back. The other two quantities are settled from source in the Bug Description: the withdrawal reserve's base is `maxLiabilityAssets + traderClaimBalanceAssets + supplementalReservedAssets`, with no VPI term (`HousePoolAccountingLib.sol:46-55`), and the pool only layers further reservations on top of that base, never fewer (`HousePool.sol:1727-1735`); and `vpiAccrued` appears in `TerminalNavBookV2.sol` only in a declared interface at `:28` and in none of its arithmetic |
| The Junior redemption is priced on that inflated principal | Measured, measurement 1: the exit quote is read before and after the charge arrives and asserted to have risen, and the claimed assets are asserted to exceed the leaver's own deposit |
| **The epoch that prices the exit is settled while the charge-bearing position is still open** | Measured, measurement 1, through `OrderRouter.settleLpEpoch` from a roleless account, with `hasOpenPositions` asserted true after the warp into the settling epoch and before the call, with nothing in between that can close a position, and with the direct `HousePool.settleLpEpoch` call by the same account asserted to revert `HousePool__Unauthorized` |
| **Each of the steps that sets up the state is individually valid through its production entry point, and valid in that order** | Measured, measurement 1: every step is a call to `TrancheVault` or `OrderRouter`, each returning its own id, status or settlement event, with no `prank` of the Router or the Engine anywhere in the file and no depth, mark or price argument supplied that production computes |
| **The exit is funded in full from free cash while that position is open, after `maxLiabilityAssets`, trader claims and the settlement buffer are reserved** | Measured, measurement 1: the pool's own settlement event reports the funded assets and no Junior backlog (`:596`), the funded share count is asserted equal to the whole requested balance (`:597-601`), the pending part of the request is asserted zero (`:602`), the pass is asserted to have funded exactly the entitlement it reported (`:604`), and that entitlement is asserted to equal the quote the inflated principal produces, to within the senior-coupon tolerance (`:608-613`). On the run the pass funded 6,003,468 USDG with no backlog. The quote here is read after the mark is refreshed, which is the state the pass itself installs before it reconciles; against a mark left stale by the warp the pre-pass view conservatively omits the charge |
| **Nothing on the exit path offsets the extraction** | Measured, measurement 1: the configured frozen Junior exit fee is read out of pool storage and asserted nonzero (`:223`), `frozenLpFeeBps(false)` is asserted zero while the oracle is live (`:578`), the redemption quote is asserted to carry no fee deduction (`:579-583`), and the claim is asserted to pay the whole entitlement with nothing withheld (`:620`). All passed on the 2026-11-15 run |
| **The leaver is strictly better off exiting than holding, and the holder that stayed is strictly worse off than in the hold control** | Measured, measurement 1: `assertGt` on the leaver's total value and `assertLt` on the staying holder's, against the hold branch (`:455-464`), both passed. Total value is settlement token held plus the vault's own redemption quote on shares still held, so the leaver's exit-branch figure is realized cash and every figure for a party still holding shares is that quote; the step from quote to cash is driven on the leaver and its legs passed (`:578-583`, `:608-613`, `:620`). On the run the leaver ends at 6,003,468 USDG against 6,000,822 USDG holding, and the staying holder at 3,997,898 USDG against 4,000,548 USDG |
| **The remaining holder's loss reaches deposited principal and not only accrued yield** | Measured, measurement 1, three ways: the whole Junior tranche's equity, taken over every share that exists rather than at one holder's slot, ends below the principal deposited into the shares that still exist; `juniorPrincipal` in pool storage ends below it too, once the pass after the close books the shortfall; and the staying holder's own position ends below its own deposit. The hold branch asserts the opposite for two of the three, the tranche's equity and the staying holder's own position, so the trade on its own does not do this |
| **The leaver's gain is the staying holder's loss** | Measured, measurement 1: the leaver's gain over the hold branch and the staying holder's loss against the same branch are asserted to agree within two percent (`:490-495`), and did, the gain 2,646 USDG beside the loss 2,650 USDG, agreeing to within 0.13 percent. The residual sources the two percent allows for are the integer share conversions, the one settlement token the permanent seed holds, any difference between the two branches' close rebates, which the clamp bounds by the accrual in both, and any difference in the senior coupon the two branches book. The exact identity of gain to escaped refund share is not asserted |
| **Permanence: the extracted settlement token leaves the protocol with no path back** | Measured, measurement 1, for the balance and for the two owner recovery paths the test drives, and argued from source for the third. The four balance legs on the leaver's own ERC-20 balance passed: zero before the claim (`:617`), equal to the claimed assets after it (`:620-622`), unchanged after the trader's close (`:638`), and unchanged after the settlement pass that books the shortfall against the holders that remain (`:653-658`). The two owner calls driven in the test also passed (`:664-677`): the clearinghouse `rescueToken` reverts for the settlement asset, and a sweep of the pool's whole unaccounted excess leaves both the leaver's balance and `juniorPrincipal` unchanged. The third, `assignUnassignedAssets`, is not called by the test and is argued from source: it reassigns the pool's own `unassignedAssets` into tranche principal and mints matching shares against assets the pool already holds, and performs no transfer of settlement token at all, which its body and its own documentation both state (`HousePool.sol:660-704`, `:656`). That is what carries this path, and it holds whatever the slot contains: a call that moves no settlement token cannot return a claimed redemption to anyone. The call also reverts `HousePool__NoUnassignedAssets` while that balance is zero (`:673-676`), behind an oracle-live and a mark-freshness guard that both run before the balance is read (`:669-670`) |
| The attack needs no depth manipulation, because the clamp bounds the refund by the accrual at any depth | Measured, measurement 1: the pool's net settlement-token outflow at the close is asserted to agree between the hold branch and the exit branch within two percent (`:501-506`), and did, 4,414 USDG against 4,410 USDG, a 3 USDG gap beside a 2,646 USDG transfer, although the exit branch has already paid the leaver out and so closes against a smaller pool. The bound on each close's own rebate is `CloseAccountingLib.sol:116-118`, read from source; it caps each branch rather than equating the two |
| The mechanism does not depend on a price move | Measured, measurement 1: the mark at the open and the mark at the close are read out of the Engine in both branches, and the three assertions that they are equal (`:507-509`) passed; both marks read 1.00000000 |
| A charge and a refund really occurred, so neither branch is vacuous | Measured, measurement 1: the outstanding accrual is asserted strictly positive (`:531`) and the pool's net outflow at the close is asserted strictly positive (`:635`) |
| The entry side: a Junior LP that buys in while the charge is outstanding funds the refund after entry | Measured, measurement 2, with both of its controls in the same test. The entrant deposits 2,000,000 USDG and ends at 1,999,426 USDG, 574 below its own deposit (`:707-711`), which is the clean measure this case rests on. The pre-pass read, that with the mark refreshed the tranche the entrant is priced against already contains the charge, passed (`:751-757`); against a mark left stale by the warp that pre-pass view conservatively omits the charge, which is why the read refreshes first, the state the pass itself installs before it reconciles (`HousePool.sol:835`, `:915`). The incumbent-side comparisons (`:712-721`) also passed, but part of the incumbent gain is the trader's smaller close rebate against the deeper pool the entrant's deposit creates, a separate and already-known depth effect, so the finding rests on the entrant's sub-deposit loss rather than the incumbent-side figure |
| Every parameter the measurements depend on is at its committed release value | Measured: twenty-three parameters are read back out of protocol storage in `setUp` and asserted, including the two senior limits installed through the owner's own timelocked configuration path, plus assertions that trading is active and that the oracle is neither frozen nor in the funding-adjusted window. Three rows diverge, all inert here: `minOpenNotionalAssets`, a minimum the opened notional clears by three orders of magnitude; the oracle's adverse confidence multiplier, which both pricing sites that apply it multiply by a confidence the fixture's price updates set to zero; and the Junior maintenance fee recipient, which is never paid because `juniorMaintenanceFeeAprBps` is asserted zero. The full table is in `RUN.md` |
| One event's charge is bounded by `vpiFactor * maxSkewRatio^2 / 2`, which is 8 basis points of economic depth at the release parameters | Argued from source: the charge curve is `cost = vpiFactor * skew^2 / depth / 2` (`CfdMath.sol:118-140`) and the skew cliff rejects an open whose post-open skew exceeds `maxSkewRatio` of depth without strictly reducing it (`libraries/CfdEnginePlanLib.sol:473-480`), and the depth both of them run on is the Engine's economic depth, physical assets net of outstanding trader claims (`libraries/CfdEnginePlanLib.sol:449`, `:487`, `libraries/PositionRiskAccountingLib.sol:21-26`). That the open sits inside the cliff is established by the production planner admitting it. Eight basis points bounds the charge, not the extraction: the attacker's realized take is its tranche share of that charge and so is smaller, 2,646 USDG against the 5,780 USDG charge on the run |
| The Engine's own accrual on the open is inside 8 basis points of the pool's total assets at the open | Measured, measurement 1: the ceiling is computed from the release parameters and `pool.totalAssets()` read at the open, and the Engine's own accrual is asserted to be inside it. The cliff comparison beside it is taken on that same gross figure. Total assets is at or above economic depth, so this is the looser of the two ceilings and it does not establish the row above |
| A co-holder exists on any live pool | Argued, from protocol invariant rather than from a survey: trading cannot be activated before both seeds are initialized and seed shares cannot be redeemed below the floor. The test creates its own co-holder through the ordinary deposit path, so nothing here rests on a survey of holders |

### What this proof of concept does not establish

None of what follows sits under the impact entry this report files on.

1. **A repeated cycle.** Both measurements take one charge, one exit or one entry, and one close.
   Re-entering after the close and repeating the cycle would pay a fresh deposit cooldown and sit
   through another epoch, and the net result over several cycles is not measured here.
2. **The state of any live pool.** The tests create the tranche, the holders and the position they
   attack through the project's own entry points. They do not survey a deployment for an existing
   one, and nothing in the report rests on such a survey.
3. **The behaviour of the Pyth price feeds themselves.** The price update blobs come from the
   repository's own mock, which is one of the two components the programme permits replacing.
   Neither measurement uses a price movement, so the content of a real blob does not bear on either.

### The capture

`CAPTURE.log` holds the runner's own metadata and the filtered test stream, which is what the runner
writes into it, and `POC_EXECUTED_GREEN.md` reproduces that file byte for byte. The run's result
lines, copied from the capture:

```text
Ran 2 tests for test/perps/vpi_prepayment/VpiPrepaymentJuniorExit.t.sol:VpiPrepaymentJuniorExitTest
[PASS] test_aJuniorLpRedeemsAtAPriceContainingARefundableTraderPrepayment() (gas: 12731648)
[PASS] test_aNewJuniorLpBuysInAtTheSamePriceAndFundsTheRefundAfterEntry() (gas: 12274179)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 62.81ms (53.73ms CPU time)
```

Placing the file, from the repository root:

```bash
cp VpiPrepaymentJuniorExit.t.sol packages/perps/test/perps/vpi_prepayment/
cd packages/perps
forge test --threads 1 --match-path 'test/perps/vpi_prepayment/*.t.sol' -vv
```
