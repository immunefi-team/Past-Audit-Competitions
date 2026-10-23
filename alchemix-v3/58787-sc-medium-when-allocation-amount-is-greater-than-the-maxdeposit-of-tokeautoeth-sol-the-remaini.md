# 58787 sc medium when allocation amount is greater than the maxdeposit of tokeautoeth sol the remaining is stuck in tokeautoeth sol

**Submitted on Nov 4th 2025 at 13:40:07 UTC by @kaysoft for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58787
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The autoEth contract of Tokemac has a maxDeposit cap amount for each addrress when unlocking profit.

When allocation amount is greater than the maxDeposit of TokeAutoETh.sol, the remaining is stuck in TokeAutoEth.sol

This will allow depositors in VaultV2 after this lock, to mint more shares than expected when these remaining WETH is left in the TokeAutoEth.sol because the totalAssets() function of VaultV2.sol does not account for locked WETH on strategy making the denomenator for shares minting smaller.

This allows them to withdraw more leading to loss for earlier depositors.

## Vulnerability Details

The \_allocate(...) function in the TokeAutoETh.sol function, calls `router.depositMax(autoEth, address(this), 0)` to deposit the maximum amount allowed to the particular `autoEth` vault.

The issue is that if the amount sent to the TokeAutoEth.sol from VaultV2.allocate(...) function is more than the `autoEth.maxDeposit`, only the `autoEth.maxDeposit`for the strategy will be pulled and deposited and the remaining will be left in the TokeAutoEth.sol contract.

```solidity
File: Router 
function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);//@audit check maxDeposit
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;//@audit if maxDeposit is smaller, take maxDeposit
        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```

As can be seen above the `router.depositMax(autoEth, address(this), 0)` call in the \_allocate(...) functon of the TokeAutoEth.sol will not deposit all the `amount` if maxDeposit is smaller. This leaves the remaining WETH stuck in the TokeAutoEth.sol.

```solidity
File: TokeAutoEth.sol
function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(router), amount);
        uint256 shares = router.depositMax(autoEth, address(this), 0);//@audit when maxDeposit for strat is less than amount, only maxDeposit is pulled.
        TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

## Impact Details

* Remaining Asset from maxDeposit is stuck in the TokeAutoEth.sol when allocation amount is greater.
* Subsequent depositors to VaultV2 mint more shares because stuck asset in Strategy is not accounted for when VaultV2.totalAsset() is calcaulted for shares minting. This means these depositors will with withdraw more assets with this inflated shares.

## Recommendation

Consider checking the autoEth.maxDeposit(strat) amount for the strategy first before calling router.depositMax(...) function and handle the remaining if there is any

## Proof of Concept

## Proof of Concept

1. Create a file named POC.t.sol in the `src/test/strategies/POC.t.sol` directory
2. Run `forge test --match-test test_allocate_depositCap`

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

    function test_allocate_depositCap() public {
        uint256 ethAmt = 5000 ether;
        deal(WETH, address(strat), ethAmt);

        uint256 mockedMaxDeposit = 30 ether;

        vm.startPrank(address(0xbeef));

        uint256 stratbalanceBefore = IERC20(WETH).balanceOf(address(strat));

        console.log("stratbalanceBefore", stratbalanceBefore);
        bytes memory prevAllocationAmount = abi.encode(0);

        vm.mockCall(AUTOETH, abi.encodeWithSelector(IAutoPool.maxDeposit.selector, address(0xbeef)), abi.encode(mockedMaxDeposit));

        (bytes32[] memory strategyIds, int256 change) = strat.allocate(prevAllocationAmount, ethAmt, "", address(MYT));
        vm.stopPrank();

        uint256 stratbalanceAfter = IERC20(WETH).balanceOf(address(strat));

        console.log("stratbalanceAfter", stratbalanceAfter);
    }
}
```
