# 57633 sc high block gated earmark call in redeem nullifies prefunded transmuter cover on the first redemption of each block leading to collateral overpayment and potential protocol insolvency

**Submitted on Oct 27th 2025 at 18:56:10 UTC by @al0x23 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57633
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Because redeem() always calls \_earmark() first, and \_earmark() snapshots the transmuter’s balance into lastTransmuterTokenBalance, the first redemption in each block effectively erases any prefunded MYT cover accumulated before that block. When redeem() then computes the delta (transmuterBal - lastTransmuterTokenBalance), it sees zero and ignores the available cover. As a result, the protocol burns earmarked debt and pulls new collateral even though matching MYT already exists in the Transmuter, gradually draining collateral and unclaimed yield, risking long-term protocol insolvency.

## Vulnerability Details

Root cause

The Alchemist determines how much prefunded MYT (“cover”) exists on the Transmuter by comparing the current vault balance to the previously stored lastTransmuterTokenBalance. The \_earmark() function both computes and updates this reference value.

When redeem() calls \_earmark() first (and this is the first \_earmark() of the block), the reference point is immediately reset before redeem() measures the delta. This causes deltaYield to evaluate to zero, even if prefunded cover was present.

```solidity
function redeem(uint256 amount) external onlyTransmuter {
    _earmark(); // snapshots current Transmuter balance
    uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 deltaYield = transmuterBal > lastTransmuterTokenBalance
        ? transmuterBal - lastTransmuterTokenBalance
        : 0; // always 0 right after _earmark()
}
```

When it triggers

On the first call to \_earmark() within a block, typically the first redeem() that block.

Any prior prefunded MYT (yield or earmarked debt repayments) will be “seen” by \_earmark(), but then immediately overwritten in lastTransmuterTokenBalance. Subsequent redemptions in the same block skip \_earmark() due to the block guard, so they may work correctly — but by then, the prefunded cover has already been zeroed out.

Timeline illustration Block N — before redemption Transmuter MYT balance: 1,000 Alchemist.lastTransmuterTokenBalance = 0 Prefunded cover available: 1,000 MYT Expected: next redeem uses this 1,000 as cover

Block N+1 — redeem() starts

redeem() calls \_earmark() • Reads balance = 1,000 • Computes coverInDebt = convert(1,000) • Updates lastTransmuterTokenBalance = 1,000 → Cover is acknowledged but not spent.

redeem() resumes • Reads transmuterBal = 1,000 • Computes deltaYield = 0 • Believes no cover is available → overdraws collateral.

Result • Prefunded MYT remains unused. • Earmarked debt reduced as if no cover existed. • Collateral and accounting drift apart.

Why the bug persists and matters

Accounting invariant break: the relationship between totalDebt, cumulativeEarmarked, and transmuter-held MYT is violated as soon as the first redemption runs.

No self-correction: subsequent \_earmark() calls only recognize new deltas; the lost prefunded cover never re-enters accounting.

Masking by ongoing yield: future yield can hide the mismatch temporarily, but once yield slows or stops, redemptions start pulling principal.

System-wide desync: LTV, solvency, and earmarking calculations now operate from an incorrect baseline.

Irreversible: no user-accessible function re-syncs lastTransmuterTokenBalance, so the inconsistency persists until patched.

Even if the MYT tokens remain physically in the Transmuter, the logic that tracks and applies them to reduce debt has disconnected — meaning the protocol appears solvent while silently leaking collateral.

## Impact Details

Impact Details

Each affected redemption ignores existing prefunded cover, over-repaying debt with collateral. The Transmuter accumulates idle MYT that the Alchemist no longer accounts for. Over time, totalDebt and actual assets diverge; the protocol becomes over-leveraged. When new yield halts, redemptions begin consuming principal, depleting reserves.

When losses materialize:

1. During periods of low yield or paused strategies, unclaimed cover is permanently lost.
2. After many redemptions, cumulative collateral depletion can render the protocol insolvent.
3. During audits or migrations, apparent on-chain solvency may conceal missing collateral equivalent to the unrecognized prefunded cover.

Severity classification:

This issue causes prefunded MYT (unclaimed yield) held by the Transmuter to become permanently unrecognized by the Alchemist’s accounting logic. Although the tokens remain on-chain, they are frozen from the protocol’s perspective — they will never again be used to reduce debt or satisfy redemptions.

Because the loss is deterministic, accumulative, and triggered by normal user operations (redeem()), this aligns precisely with the High severity “Permanent freezing of unclaimed yield”.

The bug does not immediately cause theft or insolvency (so it is not Critical), but it leads to a permanent loss of claimable yield and a progressive erosion of solvency over time, which is economically equivalent to a high-severity vulnerability.

## Proof of Concept

## Proof of Concept

This PoC reuses the same contracts and libraries from the project’s integration tests but simplifies the environment to reproduce the bug deterministically. Unlike the full integration suite—which deploys through proxies, allocators, strategies, and multi-user scenarios over millions of blocks—this harness directly instantiates a fresh Transmuter and AlchemistV3Harness (a minimally modified version of AlchemistV3 with manual initialization). This isolates the relevant state transition (\_earmark() → redeem()) without unrelated side effects or governance layers. All other dependencies, interfaces, and math are identical to production code; only the initialization path is shortened to make the test deterministic and focused.

Copy this file in the test folder src/test/, same place as IntegrationTest.t.sol

and run $forge test --mt testAudit\_PrefundedCoverFrozen\_FirstRedeemInBlock\_PoC --rpc-url $MAINNET\_RPC\_URL -vvvv

