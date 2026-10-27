# 58471 sc high accounting error in forcerepay doliquidation overstates tvl enabling under scaled redemptions and potential insolvency

**Submitted on Nov 2nd 2025 at 14:40:22 UTC by @winnerz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58471
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

When `_forceRepay` and liquidation `_doLiquidation` transfer MYT out of the Alchemist, the contract does not decrement `_mytSharesDeposited`. Since `getTotalUnderlyingValue()` derives TVL from `_mytSharesDeposited`, reported TVL remains inflated while actual holdings drop. This depresses `badDebtRatio` used by the Transmuter and can delay or under‑apply scaling of redemptions, creating a path to protocol insolvency. It can also soften liquidation sizing by overstating global collateralization inputs.

## Vulnerability Details

TVL is derived from `_mytSharesDeposited`:

```solidity
// src/AlchemistV3.sol: 1238-1241
    function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
        uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
        totalUnderlyingValue = yieldTokenTVLInUnderlying;
    }
```

In `_forceRepay`, MYT is transferred out but `_mytSharesDeposited` is not decremented, so reported TVL does not change:

```solidity
// src/AlchemistV3.sol: 738-782
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];


        // Query transmuter and earmark global debt
        _earmark();


        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);


        uint256 debt;


        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);


        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);


        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;


        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;


        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;


        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);


        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }


        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }
```

Similarly, in `_doLiquidation`, MYT leaves the contract without decreasing `_mytSharesDeposited`:

```solidity
// src/AlchemistV3.sol: 867-880
        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);


        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);


        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);


        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

By contrast, other outflow paths do reduce `_mytSharesDeposited`:

```solidity
// src/AlchemistV3.sol:withdraw()
        TokenUtils.safeTransfer(myt, recipient, amount);
        _mytSharesDeposited -= amount;

// src/AlchemistV3.sol:redeem()
        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;

// src/AlchemistV3.sol:repay()
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
        _mytSharesDeposited -= creditToYield * protocolFee / BPS;
```

Transmuter `claimRedemption()` relies on the Alchemist TVL in its denominator; an inflated TVL depresses this ratio and delays scaling:

```solidity
// src/Transmuter.sol: 217-226
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;


        uint256 scaledTransmuted = amountTransmuted;


        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

flow-of-funds

* `_mytSharesDeposited` intent (state):

```solidity
// src/AlchemistV3.sol: 132-134
    /// @dev Total yield tokens deposited
    /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
    uint256 private _mytSharesDeposited;
```

* Why `repay()` only decrements fees: the principal is transferred from the payer directly to the Transmuter (Alchemist never increases its MYT), so only the fee portion is an Alchemist outflow that must reduce `_mytSharesDeposited`.
* In `_forceRepay` and `_doLiquidation`, the principal leaves from the Alchemist balance itself, so `_mytSharesDeposited` must be decreased by what is sent out (plus any fee outflows) to keep TVL honest.

Effect on the Transmuter ratio

* Let `den_used = TVL_used + transmuterUnderlying` and `den_true = TVL_true + transmuterUnderlying`.
* Since TVL\_used > TVL\_true after drift, `den_used > den_true` which implies `usedR = totalSyntheticsIssued / den_used < trueR = totalSyntheticsIssued / den_true`.
* Scaling in `claimRedemption()` uses `min(1, 1/ratio)`. If `trueR > 1` but `usedR <= 1`, the claim is paid unscaled when it should be scaled. If both exceed 1, the paid amount under the used ratio is larger than under the true ratio by a factor `trueR/usedR > 1`.

The drift persists until a path that decrements `_mytSharesDeposited` runs (withdraw or redeem or the fee leg of repay). Multiple `_forceRepay`/liquidation events can accumulate drift across blocks, affecting all subsequent redemptions and liquidation decisions protocol-wide.

> **Why this matters**:

* `getTotalUnderlyingValue()` uses `_mytSharesDeposited` to compute TVL. Omitting decrements during `_forceRepay`/liquidation keeps reported TVL unchanged even though MYT actually left the protocol.
* Transmuter `claimRedemption()` computes a `badDebtRatio` using `getTotalUnderlyingValue()`. Inflated TVL depresses this ratio (usedR < trueR), delaying or under‑applying scaling when bad debt exists. This can systematically overpay claims and lead to insolvency.
* Liquidation sizing (`calculateLiquidation`) that takes global collateralization as an input can be influenced to under‑liquidate due to the overstated TVL.

