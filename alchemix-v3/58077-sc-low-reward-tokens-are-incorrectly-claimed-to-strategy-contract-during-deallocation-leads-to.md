# 58077 sc low reward tokens are incorrectly claimed to strategy contract during deallocation leads to permanent token loss

**Submitted on Oct 30th 2025 at 13:09:18 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58077
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

The `TokeAutoEthStrategy::_deallocate()` function incorrectly claims reward tokens to the strategy contract instead of the intended MYT vault during withdrawal operations, causing permanent loss of TOKE rewards and any additional reward tokens distributed by the rewarder.

## Vulnerability Details

In the `TokeAutoEthStrategy::_deallocate()` function, when withdrawing staked shares from the Tokemak rewarder, the contract calls `rewarder.withdraw(address(this), sharesNeeded, true)` on line 85:

```solidity
// Withdraws auto eth shares from the rewarder with any claims
// redeems same amount of shares from auto eth vault to weth  
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        sharesNeeded = actualSharesHeld;
    }
    // withdraw shares, claim any rewards
@>  rewarder.withdraw(address(this), sharesNeeded, true);
    // ... rest of function
}
```

The third parameter `true` in the `withdraw` call instructs the rewarder to claim accrued rewards. According to the `IMainRewarder` interface, this function signature is:

```solidity
function withdraw(address account, uint256 amount, bool claim) external;
```

When `claim` is set to `true`, the rewarder sends all accrued TOKE rewards and any extra reward tokens to the `account` parameter, which in this case is `address(this)` (the strategy contract).

However, the protocol's intended behavior is demonstrated in the `_claimRewards()` function:

```solidity
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
    rewardsClaimed = rewarder.earned(address(this));
    // This line here actually claims the rewards and sends them to the MorphoV2.
    rewarder.getReward(address(this), address(MYT), false);
}
```

This function correctly uses `getReward(address(this), address(MYT), false)` to claim rewards directly to the MYT vault (`address(MYT)`), not to the strategy contract.

The root cause is that `rewarder.withdraw(address(this), sharesNeeded, true)` claims rewards to `address(this)` (the strategy), while the intended recipient should be the MYT vault. The strategy contract has no mechanism to recover these trapped reward tokens.

## Impact Details

All accrued TOKE rewards are sent to the strategy contract during every `_deallocate()` call. The strategy has no recovery mechanism for these trapped tokens

The Tokemak rewarder system supports additional reward tokens beyond TOKE through "extra rewarders". When `claim=true`, ALL reward types are claimed to the strategy contract. The `_claimRewards()` function only handles TOKE tokens via `getReward(..., false)`, so extra rewards are never recovered. Extra reward tokens become permanently trapped in the strategy contract

## References

The Tokemak rewarder implementation can be found at: <https://github.com/Tokemak/v2-core-pub/blob/main/src/rewarders/MainRewarder.sol>

This is the function that is called when withdrawing: <https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/MainRewarder.sol#L84-L102>

This is the function that processes the rewards: <https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/MainRewarder.sol#L141-L151>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "../strategies/mainnet/TokeAutoEth.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IMainRewarder {
    function balanceOf(address account) external view returns (uint256);
    function earned(address account) external view returns (uint256);
    function rewardToken() external view returns (address);
}

interface IERC4626 {
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}

interface IAutopilotRouter {
    function depositMax(IERC4626 vault, address to, uint256 minSharesOut) external payable returns (uint256 sharesOut);
}

contract PoC_RewardsLostInStrategy is Test {
    // Mainnet addresses
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address public constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94; // TOKE reward token

    TokeAutoEthStrategy public strategy;
    address public mockVault = address(0xBEEF);
    uint256 public mainnetFork;

    function setUp() public {
        // Fork mainnet at a block where rewards have accrued
        // Try to get RPC from env, fallback to public RPC
        string memory rpc;
        try vm.envString("MAINNET_RPC_URL") returns (string memory envRpc) {
            rpc = envRpc;
        } catch {
            rpc = "https://eth.llamarpc.com"; // Public RPC fallback
        }
        mainnetFork = vm.createFork(rpc, 23_677_700);
        vm.selectFork(mainnetFork);

        // Deploy strategy
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 100_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = new TokeAutoEthStrategy(mockVault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2);

        // Setup vault mock to allow allocations/deallocations
        vm.etch(mockVault, bytes("mock"));
    }

    function test_PoC_RewardsLostDuringDeallocation() public {
        uint256 allocationAmount = 1000 ether;

        // Step 1: Allocate WETH to strategy
        deal(WETH, address(strategy), allocationAmount);
        bytes memory prevAllocation = abi.encode(0);
        vm.prank(mockVault);
        strategy.allocate(prevAllocation, allocationAmount, "", mockVault);

        uint256 sharesStaked = IMainRewarder(REWARDER).balanceOf(address(strategy));
        assertTrue(sharesStaked > 0, "Allocation failed - no shares staked");

        console.log("\n=== INITIAL STATE ===");
        console.log("WETH allocated:", allocationAmount / 1e18, "ETH");
        console.log("Shares staked:", sharesStaked / 1e18, "shares");

        // Step 2: Check rewards before time advancement
        uint256 rewardsEarnedBefore = IMainRewarder(REWARDER).earned(address(strategy));
        console.log("Rewards earned initially:", rewardsEarnedBefore / 1e18, "TOKE");

        // Step 3: Advance time by ~40 days to accrue rewards
        uint256 blocksToAdvance = (40 days) / 12; // ~40 days worth of blocks (12 sec per block)
        vm.roll(block.number + blocksToAdvance);
        console.log("Advanced", blocksToAdvance, "blocks (~40 days)");

        // Step 4: Check accrued rewards
        uint256 rewardsEarnedAfter = IMainRewarder(REWARDER).earned(address(strategy));
        assertTrue(rewardsEarnedAfter > 0, "Rewards should have accrued after time advancement");
        console.log("Rewards earned after 40 days:", rewardsEarnedAfter / 1e18, "TOKE");

        // Step 5: Record balances before deallocation
        uint256 strategyTOKEBefore = IERC20(TOKE_TOKEN).balanceOf(address(strategy));
        uint256 vaultTOKEBefore = IERC20(TOKE_TOKEN).balanceOf(mockVault);

        // Step 6: Deallocate - this will claim rewards via withdraw(..., true)
        console.log("\n=== DEALLOCATION ===");
        bytes memory prevAllocation2 = abi.encode(allocationAmount);

        // This is a cheat so the deallocation goes through
        uint256 sharesRequired = IMainRewarder(REWARDER).balanceOf(address(strategy));
        uint256 previewAmount = IERC4626(TOKE_AUTO_ETH_VAULT).convertToAssets(sharesRequired - 0.5e18);

        // uint256 previewAmount = strategy.previewAdjustedWithdraw(allocationAmount);
        vm.prank(mockVault);
        strategy.deallocate(prevAllocation2, previewAmount, "", mockVault);

        // Step 7: Verify where rewards went
        uint256 strategyTOKEAfter = IERC20(TOKE_TOKEN).balanceOf(address(strategy));
        uint256 vaultTOKEAfter = IERC20(TOKE_TOKEN).balanceOf(mockVault);

        uint256 rewardsClaimedToStrategy = strategyTOKEAfter - strategyTOKEBefore;
        uint256 rewardsClaimedToVault = vaultTOKEAfter - vaultTOKEBefore;

        assertEq(rewardsClaimedToVault, 0, "Vault should NOT receive TOKE");
        assertGt(rewardsClaimedToStrategy, 0, "Strategy should receive TOKE");
    }
}
```
