# 57102 sc high tvl overstatement from mytsharesdeposited desync enables softened liquidations no haircut over redemptions transmuter&#x20;

## #57102 \[SC-High] TVL Overstatement from \_mytSharesDeposited Desync Enables Softened Liquidations & No‑Haircut Over‑Redemptions (Transmuter)

**Submitted on Oct 23rd 2025 at 13:37:34 UTC by @cmds for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57102
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed royalties

### Description

### Brief/Intro

The global TVL counter \_mytSharesDeposited is not decremented on two outflow paths—\_forceRepay and \_doLiquidation. As a result, getTotalUnderlyingValue() overstates TVL, which softens liquidations and lets Transmuter.claimRedemption pay 1:1 without haircut when a haircut is due, enabling over‑redemption and amplifying insolvency risk. getTotalUnderlyingValue() exposes the internal \_getTotalUnderlyingValue() (derived from \_mytSharesDeposited), and claimRedemption’s denominator directly includes Alchemist TVL.

### Vulnerability Details

1.TVL derives from \_mytSharesDeposited: the public TVL Overstatement from ... \_getTotalUnderlyingValue(), which converts \_mytSharesDeposited to underlying value.

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    return yieldTokenTVLInUnderlying;
}
```

2.That TVL feeds the global collateralization used in liquidation math via normalizeUnderlyingTokensToDebt(\_getTotalUnderlyingValue()) \* FIXED\_POINT\_SCALAR / totalDebt.

```solidity
 (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );
}
```

3.Transmuter.claimRedemption’s haircut ratio denominator includes

```solidity
uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

       if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

The on-book TVL (denominator) is inflated → the bad debt ratio is suppressed → the reduction mechanism is often not triggered, leading to 1:1 distribution.

4.Inconsistent accounting (root cause):

— \_forceRepay transfers creditToYield to the Transmuter and protocol fee to the fee receiver without decrementing \_mytSharesDeposited.

```solidity
   if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
```

— \_doLiquidation transfers amountLiquidated - feeInYield to the Transmuter (and possibly the fee to liquidator) yet also does not decrement \_mytSharesDeposited.

```solidity
         TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

```

Conversely, other outflow paths do decrement it: withdraw reduces \_mytSharesDeposited by amount, and redeem reduces it by redeemed amount plus fee.

```solidity
    TokenUtils.safeTransfer(myt, recipient, amount);
        _mytSharesDeposited -= amount;
```

```solidity
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;
```

Similarly, this causes an overestimation of TVL. Reference comparison: Functions such as withdraw() and redeem() correctly decrease \_mytSharesDeposited when transferring out. In contrast, repay() only decreases the “fee” portion, since the principal is directly transferred by the user to the transmuter, bypassing the contract. Therefore, only the two aforementioned paths break the consistency.

5.Net effect: when collateral is moved to the Transmuter but \_mytSharesDeposited stays unchanged, Alchemist-side TVL remains inflated while the same tokens are also counted on the Transmuter side (yieldTokenBalance)—a denominator “double‑count” that suppresses the bad‑debt ratio and can skip haircuts.

### Impact Details

1.Softer/delayed liquidations: overstated global collateralization reduces liquidation amounts or defers them, slowing bad‑debt resolution.

2.No‑haircut over‑redemptions: the denominator inflation can push the ratio below the haircut threshold, enabling 1:1 payouts while under‑collateralized, draining value from other participants and widening the shortfall.

3.Severity mapping: aligns with “Protocol insolvency / Theft of unclaimed yield” (at least High; potentially Critical where predictable 1:1 over‑redemption constitutes direct value extraction).

eg: true state Alchemist=90, Transmuter=10, issuance=102 ⇒ true ratio=102/100=1.02 (haircut required); with the bug Alchemist still records 100 and Transmuter has 10 ⇒ denominator 110, ratio≈102/110=0.927 (no haircut, 1:1 payout).

### References

<https://github.com/elminnyc99/contest\\_15\\_10/blob/18085d275acf0fc53d6e063931fb573ed7a6661a/src/AlchemistV3.sol#L1240-L1242>