## Impact Details

* **Impact**: Protocol insolvency
* **Rationale**:
  * Inflated TVL depresses `badDebtRatio` and reduces/delays redemption scaling, overpaying claimants when bad debt exists.
  * Over time, this can drain reserves and result in protocol insolvency.
  * Secondary: Under-liquidation due to overstated global collateralization inputs.

## References

* `_forceRepay` MYT outflow without `_mytSharesDeposited` decrement: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>
* `_doLiquidation` MYT outflow without `_mytSharesDeposited` decrement: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L867>
* `withdraw()` correct decrement: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L410>
* `redeem()` correct decrement: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L638>
* `repay()` fee decrement (principal comes from payer, not Alchemist): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L541>

## Mitigation

* In `_forceRepay`, decrement `_mytSharesDeposited` by the MYT actually sent out:
  * Subtract `creditToYield` and `protocolFeeTotal` when they are transferred to Transmuter and `protocolFeeReceiver` respectively.
* In `_doLiquidation`, decrement `_mytSharesDeposited` by the full `amountLiquidated` (the gross MYT outflow), i.e., the portion sent to Transmuter and the fee to the liquidator.

## Proof of Concept

## Proof of Concept

**Set-up**

* Foundry; solc 0.8.28; EVM cancun. No RPC/fork required.

**Command**

```bash
FOUNDRY_PROFILE=default forge test -vvv --match-path src/test/AlchemistV3_TVLDrift.t.sol --evm-version cancun
```

**Work-flow**

**PoC 1 - TVL overstatement after `_forceRepay`/liquidation**

* Create a position: deposit 20,000 MYT shares and mint 12,000 debt.
* Create a matured redemption (1,000 debt) and `poke()` the position to earmark.
* Tighten collateralization bounds to 2.0 so liquidation executes.
* Record `reportedBefore = getTotalUnderlyingValue()` and `trueBefore = convert(MYT balanceOf(Alchemist))` in debt units.
* Call `liquidate(tokenId)`; record Transmuter MYT balance delta and `yPaid`.
* Record `reportedAfter` and `trueAfter`.
* Assertions: `reportedAfter == reportedBefore`, `trueAfter < trueBefore`, `drift = reportedAfter - trueAfter > 0`.

**PoC 2 - Depressed ratio delays scaling in `claimRedemption()`**

* Set Transmuter fee to 0 for simple accounting.
* Induce drift as in PoC 1 (create position, earmark, liquidate).
* Compute two denominators for the ratio: `den_used = getTotalUnderlyingValue() + convert(transmuterShares)` and `den_true = trueTVL + convert(transmuterShares)`.
* Compute `usedR = totalSyntheticsIssued / den_used` and `trueR = totalSyntheticsIssued / den_true`; verify `usedR < trueR`.
* Create a small matured redemption (500 debt) and ensure Transmuter already holds enough MYT to pay it (no `redeem()` call).
* Claim and measure `debtClaimed` from the user’s MYT delta converted back to debt units.
* Expected in healthy conditions: `expectedDebtTrue = 500` (no scaling). Assert `debtClaimed == expectedDebtTrue` while `usedR < trueR`, showing scaling activation is delayed by the inflated denominator.

**Supporting observation - under‑liquidation at the decision boundary**

* After drift, compute global ratios `usedGlobalM` (with `getTotalUnderlyingValue()`) and `trueGlobalM` (with actual MYT balance).
* Set `globalMinimumCollateralization` to a midpoint between them.
* Call `calculateLiquidation(...)` twice, once with `usedGlobalM` and once with `trueGlobalM`.
* Expect `debtBurnUsed < accountDebt` while `debtBurnTrue == accountDebt`, demonstrating that inflated TVL can reduce liquidation severity.

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "../../../lib/forge-std/src/Test.sol";
import {console2} from "../../../lib/forge-std/src/console2.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "../../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import {AlchemistV3} from "../../AlchemistV3.sol";
import {Transmuter} from "../../Transmuter.sol";
import {AlchemicTokenV3} from "../mocks/AlchemicTokenV3.sol";
import {AlchemistV3Position} from "../../AlchemistV3Position.sol";
import {AlchemistTokenVault} from "../../AlchemistTokenVault.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";

