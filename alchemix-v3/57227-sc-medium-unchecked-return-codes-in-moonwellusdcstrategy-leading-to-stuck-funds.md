# 57227 sc medium unchecked return codes in moonwellusdcstrategy leading to stuck funds&#x20;

**Submitted on Oct 24th 2025 at 14:49:36 UTC by @Pig46940 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57227
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `MoonwellUSDCStrategy` contains a vulnerability due to **unchecked return values** from external protocol calls. Specifically, the strategy calls `mint()` on the underlying mToken contract **without verifying success**, assuming these operations always succeed. However, the Moonwell contract’s `mint()` function **return an error code instead of reverting** when they fail. This can cause **funds to become locked** in the `MoonwellUSDCStrategy` contract.

## Vulnerability Details

The Moonwell contracts explicitly describe the return values of `mint()` when an operation fails:

* **MErc20.sol:** `@return uint 0 = success, otherwise a failure (see ErrorReporter.sol for details)`

```solidity
/**
 * @notice Sender supplies assets into the market and receives mTokens in exchange
 * @dev Accrues interest whether or not the operation succeeds, unless reverted
 * @param mintAmount The amount of the underlying asset to supply
 * @return uint 0 = success, otherwise a failure (see ErrorReporter.sol for details)
 */
function mint(uint mintAmount) external returns (uint) {
    (uint err,) = mintInternal(mintAmount);
    return err;
}
```

The `MoonwellUSDCStrategy` **fails to check the return values** from the underlying mToken contract, which can lead to **funds being locked**. The `mint()` returns **an error code instead of reverting**, but the strategy assumes that the operations always succeed, as shown below:

* MoonwellUSDCStrategy.sol

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    // Mint mUSDC with underlying USDC
    mUSDC.mint(amount); // return value is unchecked
    return amount;
}
```

* Fix suggestion:

```solidity
        // Check the returned error code from mUSDC.mint()
        uint256 mintResult = mUSDC.mint(amount);
        if (redeemResult != 0) {
            revert MoonwellCallFailed(mintResult, "mint()");
        }
```

## Impact Details

The strategy assumes that all mToken operations always succeed. However, Moonwell mTokens **return error codes instead of reverting**, and because these return values are not checked, failures can pass silently. It will cause following impacts.

* If `mint()` fails, the USDC remains in the strategy contract instead of being deposited, resulting in **stuck user's funds**.

## References

* <https://github.com/moonwell-fi/contracts-open-source/blob/e23657c5fbeb12c7393fa49da6f350dc0bd5114e/contracts/core/MErc20.sol#L39>
* <https://github.com/moonwell-fi/contracts-open-source/blob/c7da88a3fe3f0062d8a83ba808b648f1da369fec/contracts/core/ErrorReporter.sol#L4>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>

## Proof of Concept

## Proof of Concept

The PoC shows that the strategy ignores Moonwell’s error-code return pattern, treating failed `mint()` calls as successes. This causes deposits to silently fail, leaving **USDC stuck**. As a result, normal users can face stuck funds.

* Create to `./src/test/strategies/MoonwellUSDCStrategyVol.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MoonwellUSDCStrategy} from "../../strategies/optimism/MoonwellUSDCStrategy.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract MockFailingMToken {
    IERC20 public underlying;
    bool public shouldFailMint;
    bool public shouldFailRedeem;
    uint256 public mintErrorCode;
    uint256 public redeemErrorCode;
    uint256 public exchangeRate = 1e18;

    mapping(address => uint256) public balances;

    constructor(address _underlying) {
        underlying = IERC20(_underlying);
    }

    function mint(uint256 mintAmount) external returns (uint256) {
        if (shouldFailMint) return mintErrorCode;
        underlying.transferFrom(msg.sender, address(this), mintAmount);
        balances[msg.sender] += (mintAmount * 1e18) / exchangeRate;
        return 0;
    }

    function balanceOf(address owner) external view returns (uint256) {
        return balances[owner];
    }

    function setMintFailure(bool _fail, uint256 _code) external {
        shouldFailMint = _fail;
        mintErrorCode = _code;
    }

    function setExchangeRate(uint256 _rate) external {
        exchangeRate = _rate;
    }
}

contract MockMoonwellUSDCStrategy is MoonwellUSDCStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _mUSDC,
        address _usdc,
        address _permit2Address
    ) MoonwellUSDCStrategy(_myt, _params, _mUSDC, _usdc, _permit2Address) {}
}

contract MoonwellUSDCStrategyTest is BaseStrategyTest {
    address public constant MOONWELL_USDC_MTOKEN = 0x8E08617b0d66359D73Aa11E11017834C29155525;
    address public constant USDC = 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    MockFailingMToken public mockMToken;
    address public strategyWithMock;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return
            IMYTStrategy.StrategyParams({
                owner: address(1),
                name: "MoonwellUSDC",
                protocol: "MoonwellUSDC",
                riskClass: IMYTStrategy.RiskClass.LOW,
                cap: 10_000e6,
                globalCap: 1e18,
                estimatedYield: 100e6,
                additionalIncentives: false,
                slippageBPS: 1
            });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return
            TestConfig({
                vaultAsset: USDC,
                vaultInitialDeposit: 1000e6,
                absoluteCap: 10_000e6,
                relativeCap: 1e18,
                decimals: 6
            });
    }

    function createStrategy(
        address vault,
        IMYTStrategy.StrategyParams memory params
    ) internal override returns (address) {
        return address(new MockMoonwellUSDCStrategy(vault, params, MOONWELL_USDC_MTOKEN, USDC, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function setUp() public override {
        super.setUp();
        mockMToken = new MockFailingMToken(USDC);

        IMYTStrategy.StrategyParams memory params = getStrategyConfig();
        strategyWithMock = address(
            new MockMoonwellUSDCStrategy(vault, params, address(mockMToken), USDC, OPTIMISM_PERMIT2)
        );
    }

    function test_PROOF_mintFailure_fundsStuck() public {
        uint256 allocAmount = 1000e6;

        deal(USDC, strategyWithMock, allocAmount);
        mockMToken.setMintFailure(true, 3);

        vm.prank(strategyWithMock);
        IERC20(USDC).approve(address(mockMToken), type(uint256).max);

        vm.startPrank(vault);
        bytes memory prevAlloc = abi.encode(0);
        IMYTStrategy(strategyWithMock).allocate(prevAlloc, allocAmount, "", vault);
        vm.stopPrank();

        assertEq(IERC20(USDC).balanceOf(strategyWithMock), allocAmount);
        assertEq(mockMToken.balanceOf(strategyWithMock), 0);
    }
}

```

* Run the test code

```bash
$ export OPTIMISM_RPC_URL=https://mainnet.optimism.io
$ forge test --match-contract MoonwellUSDCStrategyTest --match-test PROOF -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/strategies/MoonwellUSDCStrategyVol.t.sol:MoonwellUSDCStrategyTest
[PASS] test_PROOF_mintFailure_fundsStuck() (gas: 325536)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 343.75ms (1.40ms CPU time)

Ran 1 test suite in 345.36ms (343.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