<https://github.com/elminnyc99/contest\\_15\\_10/blob/18085d275acf0fc53d6e063931fb573ed7a6661a/src/AlchemistV3.sol#L860-L867>

<https://github.com/elminnyc99/contest\\_15\\_10/blob/18085d275acf0fc53d6e063931fb573ed7a6661a/src/Transmuter.sol#L217-L226>

<https://github.com/elminnyc99/contest\\_15\\_10/blob/18085d275acf0fc53d6e063931fb573ed7a6661a/src/AlchemistV3.sol#L779-L783>

<https://github.com/elminnyc99/contest\\_15\\_10/blob/18085d275acf0fc53d6e063931fb573ed7a6661a/src/AlchemistV3.sol#L877-L882>

### Proof of Concept

### Proof of Concept

```language
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import { AlchemistV3, AlchemistInitializationParams } from "src/AlchemistV3.sol";
import { AlchemistV3Position } from "src/AlchemistV3Position.sol";
import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/// ---------------- Minimal mocks ----------------

/// @dev Simplified: MYT is an ERC20 and also exposes a 1:1 shares<->assets conversion.
contract MockMYT is ERC20 {
    constructor() ERC20("Mock MYT", "mMYT") {}
    function mint(address to, uint256 amt) external { _mint(to, amt); }
    function convertToAssets(uint256 shares) external pure returns (uint256) { return shares; }
    function convertToShares(uint256 assets) external pure returns (uint256) { return assets; }
}

/// @dev Debt token: Alchemist mints/burns this via TokenUtils.safeMint/safeBurn (simulated here).
contract MockDebtToken is ERC20 {
    address public minter;
    constructor() ERC20("Mock aAsset", "aAsset") {}
    function setMinter(address m) external { minter = m; }
    function mint(address to, uint256 amt) external returns (bool) {
        require(msg.sender == minter, "not minter"); _mint(to, amt); return true;
    }
    function burn(address from, uint256 amt) external returns (bool) {
        require(msg.sender == minter, "not minter"); _burn(from, amt); return true;
    }
}

/// @dev Minimal Transmuter: only needs to accept MYT transfers.
contract MockTransmuter {
    string public constant version = "3.0.0";
    address public immutable syntheticToken;
    address public protocolFeeReceiver;
    constructor(address _syntheticToken) { syntheticToken = _syntheticToken; protocolFeeReceiver = address(this); }
    function totalLocked() external pure returns (uint256) { return 0; }
}

/// ---------------- Test ----------------

contract TVLDesync_Liquidation is Test {
    /// The repo currently exhibits "TVL does not drop after liquidation" (desync), so keep this true to assert the bug.
    /// If your code is fixed and you want to assert "TVL drops after liquidation", set this to false.
    bool constant EXPECT_DESYNC = true;

    AlchemistV3 public alc;                // proxy instance
    AlchemistV3Position public pos;
    MockMYT public myt;
    MockDebtToken public debt;
    MockTransmuter public trans;

    address alice = address(0xA11CE);
    bytes32 constant TRANSFER_TOPIC = keccak256("Transfer(address,address,uint256)");

    function setUp() public {
        // 1) Deploy the implementation
        AlchemistV3 impl = new AlchemistV3();

        // 2) Dependencies
        myt  = new MockMYT();
        debt = new MockDebtToken();
        trans = new MockTransmuter(address(debt));

        // 3) Initialization params (match the ABI; use explicit numeric literals)
        AlchemistInitializationParams memory p;
        p.debtToken                       = address(debt);
        p.underlyingToken                 = address(myt);
        p.depositCap                      = type(uint256).max;
        p.minimumCollateralization        = 1_500_000_000_000_000_000; // 1.5e18
        p.globalMinimumCollateralization  = 1_500_000_000_000_000_000; // 1.5e18
        p.collateralizationLowerBound     = 1_300_000_000_000_000_000; // 1.3e18
        p.admin                           = address(this);
        p.transmuter                      = address(trans);
        p.protocolFee                     = 0;
        p.protocolFeeReceiver             = address(this);
        p.liquidatorFee                   = 0;
        p.repaymentFee                    = 0;
        p.myt                             = address(myt);

        // 4) Proxy + constructor-time initialize (avoid InvalidInitialization)
        bytes memory initData = abi.encodeWithSelector(AlchemistV3.initialize.selector, p);
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        alc = AlchemistV3(address(proxy));

        // 5) Position NFT contract & wire it up
        pos = new AlchemistV3Position(address(alc));
        alc.setAlchemistPositionNFT(address(pos));

        // 6) Allow Alchemist to mint/burn the debt token
        debt.setMinter(address(alc));

        // 7) Fund and approve
        myt.mint(alice, 1_000e18);
        vm.startPrank(alice);
        IERC20(address(myt)).approve(address(alc), type(uint256).max);
        vm.stopPrank();
    }

    /// Read the latest tokenId from the NFT Transfer (mint) event.
    function _lastMintedTokenId(address owner) internal returns (uint256) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i = logs.length; i > 0; i--) {
            Vm.Log memory L = logs[i-1];
            if (L.emitter == address(pos) && L.topics.length == 4 && L.topics[0] == TRANSFER_TOPIC) {
                address from = address(uint160(uint256(L.topics[1])));
                address to   = address(uint160(uint256(L.topics[2])));
                if (from == address(0) && to == owner) {
                    return uint256(L.topics[3]);
                }
            }
        }
        return 0;
    }

    /// Reproduction: after liquidation the actual balance is 0, but getTotalUnderlyingValue() still reads 100 (TVL desync).
    function test_TVLDesync_AfterLiquidation() public {
        // 0) Initial
        uint256 tvl0 = alc.getTotalUnderlyingValue();
        assertEq(tvl0, 0, "initial underlying TVL should be 0");

        // 1) Deposit 100 (returns shares, not tokenId)
        vm.startPrank(alice);
        vm.recordLogs();
        uint256 sharesOut = alc.deposit(100e18, alice, 0);
        uint256 tokenId = _lastMintedTokenId(alice);
        vm.stopPrank();
        require(tokenId != 0, "no tokenId minted");
        assertEq(sharesOut, 100e18, "deposit shares should be 100e18");

        // After deposit, actual balance matches reported TVL
        assertEq(IERC20(address(myt)).balanceOf(address(alc)), 100e18, "real balance after deposit");
        assertEq(alc.getTotalUnderlyingValue(), 100e18, "recorded TVL after deposit");

        // 2) Borrow up to the max (keep 0.1% slack) to avoid Undercollateralized revert
        uint256 max1 = alc.getMaxBorrowable(tokenId);
        require(max1 > 0, "max borrow is zero after deposit");
        uint256 borrow1 = max1 * 999 / 1000;
        if (borrow1 == 0) borrow1 = max1;
        vm.prank(alice);
        alc.mint(tokenId, borrow1, alice);

        // 3) Relax MCR to 1.0, then borrow the rest (target CR ≈ 1.0 so it falls below lowerBound=1.3 and becomes liquidatable)
        alc.setMinimumCollateralization(1e18);
        alc.setGlobalMinimumCollateralization(1e18);

        uint256 max2 = alc.getMaxBorrowable(tokenId);
        if (max2 > 0) {
            vm.prank(alice);
            alc.mint(tokenId, max2, alice);
        }

        // 4) Pre-liquidation snapshot
        uint256 balBefore = IERC20(address(myt)).balanceOf(address(alc));
        uint256 tvlBefore = alc.getTotalUnderlyingValue();
        assertEq(balBefore, 100e18, "pre-liq real balance should still be 100e18");
        assertEq(tvlBefore, 100e18, "pre-liq recorded TVL");

        // 5) Liquidation (current implementation transfers 100 MYT to the transmuter but does not update the accounting counter)
        (uint256 liqAmt,,) = alc.liquidate(tokenId);
        uint256 balAfter = IERC20(address(myt)).balanceOf(address(alc));
        assertEq(balBefore - balAfter, liqAmt, "real balance delta == liquidationAmount");
        assertEq(balAfter, 0, "real balance should be 0 after liquidation");

        // 6) Key check: does the reported TVL drop in sync?
        uint256 tvlAfter = alc.getTotalUnderlyingValue();

        if (EXPECT_DESYNC) {
            // Bug present: reported TVL did not decrease (= 100e18)
            assertEq(tvlAfter, tvlBefore, "getTotalUnderlyingValue should have decreased but didn't (TVL desync)");
        } else {
            // If fixed, TVL should drop to 0
            assertLt(tvlAfter, tvlBefore, "TVL should drop after liquidation (fixed behavior)");
            assertEq(tvlAfter, 0, "TVL after liquidation should be 0 when fixed");
        }
    }
}

```