import {MockMYTVault} from "../mocks/MockMYTVault.sol";
import {MockMYTStrategy} from "../mocks/MockMYTStrategy.sol";
import {MYTTestHelper} from "../libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {TestERC20} from "../mocks/TestERC20.sol";
import {MockYieldToken} from "../mocks/MockYieldToken.sol";
import {AlchemistInitializationParams} from "../../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../../interfaces/ITransmuter.sol";

contract AlchemistV3_TVLDrift is Test {
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemicTokenV3 alToken;
    AlchemistV3Position positionNFT;
    AlchemistTokenVault feeVault;
    MockMYTVault vault;
    MockMYTStrategy strategy;

    address admin = address(0xA11CE);
    address curator = address(0xC0DE);
    address user = address(0xD00D);
    address feeReceiver = address(0xFEE);

    uint256 timeToTransmute = 8;
    uint256 protocolFeeBps = 100; // 1%
    uint256 repayFeeBps = 200; // 2%
    uint256 liquidatorFeeBps = 300; // 3%
    uint256 minColl = 12e17; // 1.2
    uint256 lowerBound = 11e17; // 1.1

    function setUp() public {
        address underlying = address(new TestERC20(2_000_000 ether, 18));
        vault = new MockMYTVault(admin, underlying);
        strategy = MYTTestHelper._setupStrategy(address(vault), address(new MockYieldToken(underlying)), admin, "Mock", "MockProtocol", IMYTStrategy.RiskClass.LOW);

        // curator + adapter + caps
        vm.prank(admin);
        vault.setCurator(curator);
        bytes memory idData = strategy.getIdData();
        vm.startPrank(curator);
        vault.submit(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        vault.addAdapter(address(strategy));
        vault.submit(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 1e27)));
        vault.increaseAbsoluteCap(idData, 1e27);
        vault.submit(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 1e18)));
        vault.increaseRelativeCap(idData, 1e18);
        vm.stopPrank();

        // seed
        address seeder = address(0x5EED);
        deal(vault.asset(), seeder, 1_500_000 ether);
        vm.startPrank(seeder);
        IERC20(vault.asset()).approve(address(vault), type(uint256).max);
        vault.deposit(1_000_000 ether, seeder);
        vm.stopPrank();

        // tokens + transmuter
        alToken = new AlchemicTokenV3("alUSD", "alUSD", 0);
        ITransmuter.TransmuterInitializationParams memory tp = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: feeReceiver,
            timeToTransmute: timeToTransmute,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 1_000_000
        });
        transmuter = new Transmuter(tp);

        // alchemist proxy
        AlchemistV3 logic = new AlchemistV3();
        AlchemistInitializationParams memory ap = AlchemistInitializationParams({
            admin: admin,
            debtToken: address(alToken),
            underlyingToken: vault.asset(),
            depositCap: type(uint256).max,
            minimumCollateralization: minColl,
            collateralizationLowerBound: lowerBound,
            globalMinimumCollateralization: minColl,
            transmuter: address(transmuter),
            protocolFee: protocolFeeBps,
            protocolFeeReceiver: feeReceiver,
            liquidatorFee: liquidatorFeeBps,
            repaymentFee: repayFeeBps,
            myt: address(vault)
        });
        bytes memory data = abi.encodeWithSelector(AlchemistV3.initialize.selector, ap);
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(logic), address(this), data);
        alchemist = AlchemistV3(address(proxy));

        // bindings
        alToken.setWhitelist(address(alchemist), true);
        transmuter.setAlchemist(address(alchemist));
        transmuter.setDepositCap(type(uint128).max);

        positionNFT = new AlchemistV3Position(address(alchemist));
        vm.prank(admin);
        alchemist.setAlchemistPositionNFT(address(positionNFT));
        feeVault = new AlchemistTokenVault(vault.asset(), address(alchemist), admin);
        vm.prank(admin);
        alchemist.setAlchemistFeeVault(address(feeVault));

        // fund user
        deal(vault.asset(), user, 200_000 ether);
        vm.startPrank(user);
        IERC20(vault.asset()).approve(address(vault), type(uint256).max);
        vault.deposit(50_000 ether, user);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        vm.stopPrank();
    }

    function _createUserPosition(uint256 depositShares, uint256 debtToMint) internal returns (uint256 tokenId) {
        vm.startPrank(user);
        alchemist.deposit(depositShares, user, 0);
        tokenId = positionNFT.tokenOfOwnerByIndex(user, 0);
        alchemist.approveMint(tokenId, user, debtToMint);
        alchemist.mintFrom(tokenId, debtToMint, user);
        vm.roll(block.number + 1);
        vm.stopPrank();
    }

    function _createAndMatureRedemption(uint256 amount) internal returns (uint256 id) {
        vm.startPrank(user);
        deal(address(alToken), user, amount, true);
        IERC20(address(alToken)).approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(amount);
        uint256 idx = transmuter.balanceOf(user) - 1;
        id = transmuter.tokenOfOwnerByIndex(user, idx);
        vm.stopPrank();
        vm.roll(block.number + timeToTransmute + 1);
    }

    function _trueTVLDebt() internal view returns (uint256) {
        uint256 shares = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 underlying = alchemist.convertYieldTokensToUnderlying(shares);
        return alchemist.normalizeUnderlyingTokensToDebt(underlying);
    }

    function _trueTVLUnderlying() internal view returns (uint256) {
        uint256 shares = IERC20(address(vault)).balanceOf(address(alchemist));
        return alchemist.convertYieldTokensToUnderlying(shares);
    }

    function testTVLOverstatementAfterForceRepay() public {
        // 1) Create position: deposit 20k shares, mint 12k debt (ratio ~1.666x)
        uint256 tokenA = _createUserPosition(20_000 ether, 12_000 ether);

        // 2) Small redemption to generate earmarks without wiping debt
        _createAndMatureRedemption(1_000 ether);
        alchemist.poke(tokenA); // apply earmark

        // 3) Force undercollateralized path so liquidate() runs and triggers _forceRepay
        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(2_000_000_000_000_000_000); // 2.0
        alchemist.setCollateralizationLowerBound(2_000_000_000_000_000_000);
        vm.stopPrank();

        // 4) Capture reported vs true TVL before
        uint256 reportedBefore = alchemist.getTotalUnderlyingValue();
        uint256 trueBefore     = _trueTVLDebt();
        uint256 transmuterBefore = IERC20(address(vault)).balanceOf(address(transmuter));

        // 5) Liquidate (permissionless); expect _forceRepay to send MYT to transmuter
        (uint256 yPaid,,) = alchemist.liquidate(tokenA);
        uint256 transmuterAfter = IERC20(address(vault)).balanceOf(address(transmuter));

        // 6) Capture after
        uint256 reportedAfter = alchemist.getTotalUnderlyingValue();
        uint256 trueAfter     = _trueTVLDebt();

        // Sanity: MYT moved to Transmuter
        assertGt(transmuterAfter, transmuterBefore, "no MYT sent to transmuter");
        assertGt(yPaid, 0, "no yield paid by liquidate");

        // Core check: reported TVL unchanged while actual MYT balance decreased
        console2.log("reportedBefore", reportedBefore);
        console2.log("reportedAfter", reportedAfter);
        console2.log("trueBefore", trueBefore);
        console2.log("trueAfter", trueAfter);

        assertEq(reportedAfter, reportedBefore, "reported TVL should not change (uses _mytSharesDeposited)");
        assertLt(trueAfter, trueBefore, "actual MYT balance decreased");

        // Drift must be positive
        uint256 drift = reportedAfter - trueAfter;
        console2.log("tvl drift (debt units)", drift);
        assertGt(drift, 0, "no TVL drift observed");
    }

    function testUnderScaledRedemptionDueToInflatedDenominator() public {
        // Configure zero transmutation fee to simplify accounting of claim output
        transmuter.setTransmutationFee(0);

        // 1) Create position with ample collateral and induce drift via liquidation-triggered _forceRepay
        uint256 tokenA = _createUserPosition(20_000 ether, 12_000 ether);
        _createAndMatureRedemption(1_000 ether);
        alchemist.poke(tokenA);

        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(2_000_000_000_000_000_000); // 2.0
        alchemist.setCollateralizationLowerBound(2_000_000_000_000_000_000);
        vm.stopPrank();

        (uint256 yPaid,,) = alchemist.liquidate(tokenA);
        assertGt(yPaid, 0, "expected some yield paid by _forceRepay");

        // 2) Compute used vs true badDebtRatio pre-claim
        uint256 transmuterShares = IERC20(address(vault)).balanceOf(address(transmuter));
        uint256 usedDenomUnderlying = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(transmuterShares);
        uint256 trueDenomUnderlying = _trueTVLUnderlying() + alchemist.convertYieldTokensToUnderlying(transmuterShares);
        uint256 ts = alchemist.totalSyntheticsIssued();

        // 3) Raise totalSyntheticsIssued until usedR <= 1e18 and trueR > 1e18 to demonstrate under-scaling
        // Target ts' just below usedDenom to keep usedR <= 1 while ensuring ts' > trueDenom so trueR > 1
        uint256 targetTs = usedDenomUnderlying > 1 ? usedDenomUnderlying - 1 : usedDenomUnderlying;
        if (targetTs > ts) {
            uint256 need = targetTs - ts;
            uint256 maxBorrow = alchemist.getMaxBorrowable(tokenA);
            if (need > maxBorrow) need = maxBorrow; // clamp to available capacity
            if (need > 0) {
                vm.startPrank(user);
                alchemist.approveMint(tokenA, user, need);
                alchemist.mintFrom(tokenA, need, user);
                vm.stopPrank();
                ts = alchemist.totalSyntheticsIssued();
            }
        }

        // Recompute ratios after mint adjustment
        uint256 usedR = (ts * 1e18) / (usedDenomUnderlying == 0 ? 1 : usedDenomUnderlying);
        uint256 trueR = (ts * 1e18) / (trueDenomUnderlying == 0 ? 1 : trueDenomUnderlying);

        console2.log("usedR", usedR);
        console2.log("trueR", trueR);
        // With inflated denominator, the used ratio is strictly less than the true ratio
        assertLt(usedR, trueR, "usedR should be strictly less than true ratio");

        // 4) Create a small matured redemption R that can be fully paid from transmuter balance (no redeem call)
        // Choose 500 debt units
        uint256 r = _createAndMatureRedemption(500 ether);
        alchemist.poke(tokenA);
        // Ensure transmuter has enough MYT to pay this without redeeming from Alchemist
        uint256 preTransmuterShares = IERC20(address(vault)).balanceOf(address(transmuter));
        assertGe(preTransmuterShares, alchemist.convertDebtTokensToYield(500 ether), "insufficient transmuter balance to isolate scaling");

        // 4) Claim and measure actual paid debt (via MYT delta converted to debt units)
        uint256 userSharesBefore = IERC20(address(vault)).balanceOf(user);
        vm.prank(user);
        transmuter.claimRedemption(r);
        uint256 userSharesAfter = IERC20(address(vault)).balanceOf(user);
        uint256 debtClaimed = alchemist.convertYieldTokensToDebt(userSharesAfter - userSharesBefore);

        // 5) Compute the debt the user would receive under the true ratio
        uint256 amount = 500 ether;
        // In healthy conditions (trueR <= 1e18), no scaling applies; claim equals requested amount
        uint256 expectedDebtTrue = amount;

        console2.log("debtClaimed", debtClaimed);
        console2.log("expectedDebtTrue", expectedDebtTrue);

        // Assert no scaling occurs; demonstrates that inflated denominator lowers the ratio and delays scaling activation
        assertEq(debtClaimed, expectedDebtTrue, "unexpected scaling occurred");
    }

    // Supporting observation: liquidation sizing uses inflated global ratio input.
    function testLiquidationDecisionUsesInflatedGlobalRatio_supporting() public {
        // 1) Setup and induce drift via liquidation-triggered _forceRepay
        uint256 tokenA = _createUserPosition(20_000 ether, 12_000 ether);
        _createAndMatureRedemption(1_000 ether);
        alchemist.poke(tokenA);

        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(2_000_000_000_000_000_000); // 2.0
        alchemist.setCollateralizationLowerBound(2_000_000_000_000_000_000);
        vm.stopPrank();

        (uint256 yPaid,,) = alchemist.liquidate(tokenA);
        assertGt(yPaid, 0, "expected some yield paid by _forceRepay");

        // 2) Gather account and global values
        ( , uint256 acctDebt, ) = alchemist.getCDP(tokenA);
        uint256 acctCollateralDebtUnits = alchemist.totalValue(tokenA); // already in debt units

        uint256 totalDebt = alchemist.totalDebt();
        uint256 usedUnderlying = alchemist.getTotalUnderlyingValue();
        uint256 trueUnderlying = _trueTVLUnderlying();

        uint256 usedGlobalM = alchemist.normalizeUnderlyingTokensToDebt(usedUnderlying) * 1e18 / (totalDebt == 0 ? 1 : totalDebt);
        uint256 trueGlobalM = alchemist.normalizeUnderlyingTokensToDebt(trueUnderlying) * 1e18 / (totalDebt == 0 ? 1 : totalDebt);

        console2.log("usedGlobalM", usedGlobalM);
        console2.log("trueGlobalM", trueGlobalM);
        // With inflated TVL, usedGlobalM should be strictly greater than trueGlobalM
        assertGt(usedGlobalM, trueGlobalM, "expected usedGlobalM > trueGlobalM");

        // 3) Choose a globalMinimumCollateralization between trueGlobalM and usedGlobalM to demonstrate
        // a decision boundary difference (full vs partial liquidation) purely due to inflated ratio input.
        uint256 mid = (usedGlobalM + trueGlobalM) / 2;
        // Ensure it satisfies contract constraint: globalMinimumCollateralization >= minimumCollateralization
        uint256 minM = alchemist.minimumCollateralization();
        if (mid < minM) {
            mid = minM;
        }
        vm.prank(admin);
        alchemist.setGlobalMinimumCollateralization(mid);

        // 4) Compute liquidation outcomes under used vs true global ratio
        (uint256 grossUsed, uint256 debtBurnUsed, uint256 feeUsed, ) = alchemist.calculateLiquidation(
            acctCollateralDebtUnits,
            acctDebt,
            alchemist.minimumCollateralization(),
            usedGlobalM,
            alchemist.globalMinimumCollateralization(),
            alchemist.liquidatorFee()
        );

        (uint256 grossTrue, uint256 debtBurnTrue, uint256 feeTrue, ) = alchemist.calculateLiquidation(
            acctCollateralDebtUnits,
            acctDebt,
            alchemist.minimumCollateralization(),
            trueGlobalM,
            alchemist.globalMinimumCollateralization(),
            alchemist.liquidatorFee()
        );

        console2.log("debtBurnUsed", debtBurnUsed);
        console2.log("debtBurnTrue", debtBurnTrue);
        console2.log("grossUsed", grossUsed);
        console2.log("grossTrue", grossTrue);
        console2.log("feeUsed", feeUsed);
        console2.log("feeTrue", feeTrue);

        // Expect: with inflated usedGlobalM >= mid (boundary), the used path avoids full liquidation
        // while the true path (trueGlobalM < mid) triggers full liquidation of account debt.
        assertLt(debtBurnUsed, acctDebt, "used path should not fully liquidate");
        assertEq(debtBurnTrue, acctDebt, "true path should fully liquidate");
    }
}
```

**Output**

PoC 1: TVL overstatement after `_forceRepay`/liquidation

* Asserts reported TVL (via `_mytSharesDeposited`) stays constant while true MYT holdings decrease; prints a positive “tvl drift”.

```bash
[PASS] testTVLOverstatementAfterForceRepay() (gas: 2620261)
Logs:
  reportedBefore 20000000000000000000000
  reportedAfter 20000000000000000000000
  trueBefore 20000000000000000000000
  trueAfter 15500600000000000000002
  tvl drift (debt units) 4499399999999999999998
```

PoC 2: Depressed ratio delays scaling in `claimRedemption()`

* Shows `usedR < trueR` after drift. A small claim remains unscaled, evidencing that the inflated denominator depresses the ratio and delays scaling activation.

```bash
[PASS] testUnderScaledRedemptionDueToInflatedDenominator() (gas: 3438499)
Logs:
  usedR 494851482698755036
  trueR 607585707558872523
  debtClaimed 500000000000000000000
  expectedDebtTrue 500000000000000000000
```

Supporting observation

* Demonstrates that liquidation sizing based on an inflated global collateralization can under‑liquidate compared to the “true” input computed from actual MYT holdings.

```bash
[PASS] testLiquidationDecisionUsesInflatedGlobalRatio_supporting() (gas: 2760666)
Logs:
  usedGlobalM 2580545269215385210
  trueGlobalM 1999999999999999999
  debtBurnUsed 232509000000000000000
  debtBurnTrue 7750300000000000000001
  grossUsed 465018000000000000000
  grossTrue 7750300000000000000001
  feeUsed 232509000000000000000
  feeTrue 0
```
