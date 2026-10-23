# 56552 sc high liquidation fee misrouting in alchemistv3 doliquidation leads to theft of unclaimed yield liquidator fee stranded&#x20;

**Submitted on Oct 17th 2025 at 15:06:24 UTC by @dizaye for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56552
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

AlchemistV3’s liquidation path withholds the base liquidation fee from the amount sent to the Transmuter but then conditionally fails to pay that fee to the liquidator. When the condition fails, the fee remains stranded in the Alchemist contract instead of being paid out. This causes direct loss of liquidator compensation (unclaimed yield) and degrades liquidation incentives. The issue is reproducible on a mainnet fork with the live MYT vault.

## Vulnerability Details

* In [AlchemistV3.\_doLiquidation(uint256,uint256,uint256)](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852), liquidation computes four values via calculateLiquidation:
  * grossCollateralToSeize (debt units, converted to MYT shares)
  * debtToBurn
  * baseFee (debt units, converted to MYT shares)
  * outsourcedFee (underlying units)
* The function then:
  1. Debits the account by the gross liquidation amount:
     * [AlchemistV3.\_doLiquidation() — account.collateralBalance -= amountLiquidated](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L871)
  2. Sends the net amount (gross minus base fee) to the Transmuter:
     * [AlchemistV3.\_doLiquidation() — TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield)](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L875)
  3. Attempts to pay the base fee to the liquidator, but only if the account still has at least feeInYield remaining collateral:
     * [AlchemistV3.\_doLiquidation() — if (feeInYield > 0 && account.collateralBalance >= feeInYield) { TokenUtils.safeTransfer(myt, msg.sender, feeInYield); }](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L878)
* Logical mismatch:
  * The account’s collateral is reduced by the gross seized amount (which includes the base fee) before paying the fee.
  * Since net was already sent to the Transmuter, the contract holds the fee’s MYT amount and should pay it unconditionally to the liquidator.
  * The “>= feeInYield” gate against the account’s post-seizure collateral incorrectly prevents paying the fee and strands the fee inside AlchemistV3 when the condition fails.
* Consequence:
  * Liquidator fee (unclaimed yield) is withheld from the Transmuter but not paid to the liquidator, remaining in AlchemistV3. This violates the implied invariant that gross seized = net paid to Transmuter + fee paid to liquidator.

## Impact Details

* Direct loss for liquidators:
  * The fee that should be paid to the liquidator is withheld. Over time or in adverse market conditions, this can represent meaningful losses, undermining the liquidation incentive mechanism.
* Systemic effect:
  * Underpayment of liquidators discourages participation, which can degrade system resilience and timely liquidations.
* Why “Theft of unclaimed yield”:
  * The fee represents a claim on seized yield directed to the liquidator; failing to pay it is a direct misrouting of value, consistent with the “Theft of unclaimed yield” impact category.

## References

* Fee computation and transfers:
  * [AlchemistV3.\_doLiquidation()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852)
  * [Debit account by gross: account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L871)
  * [Net to Transmuter: TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield)](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L875)
  * [Conditional fee transfer to liquidator (bug): if (feeInYield > 0 && account.collateralBalance >= feeInYield) { … }](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L878)

## Proof of Concept

## Proof of Concept

This PoC is runnable on a mainnet fork using the live MYT vault (Morpho Yearn OG WETH: 0xE89371eAaAC6D46d4C3ED23453241987916224FC). It demonstrates that during liquidation feeInYield is withheld from the Transmuter but not paid to the liquidator and remains in AlchemistV3.

File: PoC\_AlchemistV3\_LiquidationFeeStuck.t.sol

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import {AlchemistV3} from "src/AlchemistV3.sol";
import {AlchemistInitializationParams} from "src/interfaces/IAlchemistV3.sol";
import {AlchemistV3Position} from "src/AlchemistV3Position.sol";
import {ITransmuter} from "src/interfaces/ITransmuter.sol";
import {Transmuter} from "src/Transmuter.sol";
import {AlchemicTokenV3} from "src/test/mocks/AlchemicTokenV3.sol";