```solidity

// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "../libraries/SafeCast.sol";
import "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {EulerUSDCAdapter} from "../adapters/EulerUSDCAdapter.sol";
import {Transmuter} from "../Transmuter.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {IAlchemicToken} from "../interfaces/IAlchemicToken.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {AlchemistETHVault} from "../AlchemistETHVault.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {MockAlchemistAllocator} from "./mocks/MockAlchemistAllocator.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

import "../../lib/forge-std/src/StdStorage.sol";

// Tests for integration with Euler V2 Earn Vault
contract IntegrationTest is Test {
    using stdStorage for StdStorage;
    StdStorage private _stdstore;
    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;
    address weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;
    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e18;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e18;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;
    // Fee receiver
    address receiver = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    address alUSD = 0xBC6DA0FE9aD5f3b0d58160288917AA56653660E9;

    address USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    address EULER_USDC = 0x797DD80692c3b2dAdabCe8e30C07fDE5307D48a9;

    // MYT variables
    VaultV2 vault;
    MockAlchemistAllocator allocator;
    MockMYTStrategy mytStrategy;
    address public operator = address(20); // default operator
    address public admin = address(21); // DAO OSX
    address public curator = address(22);
    address public mockVaultCollateral;
    address public mockStrategyYieldToken;
    uint256 public defaultStrategyAbsoluteCap = 2_000_000_000e18;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%

    event TestIntegrationLog(string message, uint256 value);

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        setUpMYT(6); // 6 decimals for USDC underlying token
        addDepositsToMYT();

        vm.startPrank(caller);

        /*         deal(EULER_USDC, address(0xbeef), 100_000e18);
        deal(EULER_USDC, address(0xdad), 100_000e18); */
        deal(alUSD, address(0xdad), 100_000e18);
        deal(alUSD, address(0xdead), 100_000e18);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: alUSD,
            feeReceiver: receiver,
            timeToTransmute: 5_256_000,
            transmutationFee: 100,
            exitFee: 200,
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
            debtToken: alUSD,
            underlyingToken: USDC,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            transmuter: address(transmuterLogic),
            protocolFee: 100,
            protocolFeeReceiver: receiver,
            liquidatorFee: 300, // in bps? 3%
            repaymentFee: 100,
            myt: address(vault)
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        transmuterLogic.setDepositCap(uint256(type(int256).max));

        transmuterLogic.setAlchemist(address(alchemist));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        vm.stopPrank();

        vm.startPrank(0x8392F6669292fA56123F71949B52d883aE57e225);
        IAlchemicToken(alUSD).setWhitelist(address(alchemist), true);
        IAlchemicToken(alUSD).setCeiling(address(alchemist), type(uint256).max);
        vm.stopPrank();
    }

    function setUpMYT(uint256 alchemistUnderlyingTokenDecimals) public {
        vm.startPrank(admin);
        uint256 TOKEN_AMOUNT = 1_000_000; // Base token amount
        uint256 initialSupply = TOKEN_AMOUNT * 10 ** alchemistUnderlyingTokenDecimals;
        mockVaultCollateral = USDC;
        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(
            address(vault),
            mockStrategyYieldToken,
            admin,
            "MockToken",
            "MockTokenProtocol",
            IMYTStrategy.RiskClass.LOW
        );
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

    function addDepositsToMYT() public {
        uint256 shares = _magicDepositToVault(address(vault), address(0xbeef), 1_000_000e6);
        emit TestIntegrationLog("0xbeef shares", shares);
        shares = _magicDepositToVault(address(vault), address(0xdad), 1_000_000e6);
        emit TestIntegrationLog("0xdad shares", shares);

        // then allocate to the strategy
        vm.startPrank(address(admin));
        allocator.allocate(address(mytStrategy), vault.convertToAssets(vault.totalSupply()));
        vm.stopPrank();
    }

    function _magicDepositToVault(address vault, address depositor, uint256 amount) internal returns (uint256) {
        deal(USDC, address(depositor), amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(USDC, vault, amount);
        uint256 shares = IVaultV2(vault).deposit(amount, depositor);
        vm.stopPrank();
        return shares;
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        vault.submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));
    }

    function testAudit_PrefundedCoverFrozen_FirstRedeemInBlock_PoC() external {
        //
        // 1. Deploy a fresh Transmuter we fully control.
        //
        ITransmuter.TransmuterInitializationParams memory freshParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: alUSD,
            feeReceiver: receiver,
            timeToTransmute: 5_256_000,
            transmutationFee: 100,
            exitFee: 200,
            graphSize: 52_560_000
        });

        Transmuter localTransmuter = new Transmuter(freshParams);

        //
        // 2. Deploy fresh AlchemistV3Harness and initialize it against localTransmuter.
        //    NOTE: harnessInit() in AlchemistV3Harness MUST set:
        //      underlyingConversionFactor = 10 ** 12;
        //
        AlchemistV3Harness harness = new AlchemistV3Harness();
        harness.harnessInit(
            address(this), // _admin
            alUSD, // _debtToken
            USDC, // _underlyingToken
            address(localTransmuter), // _transmuter
            receiver, // _protocolFeeReceiver
            address(vault), // _myt (vault shares token)
            minimumCollateralization, // _minimumCollateralization
            minimumCollateralization, // _globalMinimumCollateralization
            1_052_631_578_950_000_000, // _collateralizationLowerBound (~1.05e18)
            100, // _protocolFee (bps)
            300, // _liquidatorFee (bps)
            100, // _repaymentFee (bps)
            type(uint256).max // _depositCap
        );

        //
        // 3. Hook transmuter -> harness so it can call onlyTransmuter funcs (redeem()).
        //
        localTransmuter.setAlchemist(address(harness));

        //
        // 4. Prefund the Transmuter with MYT "cover" BEFORE calling redeem().
        //    We simulate accumulated yield / prefunded cover sitting in Transmuter.
        //    `deal()` writes directly into the vault's balanceOf(transmuter) slot.
        //
        uint256 COVER_AMOUNT = 1_000e18; // 1000 MYT shares
        deal(address(vault), address(localTransmuter), COVER_AMOUNT);

        //
        // Preconditions we want:
        //  - harness.lastTransmuterTokenBalance == 0
        //    (so harness *thinks* the transmuter had nothing so far)
        //  - BUT the transmuter actually holds COVER_AMOUNT MYT already
        //
        uint256 preLast = harness.exposed_lastTransmuterTokenBalance();
        assertEq(preLast, 0, "pre: lastTransmuterTokenBalance should start at 0");

        uint256 balTransmuterNow = IERC20(address(vault)).balanceOf(address(localTransmuter));
        assertEq(balTransmuterNow, COVER_AMOUNT, "pre: transmuter must already hold prefunded cover");

        //
        // 5. Force protocol state so that _earmark() will actually execute.
        //
        // _earmark() bails out unless:
        //   totalDebt > 0
        //   block.number > lastEarmarkBlock
        //
        // We’ll locate those storage slots with StdStorage-style logic
        // and patch them via vm.store.
        //
        uint256 totalDebtSlotIndex = uint256(_stdstore.target(address(harness)).sig(harness.totalDebt.selector).find());

        uint256 lastEarmarkBlockSlotIndex = uint256(
            _stdstore.target(address(harness)).sig(harness.lastEarmarkBlock.selector).find()
        );

        // Pretend borrowers owe something so totalDebt > 0.
        vm.store(
            address(harness),
            bytes32(totalDebtSlotIndex),
            bytes32(uint256(1e18)) // 1e18 debt
        );

        // Force lastEarmarkBlock = block.number - 1
        // so that _earmark() thinks "a new block has passed" and will run.
        vm.store(address(harness), bytes32(lastEarmarkBlockSlotIndex), bytes32(uint256(block.number - 1)));

        //
        // 6. Act as the transmuter and call the redeem-like harness helper.
        //
        // exposed_redeem_like_real(amount) simulates what redeem(amount) does:
        //   - snapshot (transmuter.balance, lastTransmuterTokenBalance) BEFORE
        //   - call _earmark()       [this is what redeem() does first]
        //       * _earmark() will:
        //           - read that prefunded COVER_AMOUNT
        //           - internally claim it against earmarks
        //           - update lastTransmuterTokenBalance = COVER_AMOUNT
        //         BUT it does NOT actually consume that cover in a user-facing redemption yet
        //   - then compute the redeem() cover math:
        //         deltaYield = transmuterBalAfter - lastTransmuterTokenBalanceAfter
        //         coverDebt  = convertYieldTokensToDebt(deltaYield)
        //
        // The bug: because lastTransmuterTokenBalance was just bumped up to COVER_AMOUNT,
        // deltaYield becomes 0. That freezes the prefunded cover for this redemption.
        //
        vm.startPrank(address(localTransmuter));
        (
            uint256 deltaYieldAfter,
            uint256 coverDebtAfter,
            uint256 snapTransmuterBalBefore,
            uint256 snapLastBalBefore,
            uint256 snapLastBalAfter
        ) = harness.exposed_redeem_like_real(500e18);
        vm.stopPrank();

        //
        // 7. Assertions proving the freeze-of-cover behavior.
        //

        // (A) Transmuter really was pre-funded before redeem().
        assertEq(
            snapTransmuterBalBefore,
            COVER_AMOUNT,
            "precondition: transmuter held prefunded cover before redeem()"
        );

        // (B) Alchemist thought lastTransmuterTokenBalance was 0 before _earmark().
        assertEq(snapLastBalBefore, 0, "precondition: lastTransmuterTokenBalance was 0 before _earmark()");

        // (C) _earmark() ran first and 'captured' that prefunded cover internally,
        //     then advanced lastTransmuterTokenBalance := COVER_AMOUNT.
        //     This mirrors production redeem(): redeem() always calls _earmark() first.
        assertEq(
            snapLastBalAfter,
            COVER_AMOUNT,
            "_earmark() advanced lastTransmuterTokenBalance to match transmuter balance"
        );

        // (D) Now redeem() measures fresh cover as:
        //         deltaYieldAfter = transmuterBalAfter - lastTransmuterTokenBalanceAfter
        //     But because _earmark() already synced lastTransmuterTokenBalance up to COVER_AMOUNT
        //     in THIS SAME redeem flow, deltaYieldAfter is 0 for this first redemption.
        //
        //     => The prefunded cover that existed going into this block is now invisible
        //        to the very redemption that should have consumed it.
        assertEq(
            deltaYieldAfter,
            0,
            "BUG: first redeem() in a block sees zero deltaYield because _earmark() consumed/synced it first"
        );

        // (E) With deltaYieldAfter = 0, coverDebtAfter = 0 too.
        //     So the redemption will NOT apply that prefunded cover to cancel earmarked debt.
        //     Instead, it will burn earmarked debt + move collateral, as if no cover existed.
        //
        //     This is exactly the permanent freezing of unclaimed yield described in the report:
        //     the cover was real, but it was made unusable for this redemption.
        assertEq(
            coverDebtAfter,
            0,
            "BUG: coverDebtAfter == 0, so earmarked debt is repaid from collateral instead of using prefunded cover"
        );
    }
}

contract AlchemistV3Harness is AlchemistV3 {
    /// @dev Manual init for tests only. We bypass Initializable.
    function harnessInit(
        address _admin,
        address _debtToken,
        address _underlyingToken,
        address _transmuter,
        address _protocolFeeReceiver,
        address _myt,
        uint256 _minimumCollateralization,
        uint256 _globalMinimumCollateralization,
        uint256 _collateralizationLowerBound,
        uint256 _protocolFee,
        uint256 _liquidatorFee,
        uint256 _repaymentFee,
        uint256 _depositCap
    ) external {
        admin = _admin;
        debtToken = _debtToken;
        underlyingToken = _underlyingToken;
        transmuter = _transmuter;
        protocolFeeReceiver = _protocolFeeReceiver;
        myt = _myt;

        minimumCollateralization = _minimumCollateralization;
        globalMinimumCollateralization = _globalMinimumCollateralization;
        collateralizationLowerBound = _collateralizationLowerBound;

        protocolFee = _protocolFee;
        liquidatorFee = _liquidatorFee;
        repaymentFee = _repaymentFee;

        depositCap = _depositCap;
        underlyingConversionFactor = 10 ** 12;
        // seed earmark-related trackers so _earmark() won't underflow / short-circuit
        lastEarmarkBlock = block.number;
        lastTransmuterTokenBalance = 0;
    }

    function exposed_lastTransmuterTokenBalance() external view returns (uint256) {
        return lastTransmuterTokenBalance;
    }

    function exposed_earmarkExternal() external {
        _earmark();
    }

    // replicate the accounting prefix of redeem() and return internals
    function exposed_redeem_like_real(
        uint256 /* amount */
    )
        external
        onlyTransmuter
        returns (
            uint256 deltaYield,
            uint256 coverDebt,
            uint256 transmuterBalBefore,
            uint256 lastBalBefore,
            uint256 lastBalAfter
        )
    {
        // Snapshot BEFORE _earmark()
        transmuterBalBefore = TokenUtils.safeBalanceOf(myt, address(transmuter));
        lastBalBefore = lastTransmuterTokenBalance;

        // redeem() calls _earmark() first
        _earmark();

        // Then redeem() recomputes the delta using updated lastTransmuterTokenBalance
        uint256 transmuterBalAfter = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 _deltaYield = (
            transmuterBalAfter > lastTransmuterTokenBalance ? (transmuterBalAfter - lastTransmuterTokenBalance) : 0
        );

        uint256 _coverDebt = convertYieldTokensToDebt(_deltaYield);

        lastBalAfter = lastTransmuterTokenBalance;
        deltaYield = _deltaYield;
        coverDebt = _coverDebt;
    }
}
```
