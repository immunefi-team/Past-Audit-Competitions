# 57023 sc high global earmark not reduced in forcerepay lets redeem over burn global debt cross account leakage protocol insolvency&#x20;

**Submitted on Oct 22nd 2025 at 18:22:57 UTC by @edantes for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57023
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Protocol insolvency

## Description

## Brief/Intro

When a weakly collateralized account with earmarked debt is liquidated, the liquidation path calls \_forceRepay(accountId, account.earmarked). This clears the account’s earmark but does not decrement the global cumulativeEarmarked. The next redeem() then consumes the stale global earmark, reducing totalDebt again, even though the earmarked amount was already consumed during liquidation.

Result: totalDebt drops below the sum of per-account debts, corrupting global accounting and enabling cross-account value leakage. Over time this can induce protocol insolvency and/or deplete available tokens.

## Vulnerability Details

**Root cause**

\_forceRepay() updates the account-level earmarked but does not decrement the global cumulativeEarmarked. In contrast, repay() reduces both account-level and global earmarks, keeping them in sync.

Excerpt (current pattern)

```solidity
// _forceRepay(...)
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;     // per-account cleared
// BUG: global cumulativeEarmarked is NOT reduced here
```

Reference (correct pattern in user repay path)

```solidity
// repay(...)
account.earmarked -= earmarkToRemove;
uint256 paidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= paidGlobal;        // global kept in sync
```

## Impact Details

* **Protocol insolvency:** Because redeem() acts on a stale global earmark, it can drive totalDebt below the sum of user debts (the system “believes” all debt is repaid while some users still owe). This breaks core solvency/accounting assumptions and can cascade into incorrect redemptions and collateral accounting.
* **Unable to operate due to lack of token funds:** redeem() transfers MYT to the Transmuter based on the inflated global earmark. Over time this can deplete MYT from the Alchemist and cause subsequent redemptions/repays/withdrawals to fail or to behave unpredictably.

**Risk Breakdown**

**Critical:** The bug enables double application of the same earmark (once in liquidation via \_forceRepay, again in redeem()), which understates totalDebt relative to actual user debts. Subsequent state transitions (redemptions, liquidations, withdrawals) operate on a corrupted ledger, leading to lossy, cross-account misallocation and potential insolvency.

## Recommendation