contract PoC_AlchemistV3_LiquidationFeeStuck is Test {
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant MYT  = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;

    uint256 constant BPS = 10_000;
    uint256 constant ONE = 1e18;

    AlchemistV3 alchemist;
    AlchemistV3Position positionNFT;
    Transmuter transmuter;
    AlchemicTokenV3 debt;

    address self;
    address protocolFeeReceiver = address(0xBEEF);
    uint256 tokenId;

    function setUp() public {
        self = address(this);

        debt = new AlchemicTokenV3("alETH", "alETH", 18);

        ITransmuter.TransmuterInitializationParams memory tparams;
        tparams.syntheticToken = address(debt);
        tparams.timeToTransmute = 7200;
        tparams.transmutationFee = 0;
        tparams.exitFee = 0;
        tparams.feeReceiver = self;
        transmuter = new Transmuter(tparams);

        AlchemistV3 logic = new AlchemistV3();
        AlchemistInitializationParams memory p;
        p.debtToken = address(debt);
        p.underlyingToken = WETH;
        p.depositCap = type(uint256).max;
        p.minimumCollateralization = 1_500_000_000_000_000_000;
        p.globalMinimumCollateralization = 1_500_000_000_000_000_000;
        p.collateralizationLowerBound = 1_500_000_000_000_000_000;
        p.admin = self;
        p.transmuter = address(transmuter);
        p.protocolFee = 0;
        p.protocolFeeReceiver = protocolFeeReceiver;
        p.liquidatorFee = BPS;
        p.repaymentFee = 0;
        p.myt = MYT;

        bytes memory initData = abi.encodeWithSelector(AlchemistV3.initialize.selector, p);
        address proxyAdmin = address(0xBEEFDEAD);
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(logic), proxyAdmin, initData);
        alchemist = AlchemistV3(address(proxy));

        // whitelist AlchemistV3 proxy to be able to mint the debt token during mint()
        debt.setWhitelist(address(alchemist), true);

        positionNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(positionNFT));

        transmuter.setAlchemist(address(alchemist));

        uint256 shares = 100 ether;
        deal(MYT, self, shares);
        IERC20(MYT).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(shares, self, 0);

        tokenId = positionNFT.totalSupply();
        assertEq(tokenId, 1, "first tokenId should be 1");

        uint256 tv = alchemist.totalValue(tokenId);
        uint256 minCollat = alchemist.minimumCollateralization();
        uint256 mintAmount = (tv * ONE) / minCollat;
        alchemist.mint(tokenId, mintAmount, self);
    }

    function test_Liquidation_FeeGetsStranded() public {
        uint256 preAlcMyt = IERC20(MYT).balanceOf(address(alchemist));
        uint256 preTrnMyt = IERC20(MYT).balanceOf(address(transmuter));
        uint256 preLiqMyt = IERC20(MYT).balanceOf(self);

        (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        console2.log("yieldAmount (gross seized, MYT):", yieldAmount);
        console2.log("feeInYield:", feeInYield);
        console2.log("feeInUnderlying:", feeInUnderlying);

        uint256 postAlcMyt = IERC20(MYT).balanceOf(address(alchemist));
        uint256 postTrnMyt = IERC20(MYT).balanceOf(address(transmuter));
        uint256 postLiqMyt = IERC20(MYT).balanceOf(self);

        uint256 transmuterDelta = postTrnMyt - preTrnMyt;
        uint256 liquidatorDelta = postLiqMyt - preLiqMyt;

        assertEq(transmuterDelta, yieldAmount - feeInYield, "transmuter must receive net (gross - fee)");
        assertEq(liquidatorDelta, 0, "liquidator unexpectedly received fee");

        uint256 expectedPostAlc = preAlcMyt - (yieldAmount - feeInYield);
        assertEq(postAlcMyt, expectedPostAlc, "alchemist should still hold the fee amount");

        console2.log("OK: Stranded fee confirmed; fee not paid to liquidator, remains in AlchemistV3");
    }
}
```

Step-by-step explanation

1. Deploy AlchemistV3 behind a TransparentUpgradeableProxy and initialize it with:
   * debtToken: locally deployed AlchemicTokenV3
   * underlying: WETH
   * transmuter: locally deployed Transmuter (setTransmutationFee=0 for clarity)
   * liquidatorFee: 10000 bps (100%) to maximize base fee visibility
   * myt: live MYT vault (0xE893…24FC) Shown in PoC setUp()
2. Seed MYT shares to the test address and deposit them into AlchemistV3; then mint debt to reach minimum collateralization:
   * deal(MYT, self, shares); alchemist.deposit(shares, self, 0)
   * alchemist.mint(tokenId, mintAmount, self)
3. Snapshot balances, liquidate, and compute deltas:
   * Pre balances: AlchemistV3 MYT, Transmuter MYT, Liquidator MYT
   * Call alchemist.liquidate(tokenId)
   * Post balances and deltas:
     * transmuterDelta == yieldAmount - feeInYield (net only)
     * liquidatorDelta == 0 (no fee paid)
     * alchemistPostMYT == alchemistPreMYT - (yieldAmount - feeInYield) ⇒ fee stranded in AlchemistV3
4. The PoC asserts the misrouting:
   * assertEq(transmuterDelta, yieldAmount - feeInYield)
   * assertEq(liquidatorDelta, 0)
   * assertEq(postAlcMyt, expectedPostAlc)

Run:

```
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/sg_AONSfMDHjsVSrDUqZA --match-path src/test/PoC_AlchemistV3_LiquidationFeeStuck.t.sol -vvvv --evm-version cancun
```

Output:

```
- yieldAmount (gross seized, MYT): 99999999999999999999
- feeInYield: 33333333333333333333
- feeInUnderlying: 0
- “OK: Stranded fee confirmed; fee not paid to liquidator, remains in AlchemistV3”
```

Recommended mitigation

* In [AlchemistV3.\_doLiquidation()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852):
  * Replace the conditional fee payment gate with an unconditional transfer whenever feeInYield > 0:
    * From: if (feeInYield > 0 && account.collateralBalance >= feeInYield) { TokenUtils.safeTransfer(myt, msg.sender, feeInYield); }
    * To: if (feeInYield > 0) { TokenUtils.safeTransfer(myt, msg.sender, feeInYield); }
  * Gross seizure already debits the account by “net + fee”; the contract holds the fee’s MYT and must pay the liquidator regardless of the post-seizure residual account.collateralBalance.
