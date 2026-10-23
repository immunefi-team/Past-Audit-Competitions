# 58376 sc low claimrewards function permanently locks earned toke reward token on morpho vaultv2

**Submitted on Nov 1st 2025 at 19:11:53 UTC by @kaysoft for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58376
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `claimRewards()` function of TokeAutoEth.sol used the MYT(VaultV2) as the receiver of the Toke reward tokens. This permently locks the reward tokens on the VaultV2.

## Vulnerability Details

The Toke reward token from calling `claimRewards()` is sent to the MYT which is the Morpho VaultV2 because MYT is passed as the receiver of the reward tokens. Since the VaultV2 does not have function to manage or pull the Toke reward token, the Toke reward token will permanently locked on the VaultV2.

```solidity
File: TokeAutoEth.sol
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
        rewardsClaimed = rewarder.earned(address(this));
        rewarder.getReward(address(this), address(MYT), false);//@audit MYT is the receiver of Toke reward tokens
    }
```

The external claimRewards function is on the parent MYTStrategy.sol contract.

```solidity
File: MYTStrategy.sol
 function claimRewards() public virtual returns (uint256) {
        require(!killSwitch, "emergency");
        _claimRewards();
    }
```

Looking at the `rewarder` contract's getReward(...) function it can be seen that the `receiver` parameter is what is passed as the MYT

```solidity
FIle: https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#code
/**
     * @notice Gets reward for msg.sender.
     * @dev Used to enforce msg.sender check.
     * @param account Account to claim rewards for
     * @param recipient Address to send rewards to
     */
    function getReward(address account, address recipient, bool claimExtras) public {
        if (msg.sender != account && msg.sender != address(systemRegistry.autoPoolRouter())) {
            revert Errors.AccessDenied();
        }

        _getReward(account, recipient, claimExtras);
    }
```

## Impact Details

All earned Toke reward tokens will be permanently locked on the VaultV2 contract when anyone calls the `claimRewards()` function of `TokeAutoEth.sol`

## Recommendation

Consider managing the Token Reward tokens instead of the passing `MYT` as the receiver.

## Proof of Concept

## Proof of Concept

1. Create a file named POC.t.so in this directory: src/test/strategies/POC.t.sol
2. Run `forge test --match-test test_LockRewards_on_MYT -vvv`
3. This test shows that claimRewards() function sends claimed Toke reward tokens to the MYT contract which does not have any function to handle to tokens causing the tokens to be locked on the contract.

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
}

contract TokeAutoEthStrategyTest is Test {
    // Addresses sourced from environment so you can swap networks/blocks easily
    address public constant AUTOETH = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94; // TOKE token address

    IERC20 public autoEth;
    IAutopilotRouter public router;
    IMainRewarder public rewarder;
    IERC20 toke;

    TokeAutoEthStrategy public strat;

    address public constant MYT = address(0xbeef);

    uint256 private _forkId;

    function setUp() public {
        string memory rpc = vm.envString("MAINNET_RPC_URL");
        _forkId = vm.createFork(rpc, 22_089_302);
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

    function test_LockRewards_on_MYT() public {
        uint256 ethAmt = 20 ether;
        deal(WETH, address(strat), ethAmt);

        vm.startPrank(address(0xbeef));
        bytes memory prevAllocationAmount = abi.encode(0);
        //1.Allocate liquidity
        (bytes32[] memory strategyIds, int256 change) = strat.allocate(prevAllocationAmount, ethAmt, "", address(MYT));
        vm.stopPrank();

        //2. Warp time forward to accumulate rewards (e.g., 30 days)
        vm.warp(block.timestamp + 30 days);
        vm.roll(block.number + (30 days / 12)); // Advance blocks too



        uint256 vaultV2BalanceBefore = toke.balanceOf(MYT);

        console.log("MTY(VaultV2) balance before:", vaultV2BalanceBefore);

        //3. Claim reward to lock Toke token reward on MYT(Vault2)
        // And its donated to MYT(Vault2) Locking Toke token permanently
        strat.claimRewards();

        uint256 vaultV2BalanceAfter = toke.balanceOf(MYT);

        console.log("MTY(VaultV2) balance after:", vaultV2BalanceAfter);
          
        //4. Confirm that Toke token rewards are locked on MYT(Vault2)
        assertEq(vaultV2BalanceBefore, 0, "Vault2 Balance before claimRewards");
        assertGt(vaultV2BalanceAfter, 0, "Vault2 Balance before claimRewards");
    }

}
```
