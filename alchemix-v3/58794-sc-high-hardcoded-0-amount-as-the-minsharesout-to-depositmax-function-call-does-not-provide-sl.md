# 58794 sc high hardcoded 0 amount as the minsharesout to depositmax function call does not provide slippage protection

**Submitted on Nov 4th 2025 at 13:54:29 UTC by @kaysoft for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58794
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_allocate(...) function of TokeAutoETh.sol hardcodes zero as the `minSharesOut` leaves the deposit without slippage protection and ready to take 0 as the output shares for deposit.

## Vulnerability Details

The \_allocate functon calls router.depositMax but hardcodes zero as the minSharesOut.

```solidity
File: TokeAutoEth.sol
// @dev Implementation can alternatively make use of a multicall
    // Deposit weth into the autoEth vault, stake the shares in the rewarder
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(router), amount);
        uint256 shares = router.depositMax(autoEth, address(this), 0);//@audit zero slippage.minShares.
        TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

```solidity
File: Router
function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut//<== this is hardcoded as zero
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

No slippage protection as minSharesOut is hardcoded to zero allowing deposit take any exchange rate.

## Recommendation

Consider passsing a \`minSharesOUt instead of hardcoding zero.

## Proof of Concept

## Proof of Concept

1. Create a file named POC.t.sol in the directory `src/test/strategies/POC.t.sol`
2. Run `forge test --match-test test_Zeroslippage -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import "src/strategies/interfaces/ITokemac.sol";
import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/Console.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
    function pause() external;
    function paused() external view returns (bool);
}

interface IAutoPool {
    function asset() external view returns (address);
    function maxDeposit(address receiver) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}

contract TokeAutoEthStrategyTest is Test {
    // Addresses sourced from environment so you can swap networks/blocks easily
    address public constant AUTOETH = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94; // TOKE token address

    address constant AUTOPOOL_ADMIN = 0x127563761083d2Ac7794c17d04E17393D8Ad9013;

    IERC20 public autoEth;
    IAutopilotRouter public router;
    IMainRewarder public rewarder;
    IERC20 toke;

    TokeAutoEthStrategy public strat;

    address public constant MYT = address(0xbeef);

    uint256 private _forkId;

    function setUp() public {
        string memory rpc = vm.envString("MAINNET_RPC_URL");
        _forkId = vm.createFork(rpc, 22_589_302);
        vm.selectFork(_forkId);

        autoEth = IERC20(AUTOETH);
        router = IAutopilotRouter(ROUTER);
        rewarder = IMainRewarder(REWARDER);
        toke = IERC20(TOKE_TOKEN);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "autoETH",
            protocol: "tokemak",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 1
        });

        address permit2Address = 0x000000000022d473030f1dF7Fa9381e04776c7c5; // Mainnet Permit2
        strat = new TokeAutoEthStrategy(MYT, params, AUTOETH, ROUTER, REWARDER, WETH, ORACLE, permit2Address);

        strat.setWhitelistedAllocator(address(0xbeef), true);

        vm.prank(address(strat));
        IERC20(WETH).approve(ROUTER, type(uint256).max);

        vm.makePersistent(address(strat));
    }

    function test_Zeroslippage() public {
        uint256 ethAmt = 50 ether;
        deal(WETH, address(strat), ethAmt);

        vm.startPrank(address(0xbeef));

        bytes memory prevAllocationAmount = abi.encode(0);

        //Zero minSharesOut is hardcoded making the deposit accept even zero shares amount
        (bytes32[] memory strategyIds, int256 change) = strat.allocate(prevAllocationAmount, ethAmt, "", address(MYT));
        vm.stopPrank();
    }
}
```