## run:

```language
forge test -vvv --match-test test_TVLDesync_AfterLiquidation
```

## Logs：

```language
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/TVLDesync_Liquidation.t.sol:TVLDesync_Liquidation
[PASS] test_TVLDesync_AfterLiquidation() (gas: 695968)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.72ms (3.43ms CPU time)

Ran 1 test suite in 12.23ms (4.72ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## Step‑by‑step

goal: Show that liquidation/forced‑repay can move MYT out of Alchemist without synchronizing the global counter that backs TVL, i.e., \_mytSharesDeposited. This overstates getTotalUnderlyingValue() (TVL), which is then used in liquidation math and in Transmuter.claimRedemption’s bad‑debt denominator—leading to under‑discounted (1:1) redemptions. getTotalUnderlyingValue() proxies to \_getTotalUnderlyingValue() (driven by \_mytSharesDeposited). Liquidation math reads \_getTotalUnderlyingValue() for global normalization. claimRedemption() computes the denominator as alchemist.getTotalUnderlyingValue() + convert(yieldTokenBalance).

Setup:

1.Configure convertToAssets(1e18)=1e18, minimumCollateralization=150%, collateralizationLowerBound=120%.

2.Deploy and wire AlchemistV3 + Transmuter. Note: getTotalUnderlyingValue() uses \_mytSharesDeposited (book TVL), while getTotalDeposited() reads the actual MYT balance.

Steps

1.Open position: User A calls deposit(100e18, A, 0), then mint(tokenIdA, 60e18, A) (\~166% CR).

2.Trigger liquidation: Push A below collateralizationLowerBound (e.g., lower PPS or increase debt) and have any user B call liquidate(tokenIdA):

Effect: MYT is sent out to transmuter/liquidator, but \_mytSharesDeposited is not decreased.

3.Observe divergence:

getTotalDeposited() ≈ 50e18 (real balance). getTotalUnderlyingValue() still ≈ 100e18 (based on stale \_mytSharesDeposited) → real < book TVL.

4.Over‑redemption: User C calls claimRedemption(id) on Transmuter: Since the denominator includes alchemist.getTotalUnderlyingValue(), the inflated TVL yields badDebtRatio ≤ 1 → no haircut, payout at 1:1, draining collateral beyond the true capacity.

5.Repeat: Repeat steps 2–4 to keep extracting during the “no‑haircut” window, compounding insolvency.

## Pass Criteria:

1.You observe getTotalDeposited() ≪ getTotalUnderlyingValue();

2.claimRedemption pays 1:1 where a haircut should have applied (total redeemed exceeds true collateral support).

## fix

1.In after transferring out the underlying (to the transmuter and/or the fee receiver), decrease the global counter to match what actually left the contract:\_forceRepay()

```language
// After tokens are moved out
_mytSharesDeposited -= creditToYield;        // principal portion
_mytSharesDeposited -= protocolFeeTotal;     // protocol fee portion (if > 0)
```

2.In after sending out the liquidated underlying, decrease the counter by the amount that left the contract:\_doLiquidation()

```language
// After transfers to transmuter and (optionally) liquidator
_mytSharesDeposited -= amountLiquidated;     // split internally by actual transfers (net + any fee in underlying)
```