Mirror the repay() logic inside \_forceRepay() to keep global/account state consistent:

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {

    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    // FIX: keep global in sync with the account-level deduction
    uint256 paidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
    cumulativeEarmarked -= paidGlobal;

    // ... continue with existing collateral adjustments ...
}
```

## Proof of Concept

## Proof of Concept

A Foundry test demonstrates the invariant break caused by the stale global earmark.

How to run:

```bash
# from repo
forge test --match-path src/test/PoC_EarmarkForceRepay_OverstatedGlobal.t.sol -vvv
```

File: src/test/PoC\_EarmarkForceRepay\_OverstatedGlobal.t.sol

```solidity
// PoC_EarmarkForceRepay_OverstatedGlobal.t.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import {AlchemistV3} from "src/AlchemistV3.sol";
import {IAlchemistV3, AlchemistInitializationParams} from "src/interfaces/IAlchemistV3.sol";
import {ERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract MockDebtToken is ERC20 {
    constructor() ERC20("alUSD", "alUSD") {}
    function mint(address to, uint256 amount) external returns (bool) { _mint(to, amount); return true; }
    function burnFrom(address from, uint256 amount) external { _spendAllowance(from, msg.sender, amount); _burn(from, amount); }
}

contract MockVaultV2 is ERC20 {
    uint256 public rate = 1e18;
    constructor() ERC20("MockYieldToken", "MYT") {}
    function setRate(uint256 newRate) external { rate = newRate; }
    function convertToAssets(uint256 shares) external view returns (uint256) { return shares * rate / 1e18; }
    function convertToShares(uint256 assets) external view returns (uint256) { require(rate != 0, "rate=0"); return assets * 1e18 / rate; }
    function mintTo(address to, uint256 amount) external { _mint(to, amount); }
}

contract MockTransmuter {
    uint256 public fixedQueryAmount;
    function setFixed(uint256 amount) external { fixedQueryAmount = amount; }
    function queryGraph(uint256, uint256) external returns (uint256) { return fixedQueryAmount; }
    function totalLocked() external pure returns (uint256) { return 0; }
}

contract MockPositionNFT {
    uint256 public lastId;
    mapping(uint256 => address) private _owners;
    function mint(address to) external returns (uint256 tokenId) { tokenId = ++lastId; _owners[tokenId] = to; }
    function ownerOf(uint256 tokenId) external view returns (address) { address o = _owners[tokenId]; require(o != address(0), "no owner"); return o; }
}

contract PoC_EarmarkForceRepay_OverstatedGlobal is Test {
    AlchemistV3 internal alch;
    AlchemistV3 internal impl;
    MockDebtToken internal debt;
    MockVaultV2  internal myt;
    MockTransmuter internal transm;
    MockPositionNFT internal posNFT;

    address internal admin = address(this);
    address internal protocolFeeReceiver;
    address internal alice;
    address internal bob;
    address internal liquidator;

    uint256 constant MIN_COLL = 1.5e18;
    uint256 constant LWR_BND  = 1.2e18;
    uint256 constant GLOBAL_MIN_COLL = MIN_COLL;
    uint256 constant DEPOSIT_CAP = 1e27;

    function setUp() public {
        protocolFeeReceiver = makeAddr("fee");
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        liquidator = makeAddr("liquidator");

        debt   = new MockDebtToken();
        myt    = new MockVaultV2();
        transm = new MockTransmuter();
        posNFT = new MockPositionNFT();

        impl = new AlchemistV3();

        AlchemistInitializationParams memory p;
        p.debtToken = address(debt);
        p.underlyingToken = address(debt);
        p.depositCap = DEPOSIT_CAP;
        p.minimumCollateralization = MIN_COLL;
        p.globalMinimumCollateralization = GLOBAL_MIN_COLL;
        p.collateralizationLowerBound = LWR_BND;
        p.admin = admin;
        p.transmuter = address(transm);
        p.protocolFee = 0;
        p.protocolFeeReceiver = protocolFeeReceiver;
        p.liquidatorFee = 0;
        p.repaymentFee  = 0;
        p.myt = address(myt);

        bytes memory data = abi.encodeWithSelector(AlchemistV3.initialize.selector, p);
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), data);
        alch = AlchemistV3(address(proxy));

        alch.setAlchemistPositionNFT(address(posNFT));
    }

    function _depositAndMint(address user, uint256 shares, uint256 debtAmount) internal returns (uint256 tokenId) {
        myt.mintTo(user, shares);
        vm.startPrank(user);
        myt.approve(address(alch), type(uint256).max);
        alch.deposit(shares, user, 0);
        tokenId = posNFT.lastId();
        alch.mint(tokenId, debtAmount, user);
        vm.stopPrank();
    }

    function test_GlobalEarmark_NotReduced_OnForceRepay_BreaksGlobalDebtInvariant() public {
        // 1) Both borrowers exist BEFORE earmark and mint equal debt
        uint256 tA = _depositAndMint(alice, 100e18, 60e18);
        uint256 tB = _depositAndMint(bob,   100e18, 60e18);

        // 2) Earmark 60 global debt (e.g., 30 for A and 30 for B)
        transm.setFixed(60e18);
        vm.roll(block.number + 1);
        alch.poke(tA); // performs _earmark
        alch.poke(tB); // only _sync for B in same block; assigns B's earmark share

        // sanity: both have earmark and cumulativeEarmarked == 60
        (, , uint256 aEar0) = alch.getCDP(tA);
        (, , uint256 bEar0) = alch.getCDP(tB);
        uint256 cum0 = alch.cumulativeEarmarked();
        console2.log("[pre] A.earmarked =", aEar0);
        console2.log("[pre] B.earmarked =", bEar0);
        console2.log("[pre] cumulativeEarmarked =", cum0);
        assertGt(aEar0, 0);
        assertGt(bEar0, 0);
        assertEq(cum0, 60e18);

        // 3) Make A undercollateralized and liquidate(A)
        myt.setRate(0.6e18);
        transm.setFixed(0);
        vm.roll(block.number + 1);

        vm.prank(liquidator);
        alch.liquidate(tA);

        // After forceRepay: A earmark cleared but global cumulativeEarmarked STALE (bug)
        (, , uint256 aEar1) = alch.getCDP(tA);
        (, , uint256 bEar1) = alch.getCDP(tB);
        uint256 cum1 = alch.cumulativeEarmarked();
        console2.log("[post-liquidate] A.earmarked =", aEar1);
        console2.log("[post-liquidate] B.earmarked =", bEar1);
        console2.log("[post-liquidate] cumulativeEarmarked (STALE) =", cum1);
        assertEq(aEar1, 0, "A earmark should be cleared");
        assertGt(cum1, 0, "global earmark left stale");

        // Record current per-account debts and global totalDebt before redeem
        (, uint256 aDebtPre, ) = alch.getCDP(tA);
        (, uint256 bDebtPre, ) = alch.getCDP(tB);
        uint256 totalDebtPre = alch.totalDebt();

        // 4) Redeem the (stale) global earmark; this reduces global totalDebt again
        vm.prank(address(transm));
        alch.redeem(cum1);

        // Apply accounting to accounts
        alch.poke(tA);
        alch.poke(tB);

        (, uint256 aDebtPost, ) = alch.getCDP(tA);
        (, uint256 bDebtPost, ) = alch.getCDP(tB);
        uint256 totalDebtPost = alch.totalDebt();

        console2.log("[after redeem] A.debt =", aDebtPost);
        console2.log("[after redeem] B.debt =", bDebtPost);
        console2.log("[after redeem] totalDebt =", totalDebtPost);

        // B should get some reduction (he had an earmark)
        assertLt(bDebtPost, bDebtPre, "B should get a redemption reduction");

        // The global totalDebt was reduced by the full stale amount, not just B's share.
        // This breaks the invariant: sum(user debts) == totalDebt.
        uint256 sumDebts = aDebtPost + bDebtPost;
        console2.log("[after redeem] sumDebts =", sumDebts);
        assertLt(totalDebtPost, sumDebts, "Global totalDebt fell below the sum of account debts (corrupted by stale earmark)");
    }
}
```
